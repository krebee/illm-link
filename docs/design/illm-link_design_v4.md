# iLLM Link 設計メモ v4

更新日: 2026-06-24

変更点: プロジェクト名称を `iLLM Link`、リポジトリ名を `illm-link`、CLI名を `illm` に統一。Codex provider ID と設定例を `illm_link` ベースに更新。

## 1. 目的

本設計の目的は、Codex App / Codex CLI から利用するモデルを、`config.toml` の頻繁な切り替えなしで柔軟に選択できるようにすることです。

ローカル環境に **iLLM Link** を配置し、Codex App / Codex CLI からのリクエストを受け取り、指定された `model` 名またはルーティングルールに基づいて、以下の provider へ振り分けます。

- OpenAI / Codex 系モデル
- Z.ai GLM
- ローカルLLM
- その他の Responses API 互換または OpenAI 互換 API

ただし、本設計では **ChatGPT Web 版 backend / 非公開 backend を直接中継しません**。  
また、OpenAI Platform API の従量課金モードを fallback として使うことも、本設計の対象外とします。

**iLLM Link は、認証基盤ではなくモデルルーティング層**として設計します。

---

## 2. 設計方針の要約

本設計の中心方針は次の通りです。

```text
Codex の認証は Codex に任せる
iLLM Link は localhost のモデルルーティングに徹する
ChatGPT Web backend / 非公開 backend は直接利用しない
OpenAI Platform API key への fallback は設計対象外にする
```

推奨する初期構成は次です。

```text
Codex custom provider: illm_link
  + wire_api = "responses"
  + requires_openai_auth = true
  + base_url = "http://127.0.0.1:8787/v1"
```

OpenAI-auth proxy mode が成立しない場合は、OpenAI 系モデルの中継は初期実装では非対応とします。  
この場合でも、Z.ai GLM / local LLM などの他社 provider への iLLM Link としては利用できます。

### 2.1 プロジェクト命名

本プロジェクトの正式名称は **iLLM Link** とします。

| 項目 | 名称 |
|---|---|
| 表示名 | iLLM Link |
| リポジトリ名 | illm-link |
| CLI名 | illm |
| 設定ディレクトリ | `~/.illm` |
| 環境変数プレフィックス | `ILLM_` |
| Codex provider ID | `illm_link` |

`iLLM` は **Interoperable LLM** を意味し、複数の LLM provider を相互運用可能な形で扱う思想を表します。  
`Link` は、Codex、ローカルモデル、外部 LLM API を接続する中継レイヤーであることを示します。

各名称の使い分けは次の通りです。

- **iLLM Link**: README、ドキュメント、UI、ログタイトルなど、人間が読む箇所で使用する
- **illm-link**: GitHub リポジトリ、Docker image、package 名などの公開識別子として使用する
- **illm**: ターミナルから実行する CLI command として使用する
- **illm_link**: Codex の `model_provider` / `model_providers` 用 ID として使用する
- **ILLM_**: iLLM Link 自身の環境変数 prefix として使用する

### 2.2 CLI 設計

CLI は `illm` コマンドとして提供します。頻繁に入力するため、リポジトリ名 `illm-link` より短い `illm` を採用します。

初期実装で提供するコマンドは次の通りです。

| コマンド | 役割 |
|---|---|
| `illm serve` | Responses API 互換 local server を起動する |
| `illm init` | `~/.illm/config.toml` の初期設定を生成する |
| `illm doctor` | API key、Ollama、外部 provider への接続状態を検査する |
| `illm model list` | 利用可能な model 一覧を表示する |
| `illm provider list` | 設定済み provider 一覧を表示する |
| `illm codex setup` | Codex 向け設定例を生成または案内する |

起動例：

```bash
illm serve
```

デフォルトでは、以下の local endpoint で listen します。

```text
http://127.0.0.1:8787/v1
```

iLLM Link 自身の設定ファイルは、以下に配置します。

```text
~/.illm/config.toml
```

環境変数例：

```bash
export ILLM_LOG_LEVEL="info"
export ILLM_CONFIG_PATH="$HOME/.illm/config.toml"
export ZAI_API_KEY="..."
```

---

## 3. 前提と非目標

### 3.1 前提

iLLM Link は、**個人利用を前提とした localhost 専用ツール**です。

想定用途は次の通りです。

