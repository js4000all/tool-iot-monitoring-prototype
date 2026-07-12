# Architecture

このドキュメントは、IoT Monitoring Prototype のシステム構成、データフロー、責務境界、主要な設計判断をまとめます。

---

## 1. システム全体像

```text
┌──────────────────────── Raspberry Pi ────────────────────────┐
│                                                              │
│  TimerScheduler                                              │
│        ↓                                                     │
│  TCPRequest / Command                                        │
│        ↓                                                     │
│  Per-IP execution queues                                     │
│        ↓                                                     │
│  TCPWorker                                                   │
│        ↓                                                     │
│  TCPResult                                                   │
│        ↓                                                     │
│  ResultProcessor                                             │
│        ↓                                                     │
│  TelemetryResultHandler                                      │
│        ↓                                                     │
│  PersistentTelemetryQueue                                    │
│        ↓                                                     │
│  TelemetryPublishWorker                                      │
│        ↓                                                     │
│  Protobuf encode → Envelope → MQTT publish                   │
│                                                              │
└──────────────────────────────┬───────────────────────────────┘
                               │ MQTT
                               ▼
┌────────────────────────── Cloud ──────────────────────────────┐
│                                                              │
│  Cloud Ingestor                                              │
│        ↓                                                     │
│  Envelope decode                                             │
│        ↓                                                     │
│  optional decompress                                         │
│        ↓                                                     │
│  TelemetryBatch decode                                       │
│        ↓                                                     │
│  MetricSink                                                  │
│        ↓                                                     │
│  StdoutMetricSink                                            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

---

## 2. 主要な設計判断

### 2.1 Edge Agent と Cloud Ingestor はシングルバイナリ

Raspberry Pi への配布、更新、ロールバック、systemd 管理を簡単にするため、Edge Agent はシングルバイナリとします。

Cloud Ingestor も同様にシングルバイナリとし、コンテナまたは systemd 等から常駐プロセスとして起動できる形にします。

### 2.2 モノレポ

Edge と Cloud で以下を共有するため、モノレポとします。

- `.proto` 定義
- 生成コード
- メッセージ互換性テスト
- 共通の設定・ログ方針
- CI 設定
- ローカル開発環境

### 2.3 同一 IP への TCP リクエストは直列化

対象機器が同時リクエストに対応しない可能性を考慮し、同一 IP アドレスへの TCP リクエストは必ず直列実行します。

異なる IP アドレスについては、独立したワーカーまたはキューにより並列実行します。

```text
IP-A: request 1 → request 2 → request 3
IP-B: request 1 → request 2
IP-C: request 1
```

MVP では優先順位を持ちません。同一 IP 内では FIFO を基本とします。

### 2.4 TCP 実行キューは揮発してよい

MVP では、Edge Agent 再起動時に未実行の TCP リクエストが消えても許容します。

定期ポーリングは次回周期で再生成されるため、実行キューを永続化する価値は低いと判断します。一方、取得済みテレメトリは失わないように永続化します。

---

## 3. Edge Agent の責務境界

### 3.1 基本フロー

```text
TimerScheduler
  ↓
Per-IP execution queue
  ↓
TCPWorker
  ↓
TCPResult
  ↓
ResultProcessor
  ↓
TelemetryResultHandler
  ↓
PersistentTelemetryQueue
  ↓
TelemetryPublishWorker
  ↓
MQTT
```

### 3.2 TCPWorker

TCPWorker は TCP 実行と TCPResult 生成に集中します。

TCPWorker へ以下を持ち込まないでください。

- MQTT トピック
- Protobuf の具体型
- SQLite の具体実装
- テレメトリ固有の保存処理
- クラウド側の事情
- 将来 command 用の API セッション処理

### 3.3 ResultProcessor

TCPWorker は TCP 結果の用途を知りません。結果処理の共通入口として `ResultProcessor` を使います。

```go
type ResultProcessor interface {
    Process(ctx context.Context, command Command, result TCPResult) error
}
```

`ResultProcessor` は用途別処理を自身に抱え込まず、登録済みの `ResultHandler` への到達方法と共通の呼び出し規約を保持します。

MVP ではテレメトリ用 Handler のみを登録します。

```text
ResultProcessor
  └─ RouteTelemetry → TelemetryResultHandler
