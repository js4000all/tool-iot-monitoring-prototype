# IoT Monitoring Prototype

Raspberry Pi上でTCP対応機器からメトリクスを収集し、クラウドへ転送するIoT監視システムのプロトタイプです。

本リポジトリはモノレポとして管理し、以下の2つの常駐プロセスをビルドします。

* **Edge Agent**: Raspberry Pi上で動作する収集・送信アプリ
* **Cloud Ingestor**: MQTTメッセージを受信し、メトリクスを処理するクラウド側アプリ

両方とも、依存ランタイムを必要としない**シングルバイナリ**として配布することを前提とします。実装言語はGoを想定します。

---

## 1. 目的

MVPでは、次の経路を最小構成で成立させます。

```text
Timer
  ↓
TCP request
  ↓
TCP response
  ↓
persistent telemetry queue
  ↓
batch
  ↓
Protobuf envelope
  ↓
MQTT publish
  ↓
Cloud Ingestor
  ↓
stdout
```

将来的には、クラウド側で受信したメトリクスを以下へルーティングできる構成を想定します。

* RDB
* OpenTelemetry
* その他のメトリクス基盤

MVPでは、受信・復号したメトリクスを標準出力へ記録できれば十分です。

将来的な構想としては、command機能(クラウド起点でエッジ側TCPリクエストを実行)や、MQTT以外の伝達方式のサポートも候補です。

---

## 2. MVPスコープ

### 実装するもの

#### Edge Agent

* 設定された周期でTCPリクエストを生成する
* TCPリクエストを対象IPアドレス別のキューへ投入する
* 同一IPアドレスへのTCPリクエストを直列実行する
* 異なるIPアドレスへのTCPリクエストを並列実行する
* TCPレスポンスをテレメトリへ変換する
* テレメトリを永続キューへ保存する
* 保存されたテレメトリを件数またはタイムアウトでバッチ化する
* バッチをProtobufでシリアライズする
* Protobuf Envelopeで包み、MQTTへpublishする
* MQTT切断中もテレメトリを保持し、再接続後に再送する
* publish成功後に永続キューから対象レコードを削除する

#### Cloud Ingestor

* MQTTブローカーへ接続する
* テレメトリ用トピックをsubscribeする
* Protobuf Envelopeをdecodeする
* Envelopeの圧縮方式を確認する
* 必要な場合にpayloadを解凍する
* TelemetryBatchをdecodeする
* 各テレメトリを標準出力へ記録する

### MVPでは実装しないもの

* クラウドからEdge Agentへのcommand送信
* command結果の返却
* APIセッション管理
* MQTTによる同期RPC
* TCPリクエストの優先順位
* TCPリクエスト実行キューの永続化
* commandの冪等性管理
* MQTT認証
* MQTT TLS
* 複数MQTTブローカーへのフェイルオーバー
* テレメトリ圧縮
* RDBへの保存
* OpenTelemetryへの送信
* 設定の動的リロード
* Edge Agentの自己更新

将来のcommand対応を妨げない構造は残しますが、MVPの実装対象には含めません。

---

## 3. システム全体像

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

## 4. 主要な設計判断

### 4.1 Edge AgentとCloud Ingestorはシングルバイナリ

Raspberry Piへの配布、更新、ロールバック、systemd管理を簡単にするため、Edge Agentはシングルバイナリとします。

Cloud Ingestorも同様にシングルバイナリとし、コンテナまたはsystemd等から常駐プロセスとして起動できる形にします。

### 4.2 モノレポ

EdgeとCloudで以下を共有するため、モノレポとします。

* `.proto`定義
* 生成コード
* メッセージ互換性テスト
* 共通の設定・ログ方針
* CI設定
* ローカル開発環境

### 4.3 同一IPへのTCPリクエストは直列化

対象機器が同時リクエストに対応しない可能性を考慮し、同一IPアドレスへのTCPリクエストは必ず直列実行します。

異なるIPアドレスについては、独立したワーカーまたはキューにより並列実行します。

```text
IP-A: request 1 → request 2 → request 3
IP-B: request 1 → request 2
IP-C: request 1
```

MVPでは優先順位を持ちません。同一IP内ではFIFOを基本とします。

### 4.4 TCP実行キューは揮発してよい

MVPでは、Edge Agent再起動時に未実行のTCPリクエストが消えても許容します。

定期ポーリングは次回周期で再生成されるため、実行キューを永続化する価値は低いと判断します。

一方、取得済みテレメトリは失わないように永続化します。

### 4.5 ResultProcessorを結果処理の入口とする

TCPWorkerは、TCP結果の用途を知りません。

```go
type ResultProcessor interface {
    Process(ctx context.Context, command Command, result TCPResult) error
}
```

`ResultProcessor`は、結果処理の具体的な実装を自身に抱え込むのではなく、登録済みの`ResultHandler`への到達方法と共通の呼び出し規約を保持します。

MVPではテレメトリ用Handlerのみを登録します。

```text
ResultProcessor
  └─ RouteTelemetry → TelemetryResultHandler
```