- 自分の端末上で Codex App / Codex CLI のモデル選択を簡略化する
- Codex 側の認証・ログインキャッシュ・トークン更新は Codex に任せる
- iLLM Link は provider 選択、request / response 変換、SSE 中継を担当する
- provider ごとの認証情報を厳格に分離する
- 他社 provider へは、受信 header を流用せず、iLLM Link が新規に header を構築する

### 3.2 非目標

iLLM Link は、以下を目的としません。

- ChatGPT / Codex サブスクリプションを汎用 API 化すること
- ChatGPT / Codex サブスクリプションを第三者に提供すること
- ChatGPT Web 版 backend / 非公開 backend を直接中継すること
- `chatgpt_base_url` などを使って ChatGPT backend を router 化すること
- OpenAI の OAuth 認証フローを再実装すること
- OAuth refresh token を自前管理すること
- OpenAI Platform API key の従量課金モードへ fallback すること
- 複数ユーザー向けサービスを提供すること
- SaaS / 社内共用 / 商用バックエンドとして使うこと
- OpenAI API 料金を回避する目的でサブスクリプション経路を利用すること
- provider の制限、レート制限、利用条件を迂回すること

---

## 4. 全体構成

```text
Codex App / Codex CLI
        |
        | Responses API 互換リクエスト
        | custom provider
        | wire_api = "responses"
        | requires_openai_auth = true
        v
127.0.0.1 の iLLM Link
        |
        +-- OpenAI / Codex 系モデル
        |     - Codex 由来の OpenAI 認証を利用可能か検証する
        |
        +-- Z.ai GLM
        |     - Codex 由来 header は破棄
        |     - ZAI_API_KEY で新規 header を構築
        |
        +-- local LLM / Ollama / LM Studio 等
```

iLLM Link は、Codex App / Codex CLI から見ると単一の Responses API 互換エンドポイントとして振る舞います。

内部では `model` 名、provider prefix、設定ファイル、環境変数などに基づいて、実際の送信先 provider を決定します。

---

## 5. 認証モード

### 5.1 採用: OpenAI-auth proxy mode

本設計で採用する OpenAI 系モデル向けの第一候補です。

Codex の custom provider に `requires_openai_auth = true` を設定し、Codex 側が管理する OpenAI 認証を iLLM Link に渡します。

設定例：

```toml
# ~/.codex/config.toml

model = "openai/gpt-5.5"
model_provider = "illm_link"

[model_providers.illm_link]
name = "iLLM Link"
base_url = "http://127.0.0.1:8787/v1"
wire_api = "responses"
requires_openai_auth = true
request_max_retries = 4
stream_max_retries = 5
stream_idle_timeout_ms = 300000
```

この構成の意図は次の通りです。

```text
Codex が OpenAI 認証を管理する
iLLM Link は Responses API 互換エンドポイントとして受ける
iLLM Link は model 名に応じて provider を選択する
```

重要な制約は次の通りです。

- `requires_openai_auth = true` と `env_key` は併用しない
- Codex から iLLM Link へ OpenAI 認証情報が届く可能性がある
- iLLM Link は受信した認証情報をログに出さない
- 他社 provider へ転送する場合、Codex 由来 header はすべて破棄する
- 他社 provider には、iLLM Link 側の provider 固有 API key を使う
- OpenAI / Codex 系モデルへ転送する場合のみ、Codex 由来の OpenAI 認証情報の利用可否を検証する

このモードでは、iLLM Link は OpenAI OAuth の token endpoint や refresh endpoint を直接叩きません。  
認証キャッシュ、期限切れ対応、トークン更新は Codex 側の責務とします。

### 5.2 非採用: API key proxy mode

OpenAI Platform API key による従量課金経路は、本設計では採用しません。

理由：

- iLLM Link の目的は、Codex App / Codex CLI の個人利用体験を改善することです
- OpenAI Platform API の従量課金モードを使う予定がありません
- OpenAI-auth proxy mode が成立しない場合に API key mode へ fallback する運用は想定しません

したがって、OpenAI-auth proxy mode が成立しない場合は、OpenAI 系モデルの中継を初期実装では非対応とします。

### 5.3 非採用: ChatGPT-backend proxy mode

ChatGPT Web 版 backend / 非公開 backend を直接中継する方式は、本設計では採用しません。

非採用理由：

- 非公開 backend は公式 API として安定提供されているものではありません
- payload / header / endpoint が予告なく変更される可能性があります
- リバースエンジニアリングや非公開 endpoint の自動利用に見える設計は、利用条件上のリスクがあります
- 本設計の非目標である「ChatGPT / Codex サブスクリプションの汎用 API 化」と混同される可能性があります
- 長期的な保守性が低いです

