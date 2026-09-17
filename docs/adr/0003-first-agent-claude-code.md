# ADR-0003: 最初に対応する Agent は Claude Code(claude-agent-acp)とする

- 日付: 2026-09-16(2026-09-17 に自分用プロジェクトとしての前提を追記)
- 状態: 採用(MVP)

## 背景

MVP は 1 Agent に絞る。候補は Claude Code と Codex。どちらも ACP アダプタが存在する。

| | Claude Code | Codex |
|---|---|---|
| ACP アダプタ | `@agentclientprotocol/claude-agent-acp` v0.78.0(2026-09-15)、2.5k★、Claude Agent SDK 上 | `@agentclientprotocol/codex-acp` v1.12.0、TypeScript、app-server 上。旧 Rust 版は 2026-07 アーカイブ |
| アダプタの利用実績 | Zed / JetBrains / Emacs / Neovim / acp-ui など、ACP クライアントの参照実装として最も使われている | 同じクライアント群で利用可 |
| 認証(技術) | `terminal` 型: `claude auth login --claudeai`(サブスク)/ `--console`(API 課金) | ChatGPT login / `CODEX_API_KEY` |
| 認証(公式方針) | 第三者開発者が claude.ai ログインを提供 / 資格情報を仲介することは不許可。個人が自作クライアントを自分のサブスクで使う可否は明文なし。課金変更は 2026-06-16 に延期、再変更予告あり | CEO が第三者ハーネスでのサブスク利用を公言(2026-05-01)。明文の許可規約は未確認。再販・第三者サービス提供は禁止 |
| 設定継承 | CLAUDE.md / settings.json / hooks / .mcp.json / skills を SDK 既定で読む | config.toml / AGENTS.md を CODEX_HOME 経由で読む |
| diff | `tool_call.content.diff` | 同左 |
| ブランド | 「Claude Code」を製品名に使えない | 制約の記述なし |

## 決定

**Claude Code を最初の Agent とする。** Codex は MVP 後の最初の拡張として設定ファイルの起動コマンドを差し替えて動作検証する。

## 理由

1. **アダプタの成熟度**。claude-agent-acp は ACP の参照実装的な位置にあり、terminal / edit review / modes / slash commands / session load・resume・fork まで揃っている。ACP の全機能を最初に踏むのに適している
2. **認証フローが ACP に明示されている**。`terminal` 型の authMethods(`claude-ai-login` / `console-login`)を返すので、アプリはコマンドを別ターミナルで起動するだけで済み、トークンに触れない設計が自然に成立する
3. **開発者自身が Claude Code を日常的に使っている**ため、動作の妥当性を判断しやすい。口調の設定も output style という正式機能で済む
4. Codex を先にしない理由は「劣る」からではなく、アダプタが 2026-07 に全面書き換え(Rust → TS)されたばかりで、挙動の安定を見極めたいため

## 前提と注意(重要)

- **サブスクリプションでの利用可否は Anthropic の利用条件に依存し、変更され得る**。自分用でも次を守る(コストがかからず、後で効く)
  - アプリは API キー / OAuth トークンを保存・仲介・収集しない
  - 認証は無改変の `claude` バイナリ自身のフロー(`claude auth login`)に委ねる。法務ページの「unmodified Claude Code binary with their own Claude subscription」に沿う形
  - **API キーでも必ず通す**ようにしておく(規約変更時の退路)
  - 個人が自分のサブスクで自作クライアントを使うことが許容されるかは **公式に確認できていない**。README にその旨を明記する
- 配布する場合は「Claude Code」を製品名に含められない(ブランド規約)。自分用の間は無関係
- アダプタのバージョンを固定する(`npx @agentclientprotocol/claude-agent-acp@0.78.0`)

## 結果

- 設定ファイルの既定 Agent は Claude Code(claude-agent-acp)
- 受け入れ条件(`docs/mvp.md` §3)は Claude Code で検証する
- Codex 対応は MVP 後に別 ADR を書かずに設定変更 + 検証で進める(アーキテクチャ上の変更が不要なため)
