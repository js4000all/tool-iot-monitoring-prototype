# MVP Implementation Plan

このドキュメントは、MVP へ向けた当面の実装順序、PR 分割、進捗管理、設計判断の継承方法を定義します。正とするスコープは [MVP.md](MVP.md)、責務境界は [ARCHITECTURE.md](ARCHITECTURE.md)、作業規約は [CONTRIBUTING.md](CONTRIBUTING.md) です。

---

## 1. 方針

- 1 PR は 1 つの意図に集中させ、レビューと revert を容易にする。
- `proto`、Edge、Cloud、E2E の変更は可能な限り独立したファイル群に分け、同時作業時のコンフリクトを避ける。
- データフローを縦に薄く通す前に、互換性の土台となる Protobuf と設定・Composition Root の骨格を先に固定する。
- 永続化成功と MQTT publish 成功を混同しない。
- MVP 外の command、認証、TLS、RDB 保存、圧縮実装は計画に含めない。

---

## 2. 現在のデータフロー

現時点では、リポジトリは設計ドキュメント中心で、Go モジュール、実装ファイル、`.proto`、設定例は未作成です。MVP で成立させるデータフローは以下です。

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

---

## 3. 変更後の到達状態

MVP 完了時には、以下を満たします。

1. Edge Agent と Cloud Ingestor を単一バイナリとしてビルドできる。
2. Edge Agent が疑似 TCP サーバーからレスポンスを取得し、TelemetryItem へ変換できる。
3. 取得済みテレメトリが SQLite に保存され、プロセス停止や MQTT 停止中も残る。
4. 保存済みテレメトリが件数または待機時間でバッチ化される。
5. TelemetryBatch が Protobuf Envelope に包まれ、MQTT へ QoS 1 で publish される。
6. publish 成功後にのみ SQLite の対象レコードが削除される。
7. Cloud Ingestor が MQTT から受信した Envelope と Batch を decode し、各メトリクスを stdout へ出力する。

---

## 4. PR 分割計画

### Phase 0: 実装基盤

| PR | 目的 | 主なファイル領域 | 依存 | 並行可否 |
| --- | --- | --- | --- | --- |
| 0-1 | Go モジュール、基本ディレクトリ、空の起動点を追加 | `go.mod`, `cmd/`, `internal/` | なし | 起点のため先行 |
| 0-2 | 設定型、設定例、起動時 validation 方針を追加 | `configs/`, `internal/*/config` | 0-1 | 0-3 と一部並行可 |
| 0-3 | ログ初期化とアプリケーション骨格を追加 | `internal/*/bootstrap`, `cmd/` | 0-1 | 0-2 と一部並行可 |

### Phase 1: 共有プロトコル

| PR | 目的 | 主なファイル領域 | 依存 | 並行可否 |
| --- | --- | --- | --- | --- |
| 1-1 | `telemetry.proto` と生成手順を追加 | `proto/`, `gen/`, `scripts/` | 0-1 | Edge/Cloud の具象実装前に先行 |
| 1-2 | Protobuf encode/decode 互換性テストを追加 | `internal/shared/telemetry` または `gen/` 周辺 | 1-1 | Phase 2 の設計作業と並行可 |

### Phase 2: Edge の収集と永続化

| PR | 目的 | 主なファイル領域 | 依存 | 並行可否 |
| --- | --- | --- | --- | --- |
| 2-1 | TCP 実行の最小型、TCPWorker、疑似 TCP テストを追加 | `internal/edge/tcp` | 0-1 | 2-2 と並行可 |
| 2-2 | TimerScheduler と Per-IP queue を追加 | `internal/edge/scheduler`, `internal/edge/queue` | 0-1 | 2-1 と境界合意後に並行可 |
| 2-3 | ResultProcessor と TelemetryResultHandler を追加 | `internal/edge/result`, `internal/edge/telemetry` | 2-1 | 2-4 と一部並行可 |
| 2-4 | SQLiteTelemetryQueue と migration を追加 | `internal/edge/telemetry`, `migrations/edge` | 0-1 | 2-3 と interface 合意後に並行可 |
| 2-5 | Edge Composition Root で収集→永続化を接続 | `internal/edge/bootstrap`, `cmd/edge-agent` | 2-1, 2-2, 2-3, 2-4 | 統合 PR のため単独推奨 |