したがって、以下は実装しません。

- `chatgpt_base_url` による推論 backend の router 化
- `chatgpt.com/backend-api/...` への直接中継
- ChatGPT backend 固有 header の再現
- ChatGPT backend 固有 payload の再現
- ChatGPT backend の通信観測を前提とした互換実装

---

## 6. Codex 設定方針

### 6.1 設定ファイルの配置場所

provider / auth / base URL 系の設定は、必ず user-level config に置きます。

```text
~/.codex/config.toml
```

プロジェクトローカルの `.codex/config.toml` には、以下のような provider / auth / host-owned metadata 系の設定を置きません。Codex 側で無視される可能性があります。

- `openai_base_url`
- `chatgpt_base_url`
- `model_provider`
- `model_providers`
- `profile`
- `profiles`
- `notify`
- `otel`

### 6.2 設定の優先順位

設定の優先順位は、概ね以下の順で扱います。

```text
CLI flags / --config
  > project-local .codex/config.toml
  > selected profile
  > user-level ~/.codex/config.toml
  > system / managed config
  > built-in defaults
```

ただし、provider / auth / base URL 系の設定は project-local config ではなく user-level config に置くことを原則とします。

### 6.3 `wire_api`

iLLM Link を Codex から使う場合、`wire_api` は `responses` とします。

```toml
wire_api = "responses"
```

`/chat/completions` のみを提供する実装は、Codex App / Codex CLI から直接利用できない前提で扱います。

---

## 7. API 互換方針

### 7.1 必須エンドポイント

初期実装では、以下を必須エンドポイントとします。

```text
POST /v1/responses
```

`/chat/completions` 互換のみの実装は不可とします。

### 7.2 Streaming / SSE

Codex との接続では streaming を基本とします。

iLLM Link は、少なくとも以下の SSE event を正しく扱えるようにします。

```text
response.created
response.output_text.delta
response.output_item.done
response.completed
response.failed
response.error
```

受け入れ条件は次の通りです。

- Codex App / Codex CLI が response text delta を逐次表示できる
- `response.completed` を受けてセッションが正常終了する
- provider 側エラーを `response.failed` または Responses API 互換エラーとして返せる
- SSE connection が idle timeout を超えた場合に明確なエラーを返せる

---

## 8. Responses API 変換仕様

Responses API と Chat Completions 互換 API、Z.ai GLM Messages API、Z.ai GLM API では、request / response の構造が異なります。  
そのため、iLLM Link は単純な pass-through ではなく、provider ごとの mapping layer を持ちます。

### 8.1 初期スコープ

初期実装では、変換対象をテキスト生成と streaming に限定します。

初期対応：

- `instructions`
- `input` の text
- user / assistant の text message
- provider text response
- provider streaming text delta
- usage の基本情報
- error object

初期非対応または後続対応：

- tool call
- function call output
- multimodal input
- image input
- audio input
- encrypted reasoning content
- provider 固有 reasoning trace
- complex stateful response

### 8.2 入力変換: Responses API → provider request

#### 8.2.1 `instructions`

Responses API の `instructions` は、provider ごとの system instruction に変換します。

| Responses API | Z.ai GLM | Chat Completions 互換 | Z.ai GLM |
|---|---|---|---|
| `instructions` | top-level `system` | `messages[0].role = "system"` | system instruction 相当 |

#### 8.2.2 `input` が文字列の場合

```json
{
  "model": "zai/glm-4.5",
  "instructions": "You are a coding assistant.",
  "input": "Explain this error."
}
```

変換例：

```text
system = "You are a coding assistant."
user message = "Explain this error."
```

#### 8.2.3 `input` が item 配列の場合

`input` が typed item 配列の場合、初期実装では text message のみを扱います。

対応対象：

```text
message.role = user
message.role = assistant
content.type = input_text
content.type = output_text
```

未対応 item が含まれる場合は、明示的に `unsupported_input_item` を返します。

### 8.3 出力変換: provider response → Responses API

provider から通常テキスト応答が返った場合、Responses API 互換の response object に変換します。

最低限含める情報：

```text
id
object
created_at
model
output
usage
status
```

`output` には、assistant message item と text content item を構成します。

### 8.4 Streaming 変換

provider 側の streaming chunk を、Responses API 互換 SSE event に変換します。

| provider event | router SSE event |
|---|---|
| stream start | `response.created` |
| text delta | `response.output_text.delta` |
| message completed | `response.output_item.done` |
| stream completed | `response.completed` |
| provider error | `response.failed` または `response.error` |

