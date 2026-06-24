# iLLM Link

**iLLM Link** は、Codex App と Codex CLI 向けのローカル LLM ルーティング層です。

localhost 上に OpenAI Responses API 互換のエンドポイントを公開し、要求されたモデル名に基づいてローカルモデルやサードパーティ製 LLM プロバイダーへリクエストを転送します。

## 概要

iLLM Link の目的は、Codex でモデルを切り替える際に、頻繁に `~/.codex/config.toml` を編集しなくても済むようにすることです。

Codex は iLLM Link を単一のカスタムプロバイダーとして接続し、iLLM Link は次のようなプロバイダーへリクエストを振り分けます。

- OpenAI / Codex 互換モデル
- Z.ai GLM
- Ollama や LM Studio などのローカル LLM
- その他 Responses API 互換または OpenAI 互換プロバイダー

```text
Codex App / Codex CLI
        |
        | Responses API 互換のリクエスト
        v
iLLM Link 127.0.0.1:8787
        |
        +-- OpenAI / Codex 互換モデル
        +-- Z.ai GLM
        +-- ローカル LLM
        +-- その他互換プロバイダー
```

## プロジェクト名

| 項目 | 名前 |
|---|---|
| 製品名 | iLLM Link |
| リポジトリ | illm-link |
| CLI コマンド | illm |


## スコープ

iLLM Link は **localhost のみで使う個人向けツール** として設計されています。

想定される用途:

- Codex からのリクエストを異なるモデルプロバイダーへ転送する
- プロバイダーごとの認証情報を分離する
- 必要に応じてリクエストやレスポンス形式を変換する
- プロバイダーのレスポンスを Codex へストリーミングで返す

意図されていない用途:

- ChatGPT や Codex のサブスクリプションを汎用 API として公開する
- ChatGPT の非公開 Web バックエンドエンドポイントを代理する
- OpenAI OAuth フローを再実装する
- プロバイダーの制限、価格、利用規約を回避する
- 共有 SaaS やマルチユーザー向けバックエンドとして運用する

## 初期実装スコープ

最初の実装では、次の点に重点を置きます。

- `POST /v1/responses`
- テキスト入力とテキスト出力
- ストリーミング / SSE
- モデルプレフィックスによるプロバイダー振り分け
- プロバイダー間のヘッダー分離
- 基本的なエラー変換
- localhost のみでバインドするサーバー

ツール呼び出し、マルチモーダル入力、永続的な会話状態、プロバイダー固有の reasoning trace などの高度な機能は、後続フェーズで計画されています。

## ステータス

設計フェーズです。

## ライセンス

This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.