将来command結果を追加する場合は、新しいHandlerを組み立ててコンストラクタへ追加します。

```text
ResultProcessor
  ├─ RouteTelemetry     → TelemetryResultHandler
  └─ RouteCommandResult → CommandResultHandler
```

### 4.6 ResultProcessorはimmutable

Handlerの登録はコンストラクタで完了させ、起動後に追加・変更しません。

```go
processor, err := NewResultProcessor(
    map[ResultRoute]ResultHandler{
        RouteTelemetry: telemetryHandler,
    },
)
```

これにより以下を避けます。

* 実行中の登録変更
* map更新に伴う同期
* 登録漏れの遅延発覚
* 実行時構成の不透明化

### 4.7 ResultHandlerはメッセージ生成とキュー投入まで

`TelemetryResultHandler`の責務は以下です。

1. `Command`と`TCPResult`を解釈する
2. `TelemetryItem`を生成する
3. 永続キューへ投入する

MQTT接続、バッチ化、圧縮、再送は扱いません。

```text
TCPResult
  ↓
TelemetryResultHandler
  ↓
TelemetryItem
  ↓
PersistentTelemetryQueue
```

`Handle`成功は、テレメトリが永続キューへ保存されたことを意味します。MQTT publish完了までは意味しません。

### 4.8 永続化してからバッチ化する

テレメトリは1件単位で永続化します。

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

### 4.9 MQTTメッセージはProtobufベース

MVPでは以下のメッセージを定義します。

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

テレメトリは圧縮有無をメッセージ自身で判別できるよう、Protobuf Envelopeで包みます。

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

MVPでは常に以下を使用します。

```text
compression = COMPRESSION_NONE
```

将来圧縮を有効化しても、受信側は各メッセージの`compression`を見て処理できるため、送信側と受信側の設定同期に依存しません。

### 4.10 圧縮方針は設定で固定する

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

処理順序は以下です。

```text
TelemetryBatch
  ↓ Protobuf serialize
payload
  ↓ configured compression
compressed or raw payload
  ↓
TelemetryEnvelope
```

受信側は設定を推測せず、Envelopeのマークだけを見て解凍します。

### 4.11 クラウド側の出力先は抽象化する

MVPでは標準出力に記録します。

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

MVP段階で将来のSinkを実装する必要はありません。

---

## 5. MQTTトピック

MVPではテレメトリ送信のみを使用します。

```text
iot/{site_id}/{gateway_id}/up/telemetry
```

例:

```text
iot/site-a/gateway-01/up/telemetry
```

将来のcommand対応では、以下を候補とします。

```text
iot/{site_id}/{gateway_id}/down/command
iot/{site_id}/{gateway_id}/up/command-result
```

MVPではcommand用トピックを実装しません。

---

## 6. MQTT方針

MVPの初期設定は以下です。

* MQTT認証なし
* TLSなし
* QoS 1
* retainなし
* 単一ブローカー
* 再接続はMQTTクライアントライブラリの機能を利用
* publish失敗時は永続キューのレコードを残す
* publish成功確認後に対象レコードを削除する

認証なしは、閉域ネットワークまたは接続元制限がある環境でのプロトタイプ用途に限定します。

---

## 7. テレメトリ永続キュー

SQLiteを想定します。

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

MVPでは複雑な汎用キュー抽象を作らず、テレメトリ用途に必要な操作へ限定します。

```go
type TelemetryQueue interface {
    Enqueue(ctx context.Context, item TelemetryItem) error
    FetchBatch(ctx context.Context, limit int) ([]QueuedTelemetry, error)
    Delete(ctx context.Context, ids []int64) error
}
```

必要になった時点で、Lease/Ack/Nackモデルへ拡張します。

---

## 8. バッチング

テレメトリは以下のいずれかを満たした時点でpublish対象とします。

* 最大件数に到達
* 最大待機時間に到達

初期値は設定可能にします。

```yaml
telemetry:
  batch:
    max_items: 100
    max_wait: 1s
```

MVPでは最大バイト数による制御を実装しません。

---

## 9. 想定コンポーネント

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

すべてを最初からインターフェース化しません。

抽象化する主な候補は以下です。

* 外部I/O境界
* 複数実装が現実的に存在する箇所
* 障害特性が異なる箇所
* 将来独立して変更される可能性が高い箇所

単純な変換は、関数または具象クラス内部のprivate処理で構いません。

---

## 10. リポジトリ構成案

```text
.
├── README.md
├── go.mod
├── cmd/
│   ├── edge-agent/
│   │   └── main.go
│   └── cloud-ingestor/
│       └── main.go
├── internal/
│   ├── edge/
│   │   ├── app/
│   │   ├── config/
│   │   ├── scheduler/
│   │   ├── tcp/
│   │   ├── result/
│   │   ├── telemetry/
│   │   └── mqtt/
│   ├── cloud/
│   │   ├── app/
│   │   ├── config/
│   │   ├── ingest/
│   │   └── sink/
│   └── shared/
│       ├── logging/
│       └── mqtt/
├── proto/
│   └── telemetry.proto
├── gen/
│   └── proto/
├── migrations/
│   └── edge/
├── configs/
│   ├── edge.example.yaml
│   └── cloud.example.yaml
├── deployments/
│   ├── systemd/
│   └── docker/
├── scripts/
└── test/
```

