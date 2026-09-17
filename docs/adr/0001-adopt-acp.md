# ADR-0001: Agent との通信に ACP(Agent Client Protocol)を採用する

- 日付: 2026-09-16
- 状態: 採用(MVP)

## 背景

既存の Claude Code / Codex をバックエンドとして使うクライアントを作る。接続経路の候補は次の 3 つ。

| 経路 | Claude Code | Codex |
|---|---|---|
| A. ACP アダプタ | `@agentclientprotocol/claude-agent-acp`(Claude Agent SDK 上に実装、Anthropic 非公式、v0.78.0) | `@agentclientprotocol/codex-acp`(app-server 上に実装、OpenAI 非公式、v1.12.0) |
| B. Agent 固有 API | Claude Agent SDK(公式、Node 必須、`canUseTool` で permission) | `codex app-server`(公式、JSON-RPC、承認 2 系統) |
| C. CLI 直叩き | `claude -p --input-format stream-json`(stdin 仕様・permission control が未文書化) | `codex exec --json`(承認が非対話、diff 本文なし) |

調査の詳細は `docs/research.md` §4。

## 決定

**MVP は A(ACP)を採用する。** 内部イベントモデルも ACP v1 の語彙(`agent_message_chunk` / `agent_thought_chunk` / `tool_call` / `tool_call_update` / `plan` / `request_permission`)に合わせる。
ただし `AgentBackend` interface を挟み、将来 B を同じ interface の別実装として追加できるようにする。

## 理由

1. **MVP の要件を全て満たす**。permission(options 付き)、tool call、`content.diff`、terminal、session resume、cwd、modes が ACP v1 で揃う。Agent SDK 直結では diff を Edit ツール入力から自前解釈する必要があり、`codex exec` では承認ができない
2. **ノベル UI の描画プリミティブと 1 対 1 で対応する**。台詞 = message chunk、思考 = thought chunk、演出 = tool_call、選択肢 = request_permission、クエストログ = plan。Agent ごとに写像を書く必要がない
3. **Agent 差し替えが設定 1 行**。Claude Code / Codex / Gemini CLI がすべて ACP Agent として存在し、ACP UI は 11 Agent を同じコードで扱っている。「Agent ごとにキャラクターを割り当てる」将来像に直結する
4. **実装コストが最小**。公式 TypeScript SDK(`@agentclientprotocol/sdk`、Web Streams 対応)があり、Windows のプロセス起動は ACP UI(MIT)などに先例がある
5. **エコシステムが安定している**。Zed と JetBrains の共同ガバナンス、protocolVersion 1 は安定版、SDK 1.0 リリース済み、Registry に約 50 Agent

## 反対意見と扱い

- **アダプタが非公式で破壊的変更がある**(claude-agent-acp は 2026-09 だけで 5 リリース、v0.77.0 に BREAKING)。Anthropic は ACP 対応要望を not planned でクローズしている
  → `npx <pkg>@<version>` でバージョン固定し、手動でアップグレードする。UI は `AgentEvent` しか見ないので、アダプタ差し替えの影響は `AcpBackend` に閉じる
- **Agent 固有機能が ACP に圧縮される**(Claude の `updatedInput` / hooks、Codex の execpolicy amendment、AskUserQuestion のプレビューなど)
  → MVP では不要。必要になった時点で `ClaudeSdkBackend` / `CodexAppServerBackend` を追加する。内部イベントは ACP 語彙のままにして、固有バックエンドがそこへ写像する
- **B を先に作る方が規約変更に強い**という意見(Anthropic / OpenAI の公式経路であり、認証の扱いが公式に文書化されている)
  → 認証は A でも Agent 自身のフロー(`claude auth login`)に委ねており、アプリがトークンに触れない点は同じ。規約リスクは経路ではなく「サブスクを第三者クライアントで使うこと」自体にあり、A/B で差はない
- **ACP v2 が draft 中**
  → v1 で実装し、v2 は `AcpBackend` 内で吸収する

## 結果

- `AgentBackend` interface + `AcpBackend` 実装(`docs/architecture.md` §7)
- Agent の起動コマンドは設定ファイルに置き、Codex 等への切り替えは設定変更だけで済ませる
- 内部イベントは ACP v1 語彙
- 将来の固有バックエンドは同じ interface で追加。UI に手を入れない
