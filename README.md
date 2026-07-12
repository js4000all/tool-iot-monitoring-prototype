# IoT Monitoring Prototype

Raspberry Pi 上で TCP 対応機器からメトリクスを収集し、MQTT 経由でクラウド側へ転送する IoT 監視システムのプロトタイプです。

本リポジトリはモノレポとして管理し、以下の 2 つの常駐プロセスをシングルバイナリとしてビルドすることを想定します。

- **Edge Agent**: Raspberry Pi 上で動作する収集・送信アプリ
- **Cloud Ingestor**: MQTT メッセージを受信し、メトリクスを処理するクラウド側アプリ

実装言語は Go を想定します。

---

## MVP の最小データフロー

MVP では、次の経路を最小構成で成立させます。

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

受信・復号したメトリクスを Cloud Ingestor が標準出力へ記録できれば MVP として十分です。

詳細な MVP スコープ、非スコープ、受け入れ条件は [docs/MVP.md](docs/MVP.md) を参照してください。

---

## コンポーネント概要

### Edge Agent

Edge Agent は、設定された周期で TCP リクエストを生成し、対象 IP アドレスごとに直列実行します。取得した TCP レスポンスはテレメトリへ変換し、永続キューへ保存してから MQTT へ publish します。

### Cloud Ingestor

Cloud Ingestor は MQTT ブローカーから Telemetry Envelope を受信し、必要に応じて payload を復号・解凍して、TelemetryBatch 内の各メトリクスを `MetricSink` へ渡します。MVP の Sink は標準出力です。

---

## 現状ステータス

このリポジトリは、IoT 監視プロトタイプの設計と実装準備を進めるためのものです。実装ファイルや設定例が追加された後は、以下のコマンドを基本チェックとして使う想定です。

```bash
go build ./...
go test ./...
go vet ./...
```

ローカル MQTT ブローカーや Raspberry Pi 実機が必要な検証は、環境がある場合のみ実施します。検証レベルの詳細は [docs/MVP.md](docs/MVP.md) を参照してください。

---

## 想定リポジトリ構成

```text
.
├── README.md
├── AGENTS.md
├── go.mod
├── cmd/
│   ├── edge-agent/
│   │   └── main.go
│   └── cloud-ingestor/
│       └── main.go
├── internal/
│   ├── edge/
│   ├── cloud/
│   └── shared/
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
├── scripts/
├── test/
└── docs/
```

Go の `internal` 境界を利用し、Edge 固有実装と Cloud 固有実装を分離します。共有パッケージへ業務ロジックを安易に集約せず、本当に同じ契約と意味を持つものだけを共有します。

---

## ドキュメント

詳細は以下を参照してください。

- [docs/README.md](docs/README.md): ドキュメント索引
- [docs/MVP.md](docs/MVP.md): MVP スコープ、非スコープ、受け入れ条件
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): アーキテクチャ、データフロー、設計判断
- [docs/IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md): MVP へ向けた実装計画、PR 分割、進捗管理、タスク間の完了情報継承
- [docs/CONTRIBUTING.md](docs/CONTRIBUTING.md): 開発規約、テスト規約、コミット・PR 規約
- [AGENTS.md](AGENTS.md): AI エージェント向け作業ルール