### Phase 3: Edge の publish

| PR | 目的 | 主なファイル領域 | 依存 | 並行可否 |
| --- | --- | --- | --- | --- |
| 3-1 | TelemetryBatch encoder と Envelope encoder を追加 | `internal/edge/telemetry`, `internal/shared/telemetry` | 1-1, 2-4 | 3-2 と並行可 |
| 3-2 | MQTTPublisher 境界と publish 成功/失敗テストを追加 | `internal/edge/mqtt` | 0-2 | 3-1 と並行可 |
| 3-3 | TelemetryPublishWorker を追加 | `internal/edge/telemetry` | 2-4, 3-1, 3-2 | 単独推奨 |
| 3-4 | Edge Composition Root で永続化→publish を接続 | `internal/edge/bootstrap`, `cmd/edge-agent` | 3-3 | 単独推奨 |

### Phase 4: Cloud Ingestor

| PR | 目的 | 主なファイル領域 | 依存 | 並行可否 |
| --- | --- | --- | --- | --- |
| 4-1 | Envelope/Batch decoder と unsupported compression 拒否を追加 | `internal/cloud/telemetry` | 1-1 | Phase 2/3 と並行可 |
| 4-2 | MetricSink と StdoutMetricSink を追加 | `internal/cloud/sink` | 0-1 | 4-1 と並行可 |
| 4-3 | MQTTSubscriber と Cloud Composition Root を追加 | `internal/cloud/mqtt`, `internal/cloud/bootstrap`, `cmd/cloud-ingestor` | 4-1, 4-2 | 単独推奨 |

### Phase 5: E2E と運用最小化

| PR | 目的 | 主なファイル領域 | 依存 | 並行可否 |
| --- | --- | --- | --- | --- |
| 5-1 | ローカル MQTT を使う E2E テストを追加 | `test/`, `deployments/` | Phase 3, Phase 4 | 単独推奨 |
| 5-2 | Raspberry Pi 向け build 手順と systemd 例を追加 | `docs/`, `deployments/` | 0-1 | 5-1 と並行可 |
| 5-3 | MVP 受け入れ条件チェックリストを更新 | `docs/` | 全 Phase | 最終確認 |

---

## 5. 追加・変更する型やコンポーネント

### Edge Agent

- `TimerScheduler`
- `Command`
- `TCPResult`
- `PerIPCommandQueue`
- `TCPWorker`
- `ResultRoute`
- `ResultProcessor`
- `ResultHandler`
- `TelemetryResultHandler`
- `TelemetryItem`
- `QueuedTelemetry`
- `SQLiteTelemetryQueue`
- `TelemetryPublishWorker`
- `TelemetryEncoder`
- `MQTTPublisher`
- Edge 用 Composition Root

### Cloud Ingestor

- `MQTTSubscriber`
- `TelemetryEnvelopeDecoder`
- `TelemetryBatchDecoder`
- `MetricSink`
- `StdoutMetricSink`
- Cloud 用 Composition Root

### Shared / Protocol

- `TelemetryItem`
- `TelemetryBatch`
- `TelemetryEnvelope`
- `Compression`
- 生成コード
- Protobuf 互換性テスト

---

## 6. 永続化境界

- TCP 実行キューは揮発でよい。
- 取得済みテレメトリは SQLite の `telemetry_queue` に 1 件単位で保存する。
- `TelemetryResultHandler.Handle` の成功は SQLite 保存完了を意味する。
- `TelemetryPublishWorker` は SQLite から batch を取得し、publish 成功後に対象 ID を削除する。
- publish 失敗、MQTT 切断、Cloud 側 decode 失敗は SQLite レコード削除の理由にしない。

---

## 7. エラー時の挙動

| 箇所 | 挙動 |
| --- | --- |
| 設定 validation 失敗 | 起動失敗にする |
| TCP 接続/読み取り失敗 | エラー文脈を付け、該当周期の結果として処理する。取得済みでないテレメトリは保存しない |
| Telemetry 変換失敗 | 永続化せず、対象 IP や command を含めてエラー化する |
| SQLite Enqueue 失敗 | publish 済み扱いにしない。呼び出し元へ返す |
| MQTT publish 失敗 | SQLite レコードを残す |
| Envelope decode 失敗 | Cloud 側で該当メッセージを拒否し、MetricSink へ渡さない |
| unsupported compression | 明示的にエラーにし、正常 decode と扱わない |