最低限、Codex 側が text delta と completion を解釈できる event sequence を維持します。

### 8.5 状態管理

初期実装では、iLLM Link は会話履歴を永続保存しません。

方針：

- `store` は provider ごとに扱いを分ける
- `previous_response_id` は初期実装では非対応または best-effort とする
- Codex から渡される `input` に必要な文脈が含まれる前提で扱う
- 永続ストアは採用しない
- 必要な場合のみ、プロセス内メモリで短期 mapping を保持する

---

## 9. ルーティング仕様

### 9.1 決定順序

model から provider を決定する順序は以下です。

```text
1. 明示 provider prefix
2. 完全一致ルール
3. pattern match
4. default provider
5. unresolved error
```

### 9.2 明示 provider prefix

推奨形式は以下です。

```text
openai/gpt-5.5
zai/glm-4.5
local/qwen3-coder
```

parse ルール：

```text
最初の "/" より前を provider とみなす
最初の "/" より後ろ全体を upstream model 名とみなす
```

例：

```text
zai/glm-4.5
provider = "openrouter"
upstream_model = "deepseek/deepseek-chat"
```

### 9.3 model 名の正規化禁止

model 名は原則として変換しません。

禁止例：

```text
gpt-5.4 -> gpt-5-4
claude-sonnet-4.5 -> claude-sonnet-4-5
deepseek/deepseek-chat -> deepseek-deepseek-chat
```

ドット、スラッシュ、コロン、ハイフン、アンダースコアなどは、provider が要求する model 名の一部として保持します。

### 9.4 pattern match

明示 prefix がない場合のみ、以下のような pattern match を行います。

| pattern | provider |
|---|---|
| `gpt-*` | `openai` |
| `codex-*` | `openai` |
| `glm-*` | `zai` |
| `o3*` / `o4*` | `openai` |
| `local/*` | `local` |

曖昧な model 名は default provider に流さず、明示 prefix を要求する方針とします。

---

## 10. Header / 認証情報分離

### 10.1 基本方針

受信 request の header は、upstream provider へ原則として転送しません。

ブラックリスト方式ではなく、**許可リスト方式**で provider ごとに header を新規構築します。

```text
受信 header:
  原則として破棄

router が新規構築する header:
  Content-Type
  Accept
  User-Agent
  provider 固有 Authorization
  provider 固有 API key header
  provider 固有 version header
```

### 10.2 provider 別 header 構築

#### OpenAI / Codex 系

OpenAI-auth proxy mode で OpenAI 系モデルへ転送する場合のみ、Codex 由来の OpenAI 認証情報の利用可否を検証します。

方針：

- Codex 由来の `Authorization` をそのまま使えるか検証する
- OpenAI 系以外には絶対に転送しない
- `Cookie` は転送しない
- `ChatGPT-Account-ID` などの ChatGPT backend 固有 header は扱わない

#### Z.ai GLM

```text
Authorization: Bearer ${ZAI_API_KEY}
Content-Type: application/json
Accept: application/json
```

#### Local LLM

```text
Content-Type: application/json
Accept: application/json
Authorization: Bearer ${LOCAL_LLM_API_KEY}  # 必要な場合のみ
```

### 10.3 転送禁止

以下は provider に転送しません。

```text
受信 Authorization
Cookie
ChatGPT-Account-ID
OpenAI-Organization
OpenAI-Project
OpenAI-Beta
originator
Sec-*
Proxy-*
Forwarded
X-Forwarded-*
traceparent
baggage
```

ただし、上記は「破棄対象リスト」ではなく注意喚起です。  
実装上は、受信 header を元に forwarding header を作らず、provider 別に header を新規構築します。

### 10.4 ログ出力禁止

以下の値はログに出力しません。

- `Authorization` ヘッダー
- API key
- access token
- refresh token
- cookie
- `~/.codex/auth.json` の内容
- provider 別の秘密情報
- prompt / response の全文

ログには、以下のような最小限の情報のみを出力します。

```text
request_id
provider
model
route
status_code
latency_ms
stream_duration_ms
error_kind
retry_count
```

---

## 11. セキュリティ方針

### 11.1 localhost 限定

iLLM Link は `127.0.0.1` のみで listen します。

```text
OK:  127.0.0.1:8787
NG:  0.0.0.0:8787
```

IPv6 を使う場合も loopback のみに限定します。

```text
OK:  [::1]:8787
NG:  [::]:8787
```

### 11.2 ローカル共有シークレット