```

将来 command 結果を追加する場合は、新しい Handler を組み立ててコンストラクタへ追加します。

```text
ResultProcessor
  ├─ RouteTelemetry     → TelemetryResultHandler
  └─ RouteCommandResult → CommandResultHandler
```

Handler の登録はコンストラクタで完了させ、起動後に追加・変更しません。

```go
processor, err := NewResultProcessor(
    map[ResultRoute]ResultHandler{
        RouteTelemetry: telemetryHandler,
    },
)
```

### 3.4 ResultHandler

`TelemetryResultHandler` の責務は以下です。

1. `Command` と `TCPResult` を解釈する
2. `TelemetryItem` を生成する
3. 永続キューへ投入する

MQTT 接続、バッチ化、圧縮、再送は扱いません。

```text
TCPResult
  ↓
TelemetryResultHandler
  ↓
TelemetryItem
  ↓
PersistentTelemetryQueue
```

`Handle` 成功は、テレメトリが永続キューへ保存されたことを意味します。MQTT publish 完了までは意味しません。

---

## 4. テレメトリ永続化と Publish

### 4.1 永続化してからバッチ化する

テレメトリは 1 件単位で永続化します。

```text
TelemetryItem
  ↓
SQLite
  ↓
複数件取得
  ↓
TelemetryBatch
  ↓
serialize / optional compress
  ↓
publish
```

メモリ上でバッチを作ってから永続化すると、未確定バッチがプロセス終了時に失われるため採用しません。

### 4.2 テレメトリ永続キュー

SQLite を想定します。

概念的なスキーマ例:

```sql
CREATE TABLE telemetry_queue (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sensor_id TEXT NOT NULL,
    value REAL NOT NULL,
    measured_at_unixtime_s INTEGER NOT NULL,
    created_at_unixtime_s INTEGER NOT NULL
);
```

MVP では複雑な汎用キュー抽象を作らず、テレメトリ用途に必要な操作へ限定します。

```go
type TelemetryQueue interface {
    Enqueue(ctx context.Context, item TelemetryItem) error
    FetchBatch(ctx context.Context, limit int) ([]QueuedTelemetry, error)
    Delete(ctx context.Context, ids []int64) error
}
```

必要になった時点で、Lease / Ack / Nack モデルへ拡張します。

### 4.3 バッチング

テレメトリは以下のいずれかを満たした時点で publish 対象とします。

- 最大件数に到達
- 最大待機時間に到達

初期値は設定可能にします。

```yaml
telemetry:
  batch:
    max_items: 100
    max_wait: 1s
```

MVP では最大バイト数による制御を実装しません。

---

## 5. MQTT と Protobuf

### 5.1 MQTT メッセージは Protobuf ベース

MVP では以下のメッセージを定義します。

```proto
message TelemetryItem {
  string sensor_id = 1;
  double value = 2;
  int64 measured_at_unixtime_s = 3;
}

message TelemetryBatch {
  repeated TelemetryItem items = 1;
}
```

テレメトリは圧縮有無をメッセージ自身で判別できるよう、Protobuf Envelope で包みます。

```proto
enum Compression {
  COMPRESSION_NONE = 0;
  COMPRESSION_ZSTD = 1;
}

message TelemetryEnvelope {
  uint32 envelope_version = 1;
  Compression compression = 2;
  bytes payload = 3;
}
```

MVP では常に以下を使用します。

```text
compression = COMPRESSION_NONE
```

### 5.2 圧縮方針は設定で固定する

将来圧縮を導入する場合、送信側の設定ファイルまたは環境変数で方式を固定します。

```yaml
telemetry:
  compression: zstd
```

または、

```yaml
telemetry:
  compression: none
