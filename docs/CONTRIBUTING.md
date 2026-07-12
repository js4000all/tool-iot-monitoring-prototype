# Contributing Guide

このリポジトリへの変更は、小さく、意図が明確で、検証可能であることを重視します。

本ガイドは、人間の開発者とAIエージェントの双方を対象とします。システム全体の目的、MVPスコープ、アーキテクチャ上の設計判断については、先に[README.md](../README.md)を参照してください。

---

## 1. 基本方針

- 読みやすさを短さより優先する
- 変更範囲を必要最小限にする
- MVPに不要な機能を先回りして実装しない
- 既存の責務境界を尊重する
- 外部I/O、永続化、通信、データ変換を可能な範囲で分離する
- エラーを握りつぶさない
- テストできない設計を避ける
- 抽象化は、実際の変更軸または複数実装がある箇所に置く
- 将来使うかもしれないという理由だけでinterfaceや汎用基盤を作らない
- Composition Rootを依存関係の組み立て場所として維持する

---

## 2. 開発の進め方

変更を始める前に、最低限以下を整理してください。

1. 変更対象はEdge Agent、Cloud Ingestor、共有プロトコルのどれか
2. 変更によってデータがどの経路を通るか
3. 永続化される地点はどこか
4. 障害時に失われるものと残るものは何か
5. Protobuf互換性への影響があるか
6. Composition Rootの変更が必要か
7. MVPスコープ外へ踏み込んでいないか

大きな変更は、実装前に小さな単位へ分けてください。

例:

```text
1. Protobuf定義を追加
2. Edge側の生成処理を追加
3. 永続キューへ保存
4. publish処理へ接続
5. Cloud側のdecode処理を追加
6. E2Eテストを追加
```

---

## 3. Goコーディング規約

### 3.1 フォーマットと静的検査

変更後は、可能な範囲で以下を実行してください。

```bash
gofmt -w .
go test ./...
go vet ./...
```

CIに追加のチェックがある場合は、それも実行してください。

### 3.2 package設計

- package名は短く、単数形を基本とする
- `common`、`util`、`helper`のような広すぎるpackageを避ける
- Edge固有とCloud固有の処理を安易に共有packageへ移さない
- 共有するのは、本当に同じ契約と意味を持つものに限定する
- `internal`境界を維持する

### 3.3 interface

- interfaceは利用側のpackageで定義することを基本とする
- 実装が1つしかなく、差し替え理由もテスト上の必要性もない場合は、無理にinterfaceを作らない
- interfaceは小さく、利用側が必要とする操作だけを持たせる
- 実装詳細を漏らす戻り値や引数を避ける
- 同じメソッド名でも成功条件が異なる実装を混在させない

例:

```go
type TelemetryQueue interface {
    Enqueue(ctx context.Context, item TelemetryItem) error
}
```

`Enqueue`成功は、永続キューへの保存完了を意味します。MQTT publish完了は意味しません。

### 3.4 constructorと依存関係

- 必要な依存はconstructorで受け取る
- 不完全な状態のオブジェクトを返さない
- 起動後に変更しない依存はconstructorで確定する
- HandlerやRouteの登録はconstructorで完了させる
- 実行時に依存関係を追加・差し替えしない
- 業務上の初期化を`init()`へ置かない

### 3.5 context

- `context.Context`は第1引数とする
- `context.Context`をstructへ保存しない
- timeoutやcancelは呼び出し元から伝播させる
- 長時間処理やgoroutineはcontext終了を監視する

### 3.6 goroutineとchannel

- goroutineを起動する箇所では、停止条件を明確にする
- エラーの伝播方法を明確にする
- goroutineを無制限に生成しない
- channelの所有者を明確にする
- channelをcloseする責任は、原則として生成側が持つ
- 同一IPへのTCPリクエスト直列化を壊さない
- 異なるIP間の並列実行を不要に阻害しない

### 3.7 timeoutとretry

- timeoutをコードへ散在させない
- timeoutは設定または呼び出し元から受け取る
- MVPで不要な複雑なRetry Policyを作らない
- retryを追加する場合は、重複実行の影響を確認する
- TCP、MQTT、永続化のretry責務を混在させない

### 3.8 error

- エラーを握りつぶさない
- エラーには処理対象を特定できる文脈を付ける
- `%w`でラップし、原因判定を妨げない
- 呼び出し元が判断すべきエラーをログだけで終わらせない
- panicは継続不能な初期化失敗または不変条件違反に限定する

例:

```go
return fmt.Errorf(
    "enqueue telemetry: sensor_id=%q target_ip=%q: %w",
    item.SensorID,
    targetIP,
    err,
)
```

---

## 4. 命名規則

型名とメソッド名は責務を具体的に表してください。

推奨する用語:

```text
Processor   処理全体の入口
Handler     特定の入力を処理する
Resolver    振り分け先を解決する
Queue       メッセージや処理待ちを保持する
Store       永続化を担う
Publisher   外部へ送信する
Subscriber  外部から受信する
Scheduler   時間起点で処理を生成する
Encoder     データをシリアライズする
Decoder     データをデシリアライズする
Sink        処理済みデータの出力先
```

避ける名前:

```text
Manager
Helper
Util
Common
Service
Data
Info
Object
```

これらを使う場合は、その名前でなければならない理由が明確であることを求めます。

一般的な頭字語は表記を統一します。

```text
ID
IP
URL
MQTT
TCP
HTTP
```

---

## 5. アーキテクチャ上の規約

### 5.1 TCPWorker

TCPWorkerは以下に責任を持ちます。

- TCPリクエストの実行
- timeoutの適用
- TCPResultの生成
- ResultProcessorへの受け渡し

TCPWorkerは以下を知りません。

- テレメトリのProtobuf形式
- MQTTトピック
- SQLiteスキーマ
- クラウド側の保存先
- command固有のAPIセッション

### 5.2 ResultProcessor

ResultProcessorは結果処理の共通入口です。

- ResultRouteから適切なResultHandlerへ委譲する
- Handlerの具体実装を呼び出し元から隠蔽する
- 共通ログ、メトリクス、エラー整形を必要に応じて扱う
- 用途固有の変換処理を自身へ追加しない

ResultProcessorへ`switch`や`if`による用途別処理が増え始めた場合は、ResultHandlerへ分離してください。

### 5.3 ResultHandler

ResultHandlerは以下に責任を持ちます。

- CommandとTCPResultの解釈
- 用途別メッセージの生成
- 対応するQueueへの投入

ResultHandlerは以下を扱いません。

- MQTT接続管理
- 再接続
- バッチ送信
- 圧縮
- publish retry
- publish成功後の削除

### 5.4 QueueとPublish Worker

- 取得済みテレメトリは永続化してからバッチ化する
- publish失敗時はレコードを残す
- publish成功確認後に対象レコードを削除する
- 永続化成功とpublish成功を混同しない
- MVPでは汎用分散キューを作らない
- MVPではTelemetryQueueに必要な操作だけを実装する

### 5.5 Cloud MetricSink

Cloud Ingestorは、decodeしたTelemetryItemをMetricSinkへ渡します。

MVP実装は`StdoutMetricSink`です。

将来のRDBやOpenTelemetry対応は、MetricSinkの実装追加として行います。Cloud IngestorのMQTT受信処理へ保存先固有ロジックを直接追加しないでください。

---

## 6. Protobuf規約

- MQTT payloadはProtobufを使用する
- 独自バイナリヘッダーを作らない
- 圧縮状態はProtobuf Envelopeで表現する
- 既存フィールド番号を変更・再利用しない
- 削除したフィールド番号と名前は`reserved`にする
- enumの`0`は安全なデフォルト値にする
- 未知のenum値を無条件に正常扱いしない
- EdgeとCloudの双方でencode/decodeテストを行う
- 生成コードは`.proto`変更と同じ変更セットに含める

例:

```proto
enum Compression {
  COMPRESSION_NONE = 0;
  COMPRESSION_ZSTD = 1;
}
```

MVPでは`COMPRESSION_NONE`のみを使用します。

---

## 7. ログ規約

ログは構造化ログを基本とします。

可能な範囲で以下を含めてください。

```text
component
site_id
gateway_id
target_ip
command_id
result_route
message_id
batch_size
```

方針:

- payload全体を通常ログへ出さない
- 認証情報や秘密情報を出さない
- TCPレスポンスの生データはdebugレベルまたは明示設定時のみ出す
- 同じエラーを複数階層で重複してログ出力しない
- 原則として、エラーを最終的に処理する境界でログを出す
- recover可能な失敗と致命的な失敗を区別する
- MQTT切断中の連続エラーでログを過剰出力しない

---

## 8. 設定規約

- 必須設定の欠落は起動時に失敗させる
- IP、ポート、duration、件数などは起動時に検証する
- 不正な設定を暗黙のデフォルトへ置き換えない
- デフォルト値は1か所に集約する
- 設定例を変更した場合はドキュメントも更新する
- MVPでは動的リロードを実装しない
- 秘密情報をリポジトリへコミットしない
- 環境変数と設定ファイルの優先順位を明示する

---

## 9. テスト規約

### 9.1 基本方針

- バグ修正には可能な限り再現テストを追加する
- private実装ではなく、公開契約または振る舞いをテストする
- 実時間に依存する長いsleepを避ける
- 偶然通る並行処理テストを避ける
- 外部環境依存テストと単体テストを分離する

### 9.2 Edge Agent

最低限、以下を検証します。