localhost であっても、同一マシン上の別プロセスからアクセス可能です。  
そのため、必要に応じて簡易な共有シークレットを導入します。

例：

```text
X-ILLM-Token: <random-local-token>
```

方針：

- 初期値は無効でもよい
- 有効化した場合、Codex 側から付与できる header 設定を使う
- token はログに出さない
- token は環境変数または local-only config に置く

### 11.3 外部公開の禁止

以下の起動状態はエラーとして終了します。

- bind address が `0.0.0.0`
- bind address が public IP
- reverse proxy 配下で外部公開されている
- Docker で `-p 0.0.0.0:8787:8787` のように公開されている

---

## 12. タイムアウト・リトライ方針

Codex 側の provider 設定と整合するよう、iLLM Link 側でも同等の方針を持ちます。

推奨初期値：

```text
HTTP request retry count: 4
SSE stream retry count: 5
SSE idle timeout: 300000 ms
provider connect timeout: 10000 ms
provider total non-stream timeout: 120000 ms
```

方針：

- 429 / 500 / 502 / 503 / 504 は指数バックオフ対象
- 400 / 401 / 403 / 404 は原則リトライしない
- streaming 開始後の再試行は重複出力リスクがあるため provider ごとに扱う
- tool call 実行後の自動リトライは原則禁止
- timeout は provider ごとに上書き可能にする

---

## 13. エラーマッピング

iLLM Link は provider 固有エラーを Responses API 互換のエラーへ正規化します。

| provider error | HTTP status | router error code | 方針 |
|---|---:|---|---|
| invalid API key | 401 | `authentication_error` | リトライしない |
| expired OpenAI auth | 401 | `authentication_error` | Codex 側 refresh 挙動を検証する |
| forbidden model | 403 | `permission_denied` | リトライしない |
| model not found | 404 | `model_not_found` | model 名を無変換で返す |
| rate limit | 429 | `rate_limit_exceeded` | backoff 対象 |
| provider timeout | 504 | `provider_timeout` | 条件付き retry |
| upstream 5xx | 502 | `provider_error` | retry 対象 |
| malformed response | 502 | `invalid_provider_response` | retry は provider 別 |
| unsupported field | 400 | `invalid_request_error` | 変換ルール修正対象 |
| unsupported input item | 400 | `unsupported_input_item` | 初期スコープ外として返す |

### 13.1 401 refresh 検証

OpenAI-auth proxy mode では、iLLM Link が 401 を返した場合に Codex 側が自動的に token refresh / 再送を行うかが未確定です。

そのため、Phase 1 で以下を検証します。

```text
1. iLLM Link が意図的に 401 Unauthorized を返す
2. Codex App / Codex CLI が再認証・refresh・再送するか確認する
3. 再送時の Authorization が更新されるか確認する
4. 期待通りでなければ、OpenAI-auth proxy mode は制限付きまたは非対応とする
5. API key mode へ fallback はしない
```

401 応答の body は、OpenAI 互換の error object に寄せます。

例：

```json
{
  "error": {
    "message": "Authentication expired or invalid.",
    "type": "authentication_error",
    "code": "authentication_error"
  }
}
```

---

## 14. 観測性

### 14.1 request id

各リクエストに router 内部の `request_id` を付与します。

```text
request_id = "illm_20260624_xxxxxxxx"
```

provider から request id が返る場合は、以下のように分けて記録します。

```text
router_request_id
upstream_request_id
codex_request_id
```

### 14.2 メトリクス

取得するメトリクス：

```text
request_count
error_count
latency_ms
first_token_latency_ms
stream_duration_ms
output_token_count
input_token_count
retry_count
provider_rate_limit_count
```

### 14.3 ログ粒度

通常ログ：

```text
timestamp
request_id
provider
model
route
status
latency_ms
error_kind
```

debug ログ：

- header 名のみ
- body schema のみ
- prompt / response の全文は出さない
- 認証情報は必ず mask する

---

## 15. テスト・検証方針

### 15.1 受け入れ条件

初期実装の受け入れ条件は次の通りです。

- Codex App / Codex CLI から `/v1/responses` に接続できる
- streaming で `response.output_text.delta` を表示できる
- `response.completed` で正常終了できる
- OpenAI-auth proxy mode の成立可否を検証できる
- Z.ai GLM へ routing できる
- 他社 provider へ受信 header を転送しない
- provider 別 header を iLLM Link 側で新規構築できる
- provider エラーを Responses API 互換エラーに変換できる
- model 名のドット・スラッシュを破壊しない
- 127.0.0.1 以外で起動しようとした場合に停止する

