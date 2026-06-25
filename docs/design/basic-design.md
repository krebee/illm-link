# iLLM Link 基本設計

更新日: 2026-06-25

## 1. 文書の位置づけ

本書は、iLLM Link の基本設計を定義します。

詳細な API 変換、streaming、routing、header 分離、error mapping、検証条件は、[iLLM Link 技術設計](./technical-design.md) に分離します。

## 2. 目的

iLLM Link は、Codex App / Codex CLI から利用する LLM を、Codex 側の設定を頻繁に切り替えずに選択できるようにするローカルルーティング層です。

Codex からは単一の Responses API 互換 provider として見せ、iLLM Link が `model` 名や設定に基づいて、ローカルモデルまたは外部 LLM provider へ振り分けます。

主な接続先は次を想定します。

- OpenAI / Codex 系モデル
- Z.ai GLM
- ローカル LLM
- Responses API 互換または OpenAI 互換 API

iLLM Link は認証基盤ではなく、個人利用向けの localhost 専用モデルルーターとして設計します。

## 3. 基本方針

設計上の中心方針は次の通りです。

- Codex の認証は Codex に任せます。
- iLLM Link は localhost のモデルルーティングに徹します。
- provider ごとの認証情報は分離します。
- 受信 header は upstream provider へ原則転送せず、provider ごとに必要な header を構築します。
- ChatGPT Web 版 backend / 非公開 backend は直接利用しません。
- OpenAI Platform API key への fallback は初期設計の対象外とします。

OpenAI-auth proxy mode が成立しない場合、OpenAI 系モデルの中継は初期実装では非対応とします。その場合でも、Z.ai GLM や local LLM 向けのルーターとして利用可能です。

## 4. プロジェクト命名

正式名称は **iLLM Link** とします。

| 項目 | 名称 |
|---|---|
| 表示名 | iLLM Link |
| リポジトリ名 | illm-link |
| CLI 名 | illm |
| 設定ディレクトリ | `~/.illm` |
| 環境変数プレフィックス | `ILLM_` |
| Codex provider ID | `illm_link` |

`iLLM` は Interoperable LLM を意味し、複数 provider の相互運用を表します。`Link` は Codex、ローカルモデル、外部 LLM API を接続する中継層であることを表します。

## 5. 対象範囲

### 5.1 前提

iLLM Link は、個人利用を前提とした localhost 専用ツールです。

想定用途は次の通りです。

- 自分の端末上で Codex App / Codex CLI のモデル選択を簡略化します。
- Codex 側の認証・ログインキャッシュ・トークン更新は Codex に任せます。
- iLLM Link は provider 選択、request / response 変換、streaming 中継を担当します。
- provider ごとの secret を分離し、ログやエラーに出しません。

### 5.2 非目標

iLLM Link は、以下を目的としません。

- ChatGPT / Codex サブスクリプションを汎用 API 化すること
- ChatGPT Web 版 backend / 非公開 backend を直接中継すること
- OpenAI OAuth フローを再実装すること
- OpenAI Platform API key の従量課金経路へ fallback すること
- 複数ユーザー向けサービス、SaaS、社内共用バックエンドとして提供すること
- provider の制限、レート制限、利用条件を迂回すること

## 6. 全体構成

```text
Codex App / Codex CLI
        |
        | Responses API 互換リクエスト
        v
iLLM Link
        |
        +-- OpenAI / Codex 系モデル
        +-- Z.ai GLM
        +-- local LLM / Ollama / LM Studio 等
        +-- その他の互換 provider
```

Codex から見た iLLM Link は単一の Responses API 互換 endpoint です。内部では `model` 名、provider prefix、設定ファイル、環境変数などに基づいて送信先 provider を決定します。

## 7. 外部インターフェース方針

Codex からの利用では、Responses API 互換 endpoint を提供します。

初期実装の必須 endpoint は次の通りです。

```text
POST /v1/responses
```

`/chat/completions` のみを提供する実装は、Codex App / Codex CLI 向けの基本構成には含めません。

## 8. Codex 設定方針

Codex からは custom provider として iLLM Link を利用します。

想定する初期構成は次の通りです。

```toml
model = "zai/glm-4.5"
model_provider = "illm_link"

[model_providers.illm_link]
name = "iLLM Link"
base_url = "http://127.0.0.1:8787/v1"
wire_api = "responses"
requires_openai_auth = true
```

provider / auth / base URL 系の設定は user-level config に置きます。

```text
~/.codex/config.toml
```

## 9. ルーティング方針

model 名から provider を決定します。

基本方針は次の通りです。

- 明示 provider prefix を優先します。
- model 名のドット、スラッシュ、コロン、ハイフン、アンダースコアは破壊しません。
- 曖昧な model 名は暗黙の default provider に流さず、明示 prefix を要求します。

推奨形式は次の通りです。

```text
openai/gpt-5.5
zai/glm-4.5
local/qwen3-coder
```

詳細な routing rule は技術設計で定義します。

## 10. セキュリティ方針

iLLM Link は localhost のみで listen します。

```text
OK:  127.0.0.1:8787
NG:  0.0.0.0:8787
```

外部公開される起動状態はエラーとして扱います。Docker や reverse proxy を使う場合も、外部ネットワークへ公開しないことを前提とします。

受信 request の header は、upstream provider へ原則として転送しません。provider ごとに必要な header を iLLM Link が新規構築します。

## 11. 観測性方針

各リクエストに router 内部の `request_id` を付与します。

通常ログには、運用上必要な最小限の情報のみを出力します。

- request id
- provider
- model
- route
- status
- latency
- error kind

認証情報、API key、access token、cookie、prompt / response 全文は通常ログに出力しません。

## 12. 初期実装スコープ

初期実装では、次を最小スコープとします。

- `illm serve` による localhost server 起動
- `POST /v1/responses` の受け口
- 明示 provider prefix による routing
- provider ごとの header 構築
- streaming 中継の最小対応
- local LLM または外部 provider への接続
- secret をログに出さない仕組み

CLI の追加機能、詳細な provider capability、state 管理、retry、error mapping、metrics は技術設計で範囲を定義します。

## 13. まとめ

iLLM Link は、Codex App / Codex CLI のモデル切り替えを簡略化するための、個人利用向けローカルモデルルーターです。

設計上の重要な境界は次の通りです。

> Codex の認証は Codex に任せ、iLLM Link は localhost のモデルルーティングに徹します。

この境界を維持するため、ChatGPT Web 版 backend / 非公開 backend の直接利用、OpenAI Platform API key への fallback、受信 header の無制限転送は行いません。
