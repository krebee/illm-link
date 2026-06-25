# iLLM Link 技術設計で決めること

更新日: 2026-06-25

## 1. 文書の位置づけ

本書は、iLLM Link の技術設計をいちから決め直すための論点一覧です。  
ここでは仕様を確定せず、決定が必要な項目だけを整理します。

---

## 2. 外部インターフェース

- Codex から受け付ける API エンドポイント
- 対応する OpenAI API 互換範囲
- streaming / non-streaming の対応範囲
- Codex App と Codex CLI で同じ仕様にするか
- unsupported request をどの形式で返すか
- API version や互換性レベルの表現方法

---

## 3. Responses API 変換

- Responses API request から provider request への変換対象
- `instructions` の扱い
- `input` が文字列の場合の扱い
- `input` が item 配列の場合の扱い
- message role の対応範囲
- text content の対応範囲
- tool call の対応有無
- function call output の対応有無
- multimodal input の対応有無
- reasoning / encrypted reasoning の対応有無
- provider response から Responses API response への変換形式
- usage 情報の正規化方針
- response id / created_at / status の生成方針

---

## 4. Streaming 変換

- provider stream から Responses API SSE への event mapping
- 最小対応 event sequence
- delta の粒度
- stream start / completion の表現
- stream 中 error の表現
- stream 途中切断時の扱い
- retry 可能な streaming error の範囲
- provider ごとの差分吸収方法

---

## 5. ルーティング

- model 名から provider を決定する方法
- 明示 provider prefix を採用するか
- provider prefix の構文
- prefix がない model 名の扱い
- 完全一致ルールの有無
- pattern match ルールの有無
- default provider の有無
- 曖昧な model 名の扱い
- upstream model 名を正規化するか
- provider ごとの model alias を持つか
- routing 設定の保存形式

---

## 6. Provider 抽象

- provider adapter の責務範囲
- provider ごとの request builder の設計
- provider ごとの response parser の設計
- provider ごとの streaming parser の設計
- provider capability の表現方法
- provider 固有設定の持ち方
- local LLM と remote provider を同じ抽象で扱うか
- OpenAI 互換 provider の共通化範囲

---

## 7. 認証・ヘッダー

- Codex から受け取った認証情報を利用するか
- OpenAI-auth proxy mode を実装するか
- API key mode を実装するか
- provider 別 secret の設定方法
- upstream provider へ転送する header の方針
- header を allowlist で新規構築するか
- 転送禁止 header の扱い
- `Cookie` の扱い
- `Authorization` の扱い
- OpenAI 系 provider と third-party provider の認証分離方法
- 認証情報をログに出さないための仕組み

---

## 8. 設定

- 設定ファイルの形式
- 設定ファイルの配置場所
- 環境変数で上書きできる項目
- provider 定義の形式
- routing 定義の形式
- timeout / retry の設定粒度
- observability 設定の形式
- 設定 reload の要否
- secret を設定ファイルに含めるか

---

## 9. タイムアウト・リトライ

- provider 接続 timeout
- non-streaming request timeout
- streaming idle timeout
- retry 対象 status code
- retry 対象 error kind
- retry 回数
- backoff 方針
- streaming 開始後の retry 可否
- tool call を含む request の retry 可否
- provider ごとの timeout override の有無

---

## 10. エラーマッピング

- provider error から router error への正規化形式
- HTTP status の決め方
- OpenAI 互換 error object に寄せるか
- Responses API 互換 error event に寄せるか
- authentication error の扱い
- permission error の扱い
- model not found の扱い
- rate limit の扱い
- provider timeout の扱い
- malformed provider response の扱い
- unsupported feature の扱い
- 401 を返したときの Codex 側挙動を検証するか

---

## 11. 状態管理

- `store` の扱い
- `previous_response_id` の扱い
- conversation state を保持するか
- 永続ストアを持つか
- プロセス内メモリを使うか
- response id と provider request の対応を保持するか
- state を保持しない場合の制約

---

## 12. 観測性

- request id の採番方法
- Codex request id / router request id / upstream request id の扱い
- 通常ログに出す項目
- debug ログに出す項目
- ログに出してはいけない項目
- prompt / response 本文を記録するか
- metrics の収集項目
- tracing の有無
- structured logging の形式

---

## 13. セキュリティ

- bind address の制限
- localhost 以外で起動する場合の扱い
- CORS の扱い
- request body size limit
- prompt / response の秘匿方針
- secret masking の方法
- third-party provider への情報漏えい防止策
- config / log / crash dump に secret を残さない方針

---

## 14. テスト・検証

- Codex App 接続テスト
- Codex CLI 接続テスト
- Responses API 変換テスト
- streaming event 変換テスト
- routing テスト
- provider 別 header 構築テスト
- 認証モード別テスト
- timeout / retry テスト
- error mapping テスト
- unsupported feature テスト
- secret がログに出ないことのテスト
- localhost 制限のテスト
- provider mock の作り方

---

## 15. 実装単位

- 最初に実装する最小スコープ
- Phase 1 に含める provider
- Phase 1 から除外する機能
- provider adapter の追加順序
- CLI / server 起動方法
- 設定ファイルの初期サンプル
- 受け入れ条件
- 後続 phase へ回す項目

---

## 16. ADR 化する判断

- API 互換範囲
- routing 方式
- provider prefix 方式
- 認証情報の扱い
- header forwarding 方針
- stateful / stateless 方針
- streaming retry 方針
- provider abstraction 方針
- observability 方針
- security boundary
