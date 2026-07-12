# MVP Scope

このドキュメントは、IoT Monitoring Prototype の MVP スコープ、非スコープ、受け入れ条件を定義します。

---

## 1. 目的

MVP では、Raspberry Pi 上の Edge Agent が TCP 対応機器からメトリクスを取得し、永続キューへ保存してから MQTT 経由で Cloud Ingestor へ送信する最小経路を成立させます。

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

Cloud 側では、受信・復号したメトリクスを標準出力へ記録できれば十分です。

---

## 2. MVP で実装するもの

### Edge Agent

- 設定された周期で TCP リクエストを生成する
- TCP リクエストを対象 IP アドレス別のキューへ投入する
- 同一 IP アドレスへの TCP リクエストを直列実行する
- 異なる IP アドレスへの TCP リクエストを並列実行する
- TCP レスポンスをテレメトリへ変換する
- テレメトリを永続キューへ保存する
- 保存されたテレメトリを件数またはタイムアウトでバッチ化する
- バッチを Protobuf でシリアライズする
- Protobuf Envelope で包み、MQTT へ publish する
- MQTT 切断中もテレメトリを保持し、再接続後に再送する
- publish 成功後に永続キューから対象レコードを削除する

### Cloud Ingestor

- MQTT ブローカーへ接続する
- テレメトリ用トピックを subscribe する
- Protobuf Envelope を decode する
- Envelope の圧縮方式を確認する
- 必要な場合に payload を解凍する
- TelemetryBatch を decode する
- 各テレメトリを標準出力へ記録する

---

## 3. MVP では実装しないもの

- クラウドから Edge Agent への command 送信
- command 結果の返却
- API セッション管理
- MQTT による同期 RPC
- TCP リクエストの優先順位
- TCP リクエスト実行キューの永続化
- command の冪等性管理
- MQTT 認証
- MQTT TLS
- 複数 MQTT ブローカーへのフェイルオーバー
- テレメトリ圧縮
- RDB への保存
- OpenTelemetry への送信
- 設定の動的リロード
- Edge Agent の自己更新

将来の command 対応を妨げない構造は残しますが、MVP の実装対象には含めません。

---

## 4. 受け入れ条件

MVP は以下を満たした時点で成立とします。

1. Edge Agent を Raspberry Pi 向けにビルドできる
2. Cloud Ingestor をビルドできる
3. ローカル MQTT ブローカーを使った E2E テストが通る
4. Edge Agent が TCP レスポンスを SQLite へ保存できる
5. MQTT 停止中もデータが SQLite へ残る
6. MQTT 復旧後に蓄積データが送信される
7. Cloud Ingestor が受信メトリクスを標準出力へ表示する
8. Edge Agent と Cloud Ingestor を単一バイナリで起動できる

---

## 5. 検証レベル

開発環境の制約により、すべての受け入れ条件を常に実行できるとは限りません。検証は段階を分けます。

### Level 1: 常に実行する

- `go build ./...`
- `go test ./...`
- Protobuf 互換性テスト
- SQLite 永続化テスト
- 疑似 TCP サーバーによるテスト

### Level 2: 環境に MQTT ブローカーを起動する手段がある場合

- MQTT を含む E2E
- ブローカー停止・復旧試験

### Level 3: 実機または ARM 実行環境がある場合

- Raspberry Pi 上で起動
- systemd 常駐確認
- 長時間運転

実行できなかった受け入れ条件は、未実施理由と必要な環境を明記してください。実施済みのように扱ってはいけません。