```

動的な圧縮率判定は初期対象外です。

受信側は設定を推測せず、Envelope のマークだけを見て解凍します。

### 5.3 MQTT トピック

MVP ではテレメトリ送信のみを使用します。

```text
iot/{site_id}/{gateway_id}/up/telemetry
```

例:

```text
iot/site-a/gateway-01/up/telemetry
```

将来の command 対応では、以下を候補とします。

```text
iot/{site_id}/{gateway_id}/down/command
iot/{site_id}/{gateway_id}/up/command-result
```

MVP では command 用トピックを実装しません。

### 5.4 MQTT 方針

MVP の初期設定は以下です。

- MQTT 認証なし
- TLS なし
- QoS 1
- retain なし
- 単一ブローカー
- 再接続は MQTT クライアントライブラリの機能を利用
- publish 失敗時は永続キューのレコードを残す
- publish 成功確認後に対象レコードを削除する

認証なしは、閉域ネットワークまたは接続元制限がある環境でのプロトタイプ用途に限定します。

---

## 6. Cloud Ingestor

MVP では標準出力に記録します。

```go
type MetricSink interface {
    Write(ctx context.Context, item TelemetryItem) error
}
```

初期実装:

```text
StdoutMetricSink
```

将来候補:

```text
RDBMetricSink
OpenTelemetryMetricSink
CompositeMetricSink
```

MVP 段階で将来の Sink を実装する必要はありません。

---

## 7. 想定コンポーネント

### Edge Agent

```text
TimerScheduler
Command
TCPResult
PerIPCommandQueue
TCPWorker
ResultRoute
ResultProcessor
ResultHandler
TelemetryResultHandler
TelemetryQueue
SQLiteTelemetryQueue
TelemetryPublishWorker
TelemetryEncoder
MQTTPublisher
```

### Cloud Ingestor

```text
MQTTSubscriber
TelemetryEnvelopeDecoder
TelemetryBatchDecoder
MetricSink
StdoutMetricSink
```

### Shared

```text
Protobuf definitions
Generated Protobuf code
Configuration types
Logging conventions
```

すべてを最初からインターフェース化しません。抽象化する主な候補は以下です。

- 外部 I/O 境界
- 複数実装が現実的に存在する箇所
- 障害特性が異なる箇所
- 将来独立して変更される可能性が高い箇所

単純な変換は、関数または具象クラス内部の private 処理で構いません。

---

## 8. Composition Root

依存関係の組み立ては、各アプリの起動処理または専用の bootstrap パッケージへ集約します。

例:

```go
func BuildEdgeApplication(cfg Config) (*Application, error) {
    telemetryQueue := NewSQLiteTelemetryQueue(cfg.DatabasePath)
    publisher := NewMQTTPublisher(cfg.MQTT)

    telemetryHandler := NewTelemetryResultHandler(telemetryQueue)

    resultProcessor, err := NewResultProcessor(
        map[ResultRoute]ResultHandler{
            RouteTelemetry: telemetryHandler,
        },
    )
    if err != nil {
        return nil, err
    }

    tcpWorker := NewTCPWorker(resultProcessor)
    scheduler := NewTimerScheduler(tcpWorker)
    publishWorker := NewTelemetryPublishWorker(
        telemetryQueue,
        publisher,
        cfg.Telemetry.Batch,
    )

    return NewApplication(scheduler, publishWorker), nil
}
```

Composition Root は単なる初期化コードではなく、システム全体の接続関係を読むための索引です。構築処理を複数パッケージへ分散させないでください。

---

## 9. 将来拡張

### Command

将来はクラウド API を起点として、以下の経路を追加する可能性があります。

```text
Cloud API
  ↓ request ID
MQTT command
  ↓
Edge Agent
  ↓
Per-IP execution queue
  ↓
TCP
  ↓
CommandResultHandler
  ↓
MQTT command-result
  ↓
Cloud API session
```

command は 1 命令につき 1 メッセージとし、圧縮しない想定です。command 結果も 1 命令につき 1 メッセージとし、圧縮しない想定です。

同期 API として見せる方式には未解決の論点があります。実装時には、完全同期、完全非同期、短時間同期と非同期フォールバックの比較を行ってください。

### Telemetry compression

将来、設定により zstd 圧縮を有効化できるようにします。受信側は必ず Envelope の `compression` を確認し、送信元設定を推測しません。

### Metric routing

Cloud Ingestor では、`MetricSink` の実装追加により以下へ拡張できます。

- RDB
- OpenTelemetry
- 複数 Sink への同時出力

### MQTT 以外の転送方式サポート

将来要件が明確になった時点で検討します。

---

## 10. 設計原則

- 抽象は曖昧さではなく、必要な契約の明確化である
- 上位と下位は、具体実装ではなく同じ契約へ依存する
- MVP に不要な仕組みを先回りして実装しない
- 永続化、通信、結果処理の責務境界を混在させない