Goの`internal`境界を利用し、Edge固有実装とCloud固有実装を分離します。

共有パッケージへ業務ロジックを安易に集約しないでください。共有するのは、本当に同じ契約であるものに限定します。

---

## 11. Composition Root

依存関係の組み立ては、各アプリの起動処理または専用のbootstrapパッケージへ集約します。

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

Composition Rootは単なる初期化コードではなく、システム全体の接続関係を読むための索引です。

構築処理を複数パッケージへ分散させないでください。

---

## 12. AIエージェント向け実装ガイド

### 計画時に明示すること

実装前に、最低限以下を整理してください。

1. 変更対象のアプリ
2. 追加・変更するコンポーネント
3. データの流れ
4. 永続化境界
5. エラー時に失われるもの、残るもの
6. Protobuf互換性への影響
7. Composition Rootの変更内容
8. MVPスコープ外へ踏み込んでいないか

### 実装方針

* MVPに不要な汎用化を避ける
* 将来のcommand機能を先回りして実装しない
* ResultProcessorの公開入口を単純に保つ
* ResultProcessorへ用途別の条件分岐を増やさない
* 結果固有処理はResultHandlerへ置く
* ResultHandlerからMQTTを直接操作しない
* TCPWorkerからテレメトリ固有型を参照しない
* 永続化成功とpublish成功を混同しない
* Protobufの既存フィールド番号を再利用しない
* MQTT payload用の独自バイナリヘッダーを作らない
* 動的DIコンテナを導入しない
* 接続関係は静的かつ明示的に構築する

### 避けるべき過剰設計

MVPでは以下を作らないでください。

* 汎用MessageBus
* 汎用Workflow Engine
* 動的Plugin機構
* Handlerの実行時追加
* 独自シリアライズ形式
* 汎用分散キュー
* 複雑なRetry Policy階層
* 複数圧縮方式の自動選択
* commandを想定した未使用コード
* 使われないインターフェース

### テスト方針

最低限、以下を自動テストします。

#### Edge

* 同一IPへのTCPリクエストが直列実行される
* 異なるIPへのTCPリクエストが並列実行できる
* TCP成功結果がTelemetryItemへ変換される
* TCP失敗時の扱いが明確である
* テレメトリがSQLiteへ永続化される
* publish失敗時にレコードが残る
* publish成功時に対象レコードが削除される
* 件数条件でバッチ送信される
* 時間条件でバッチ送信される

#### Shared protocol

* TelemetryBatchのencode/decode
* TelemetryEnvelopeのencode/decode
* `COMPRESSION_NONE`の処理
* 未知の圧縮方式を安全に拒否する

#### Cloud

* MQTT payloadからTelemetryEnvelopeをdecodeできる
* TelemetryBatchをdecodeできる
* 各TelemetryItemをMetricSinkへ渡せる
* StdoutMetricSinkが期待形式で出力する

### 受け入れ条件

MVPは以下を満たした時点で成立とします。

1. Edge AgentをRaspberry Pi向けにビルドできる
2. Cloud Ingestorをビルドできる
3. ローカルMQTTブローカーを使ったE2Eテストが通る
4. Edge AgentがTCPレスポンスをSQLiteへ保存できる
5. MQTT停止中もデータがSQLiteへ残る
6. MQTT復旧後に蓄積データが送信される
7. Cloud Ingestorが受信メトリクスを標準出力へ表示する
8. Edge AgentとCloud Ingestorを単一バイナリで起動できる

---

## 13. 将来拡張

### Command

将来はクラウドAPIを起点として、以下の経路を追加する可能性があります。

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

commandは1命令につき1メッセージとし、圧縮しない想定です。

command結果も1命令につき1メッセージとし、圧縮しない想定です。

同期APIとして見せる方式には未解決の論点があります。実装時には、完全同期、完全非同期、短時間同期と非同期フォールバックの比較を行ってください。

### Telemetry compression

将来、設定によりzstd圧縮を有効化できるようにします。

受信側は必ずEnvelopeの`compression`を確認し、送信元設定を推測しません。

### Metric routing

Cloud Ingestorでは、`MetricSink`の実装追加により以下へ拡張できます。

* RDB
* OpenTelemetry
* 複数Sinkへの同時出力

---

## 14. 設計原則

このプロトタイプでは、以下を重視します。

* 抽象は曖昧さではなく、必要な契約の明確化である
* 上位と下位は、具体実装ではなく同じ契約へ依存する
* 複雑さを消すのではなく、変更しやすい単位へ配置する
* 個々のクラスの小ささより、接続関係の読みやすさを重視する
* Composition Rootをアーキテクチャの索引として扱う
* 将来の可能性ではなく、確認できた変更軸に対して抽象化する
* MVPでは機能を削るが、配送保証の境界は曖昧にしない
  ::: 