---

## 8. テスト方法

### 常に実行する

```bash
go test ./...
go vet ./...
go build ./...
```

### 該当 PR で追加する

- Protobuf encode/decode 互換性テスト
- 疑似 TCP サーバーによる TCPWorker テスト
- 同一 IP 直列・異なる IP 並列の queue テスト
- SQLite 一時 DB による永続化テスト
- publish 成功時 delete / 失敗時 retain のテスト
- Cloud decode と StdoutMetricSink のテスト

### 環境がある場合のみ実行する

- ローカル MQTT ブローカーを使う E2E
- ブローカー停止・復旧試験
- Raspberry Pi または ARM 実行環境での起動確認

---

## 9. MVP スコープへの影響

この計画は [MVP.md](MVP.md) の最小データフローを実装するための分割です。以下は明示的に含めません。

- command 送受信
- MQTT 認証/TLS
- 複数 broker failover
- テレメトリ圧縮の実装
- Cloud 側 RDB 保存
- OpenTelemetry 送信
- 設定の動的リロード

---

## 10. 進捗管理

### 10.1 ステータス値

各タスクは以下のいずれかの状態にします。

| ステータス | 意味 |
| --- | --- |
| `Planned` | 着手前 |
| `In Progress` | 作業中 |
| `Blocked` | 外部条件または先行 PR 待ち |
| `Review` | PR レビュー中 |
| `Done` | main 相当へ反映済み |
| `Deferred` | MVP から外した、または後回しにした |

### 10.2 タスク台帳の置き場所

- 進捗の正本は issue tracker または project board に置く。
- リポジトリ内には大きな台帳を置かず、この計画ドキュメントと PR テンプレートから参照する。
- リポジトリ内で更新が必要な場合は、Phase 単位の完了状況だけをこのドキュメントへ反映する。

### 10.3 PR 間コンフリクトを減らすルール

- 同じ PR で Edge と Cloud の大きな実装を同時に変更しない。
- `.proto` 変更は専用 PR にし、生成コードも同じ PR に含める。
- Composition Root の接続変更は統合 PR に寄せ、部品 PR では constructor とテストを中心にする。
- 設定例のキー追加は専用または関連 PR の末尾で行い、同時に複数 PR で同じ YAML ブロックを編集しない。
- 大きなリネームや整形のみの変更は機能 PR に混ぜない。

---

## 11. コンテキストと設計判断の継承

### 11.1 各 PR に残す情報

各 PR は [templates/pr-description.md](templates/pr-description.md) を使い、少なくとも以下を残します。

- 変更したデータフロー
- 永続化境界
- エラー時の挙動
- Composition Root への影響
- MVP スコープ内である理由
- 後続 PR への引き継ぎ

### 11.2 設計判断ログ

設計判断が発生した場合は、以下のいずれかに記録します。

- 既存設計の補足であれば [ARCHITECTURE.md](ARCHITECTURE.md)
- 作業分割や進め方の判断であればこのドキュメント
- PR 固有の実装判断であれば PR 本文の `Design` と `Handoff` 欄

### 11.3 引き継ぎテンプレート

実装途中でコンテキストを引き継ぐ場合は [templates/handoff.md](templates/handoff.md) を使用します。特に以下を明記します。

- どの Phase / PR の作業か
- 直近で読んだファイル
- 変更済みファイルと未完了ファイル
- 実行済みテストと未実施テスト
- 迷った設計判断と採用理由
- 次に触るべきでないファイル

---

## 12. 初回タスク候補

最初に着手する候補は以下です。

1. `0-1`: Go モジュール、基本ディレクトリ、空の起動点を追加する。
2. `1-1`: `telemetry.proto` と生成手順を追加する。
3. `2-1`: `TCPWorker` の最小型と疑似 TCP テストを追加する。
4. `4-1`: Cloud 側 decoder を Protobuf 生成後に追加する。

`0-1` と `1-1` は後続 PR の土台になるため優先度が高いです。`2-1` と `4-1` はファイル領域が分かれるため、土台確定後に並行しやすいタスクです。