- 同一IPへのTCPリクエストが直列実行される
- 異なるIPへのTCPリクエストが並列実行される
- TCP成功結果がTelemetryItemへ変換される
- TCP失敗時の挙動が明確である
- SQLiteへテレメトリが永続化される
- publish失敗時にレコードが残る
- publish成功時に対象レコードが削除される
- 最大件数でバッチ送信される
- 最大待機時間でバッチ送信される

TCPテストには、ローカルの疑似TCPサーバーを使用してください。

SQLiteテストには、一時ディレクトリまたは一時DBを使用してください。

### 9.3 Cloud Ingestor

最低限、以下を検証します。

- TelemetryEnvelopeをdecodeできる
- `COMPRESSION_NONE`を処理できる
- 未対応compressionを安全に拒否する
- TelemetryBatchをdecodeできる
- 各TelemetryItemをMetricSinkへ渡せる
- StdoutMetricSinkが期待形式で出力する

### 9.4 E2E

MQTTブローカーを利用できる環境では、以下を検証します。

- EdgeからCloudまでテレメトリが到達する
- MQTT停止中もSQLiteへデータが残る
- MQTT復旧後に蓄積データが送信される

実行環境の制約でE2Eを実行できない場合は、未実施理由と必要な環境を明記してください。

---

## 10. コミット規約

### 10.1 基本方針

- 1コミットは1つの意図に集中させる
- リファクタリングと機能変更を可能な範囲で分ける
- 無関係な整形変更を混ぜない
- 動作しない中間状態を原則としてコミットしない
- 未追跡ファイルや他者の変更を勝手に削除しない
- ユーザーの明示なしに履歴を書き換えない
- `git push --force`を行わない
- 他者のコミットを勝手にrebase、squash、amendしない

### 10.2 コミットメッセージ

Conventional Commitsに近い形式を使用します。

```text
<type>(<scope>): <summary>
```

例:

```text
feat(edge): add per-IP TCP execution queue
fix(cloud): reject unsupported compression values
test(edge): cover persisted telemetry recovery
refactor(shared): simplify telemetry envelope decoding
docs(repo): add contribution guidelines
build(proto): regenerate telemetry protobuf code
```

使用するtype:

```text
feat      新機能
fix       バグ修正
refactor  振る舞いを変えない構造改善
test      テスト追加・修正
docs      ドキュメント
build     ビルド、依存、生成コード
ci        CI設定
chore     その他の保守作業
```

推奨scope:

```text
edge
cloud
proto
shared
repo
```

ルール:

- summaryは簡潔にする
- 末尾に句点を付けない
- `update files`、`fix issue`のような曖昧な表現を避ける
- 破壊的変更がある場合は本文に`BREAKING CHANGE:`を記載する
- 変更理由が自明でない場合は本文へ背景を書く

---

## 11. Pull Request規約

Pull Requestには、最低限以下を記載してください。

```text
## Summary
何を変更したか

## Reason
なぜ変更したか

## Design
主要な設計判断

## Tests
実行したテスト

## Not tested
実行できなかったテストと理由

## Risks
既知の制約、互換性、運用上の注意
```

大きな変更では、データフローまたは依存関係の変化も記載してください。

レビューしやすくするため、可能な範囲で以下を守ってください。

- PRを小さく保つ
- 生成コードだけの巨大差分を説明する
- 無関係なリネームを混ぜない
- 設計変更と単純な機能追加を分ける
- READMEや設定例への影響を反映する

---

## 12. 変更完了時の報告

作業完了時には、以下を簡潔に報告してください。

```text
- 変更した内容
- 主な設計判断
- 実行したテスト
- 実行できなかったテストと理由
- 残っている制約または既知の課題
```

コミットを作成した場合は、コミットハッシュとコミットメッセージも記載してください。

未実施のテストを成功扱いにしないでください。

---

## 13. 避けるべき過剰設計

MVPでは、明確な必要性がない限り以下を導入しません。

- 汎用MessageBus
- 汎用Workflow Engine
- 動的Plugin機構
- Handlerの実行時追加
- 動的DIコンテナ
- 独自シリアライズ形式
- 汎用分散キュー
- 複雑なRetry Policy階層
- 複数圧縮方式の自動選択
- commandを想定した未使用コード
- 利用箇所のないinterface
- 将来用途だけを理由とした共有package

抽象化は、確認できた変更軸に対して行ってください。

---

## 14. Definition of Done

変更は、該当する範囲で以下を満たした時点で完了とします。

- コードがビルドできる
- `gofmt`が適用されている
- 関連する単体テストが通る
- `go vet ./...`で新たな問題がない
- エラー処理が追加されている
- 設定変更が検証されている
- Protobuf互換性が維持されている
- 必要な生成コードが更新されている
- 関連ドキュメントが更新されている
- 実行できなかった検証が明記されている