### 15.2 認証モード別テスト

| テスト | 期待結果 |
|---|---|
| OpenAI-auth proxy mode | `requires_openai_auth = true` で iLLM Link に到達できる |
| OpenAI auth forwarding | OpenAI 系 provider にのみ Codex 由来 auth を利用できる |
| 401 from iLLM Link | Codex 側 refresh / 再送挙動を確認できる |
| third-party routing | 受信 header を破棄し、provider key で到達できる |
| ChatGPT backend direct access | 実装しないことを確認する |
| API key fallback | 実装しないことを確認する |

### 15.3 SSE テスト

最低限、以下を検証します。

```text
response.created
response.output_text.delta
response.output_item.done
response.completed
response.failed
response.error
```

stream が途中切断された場合の挙動も検証します。

### 15.4 変換テスト

最低限、以下を検証します。

- `instructions` が provider の system instruction に変換される
- string `input` が user message に変換される
- text delta が `response.output_text.delta` として返る
- 未対応 item を含む場合に `unsupported_input_item` を返す
- provider の rate limit が `rate_limit_exceeded` に変換される

---

## 16. 実装フェーズ

### Phase 1: Minimal Responses proxy

- `POST /v1/responses`
- text input
- streaming
- OpenAI-auth proxy mode の成立検証
- 401 refresh / resend 挙動の検証
- ログ mask
- 127.0.0.1 固定

### Phase 2: Header whitelist / security

- 受信 header の非転送
- provider 別 header の新規構築
- local shared secret
- 起動時 bind address check
- secret / prompt の非ログ化

### Phase 3: Third-party routing

- Z.ai GLM
- provider key 分離
- error mapping
- model prefix routing

### Phase 4: Responses mapping

- `instructions` mapping
- string `input` mapping
- typed text item mapping
- streaming event mapping
- usage 正規化
- unsupported item handling

### Phase 5: Tool calls / advanced compatibility

- basic tool call
- tool result
- provider 差分吸収
- `previous_response_id`
- `store`
- multimodal input
- provider 固有 option mapping

---

## 17. 運用ポリシー

iLLM Link は、次の方針で運用します。

- 自分の端末上でのみ起動する
- 外部ネットワークには公開しない
- 認証情報をログ・画面・エラーに出さない
- provider ごとに認証情報を分離する
- OpenAI / Codex の認証は Codex の正規フローに任せる
- 受信 header は原則として upstream へ転送しない
- provider 別 header は iLLM Link が新規構築する
- 他人のリクエストを処理しない
- 商用サービスや共有サーバーでは使わない
- ChatGPT Web backend / 非公開 backend を直接利用しない
- OpenAI Platform API key への fallback は行わない
- 問題が起きた場合は OpenAI / provider 側の公式利用条件を優先する

---

## 18. まとめ

iLLM Link は、Codex App / Codex CLI のモデル切り替えを簡略化するための、個人利用向けローカルモデルルーターです。

設計上の重要な境界は次の通りです。

> Codex の認証は Codex に任せ、iLLM Link は localhost のモデルルーティングに徹する。

v4 では、追加で次の境界を明確にします。

```text
ChatGPT Web backend / 非公開 backend は直接利用しない
OpenAI Platform API key への fallback は行わない
受信 header は upstream provider へ原則転送しない
Responses API 変換は mapping layer として明示的に設計する
401 refresh 挙動は Phase 1 で検証する
```

OpenAI-auth proxy mode が成立する場合、Codex 側の認証管理を維持しながら、OpenAI / Codex 系モデルと他社モデルを同一の iLLM Link で扱える可能性があります。

OpenAI-auth proxy mode が成立しない場合は、OpenAI 系モデルの中継を初期実装では非対応とし、他社 provider / local LLM 向けのモデルルーターとして運用します。

---

## 参考情報

- OpenAI Codex: Authentication
  - https://developers.openai.com/codex/auth
- OpenAI Codex: Config basics
  - https://developers.openai.com/codex/config-basic
- OpenAI Codex: Advanced Configuration
  - https://developers.openai.com/codex/config-advanced
- OpenAI Codex: Configuration Reference
  - https://developers.openai.com/codex/config-reference
- OpenAI: Migrate to the Responses API
  - https://developers.openai.com/api/docs/guides/migrate-to-responses
- OpenAI: Streaming API responses
  - https://developers.openai.com/api/docs/guides/streaming-responses
