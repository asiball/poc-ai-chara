# アーキテクチャ

対象: MVP(自分用、1 Agent、Claude Code、ACP / stdio)。将来の拡張(Codex 切替、デスクトップ化、TTS など)を塞がない範囲で最小に保つ。
判断の記録は `docs/adr/` を参照。

---

## 1. 全体像

```
+---------------------------------------------------------------+
|  ブラウザ(Vite + React + TypeScript)                          |
|                                                               |
|   Novel View                 Log View            Settings     |
|   立ち絵 / 名前欄 /           Markdown 全文 /      cwd /        |
|   メッセージウィンドウ /       diff / tool call /   キャラ /      |
|   選択肢 / テキスト送り        terminal / エラー    口調          |
|        ^                          ^                            |
|        |  Character State         |  Session Store             |
|        +------------+-------------+                            |
|                     |  AgentEvent(ACP 準拠の内部イベント)       |
+---------------------|-----------------------------------------+
                      |  WebSocket(localhost)
+---------------------v-----------------------------------------+
|  Node ホスト(TypeScript)                                       |
|   AgentBackend interface                                      |
|   AcpBackend: @agentclientprotocol/sdk ClientSideConnection   |
|   ProcessHost: 子プロセス spawn / stdio / kill                 |
+---------------------|-----------------------------------------+
                      |  stdio(JSON-RPC 2.0、改行区切り)
                      v
     npx @agentclientprotocol/claude-agent-acp@<固定版>
                      |
                      v  Claude Agent SDK → 無改変の claude バイナリ
     (将来) codex-acp / gemini --acp
```

原則:
- **Agent 本体は再実装しない**。別プロセスで動かし、ACP で会話する
- **UI は Agent の種類を知らない**。UI が扱うのは ACP 準拠の内部イベント(`AgentEvent`)だけ
- **プロトコル処理は Node 側の TypeScript**。ブラウザは WebSocket で `AgentEvent` を受け取り、prompt / permission 応答を送るだけ
- **デスクトップ化(Tauri / Electron)は後回し**。包むときは Node ホストをそのまま同梱するか、Electron の main に移す。UI と接続コードは変えない
- **Novel View と Log View は同じ Session Store を読む別ビュー**。どちらか一方にしか無い情報を作らない

---

## 2. ACP と UI の対応

### 2.1 採用理由(要約。詳細は ADR-0001)

- `session/update` の種別と `session/request_permission` が、ノベル UI の描画要素(台詞 / 思考 / 演出 / 選択肢)に 1 対 1 で対応する
- Claude Code、Codex、Gemini CLI が ACP Agent として提供されており、切り替えが設定 1 行
- `tool_call.content` に diff が乗るので、Agent 固有のツール入力を解釈しなくてよい

### 2.2 対応表

| ACP | 内部イベント | Novel View | Log View |
|---|---|---|---|
| `agent_message_chunk` | `message.delta` | 台詞(段落単位でページ送り、タイプライター) | Markdown 全文 |
| `agent_thought_chunk` | `thought.delta` | 表情 `thinking`。台詞には出さない | 折りたたみ「考えていること」 |
| `tool_call`(pending / in_progress) | `tool.start` | 表情 `thinking`、状態インジケータに kind と title | tool call カード |
| `tool_call_update`(completed / failed) | `tool.update` | failed なら `confused`(素材が無ければ `normal`) | カード更新、`content.diff` は diff 表示、terminal は出力表示 |
| `plan` | `plan.update` | 出さない | TODO リスト(MVP では保存のみ) |
| `session/request_permission` | `permission.request` | **選択肢**(options をそのまま)+「変更内容を見る」 | 対象 tool call と diff |
| `session/prompt` 応答 `stopReason` | `turn.end` | `end_turn` → `happy`、`cancelled` → `normal`、`refusal` → `confused` | 完了サマリ |
| `available_commands_update` | `commands.update` | 出さない(MVP) | 同左 |
| `usage_update` | `usage.update` | 出さない | 控えめに表示 |
| プロセス終了 / JSON-RPC エラー | `session.error` | `confused` + 短い定型台詞 | 生のエラーと stderr |

### 2.3 claude-agent-acp 固有の入口(ソース確認済み、v0.78 時点)

- `session/new` の `_meta.systemPrompt`: 文字列なら system prompt を置換、オブジェクト(`{ append: "..." }` 等)なら `claude_code` プリセットに追記。**口調は append のみ使う**(置換はコーディング指示を失う)
- `session/new` の `_meta.claudeCode.options`: Agent SDK のオプション(`permissionMode`、`resume` など)をそのまま渡せる
- `settingSources` は `["user", "project", "local"]` 固定。つまり CLAUDE.md、`.claude/settings*.json`、output style、hooks、`.mcp.json` はターミナルの `claude` と同じように読まれる
- 環境変数 `CLAUDE_CODE_EXECUTABLE` で `claude` バイナリのパスを上書きできる
- `login` / `logout` / `output-style:new` などターミナル専用コマンドは available_commands から除外される

---

## 3. プロセス構成

### 3.1 起動シーケンス

```
1. node host 起動(cwd、Agent 設定、キャラ設定を読む)
2. ProcessHost が Agent を spawn
     Windows: spawn("npx", [...], { shell: true })   ← npx.cmd のため shell 必須
              または npx の実体パスを解決して node で直起動
     Unix:    spawn("npx", [...])
   stdout → 改行単位で JSON-RPC、stderr → ログ
3. AcpBackend が initialize(protocolVersion 1, clientCapabilities: fs/terminal を無効, clientInfo)
4. 認証が必要なら authMethods をブラウザに表示
     type "terminal": 「別のターミナルで `claude auth login --claudeai` を実行してから再接続」と案内
     (アプリはトークンに触れない。無改変バイナリ自身のログインに委ねる)
5. session/new({ cwd, mcpServers: [], _meta?: { systemPrompt: { append } } }) → sessionId
6. ブラウザに「接続済み」。表情 normal
```

### 3.2 既知の注意点

| 課題 | 対処 |
|---|---|
| Windows で `npx` は `npx.cmd`。Node の `spawn` は `shell: true` 無しだと EINVAL(2024-04 のセキュリティ修正以降) | `shell: true`、または実体パスを解決して直起動 |
| `shell: true` 経由だと kill が子に届かないことがある | `taskkill /T /F /PID`、または直起動 |
| 改行で終わらない出力 | ACP は NDJSON なので問題なし |
| Agent が Node ≥22 を要求 | 自分の環境で満たす。設定で node / npx のパスを上書き可能にしておく |

### 3.3 セッション

- 1 プロセス = 1 ACP セッション(MVP)
- cwd を変えたら `session/new` を作り直す
- 終了時は `session/cancel` → プロセス kill
- 将来: Agent ごとに ProcessHost を持ち、`sessionId` でルーティング。キャラは Agent の設定に紐付ける

---

## 4. 状態管理

### 4.1 Session Store(トランスクリプト)

ACP イベントを時系列に正規化した配列。Novel / Log 両ビューの唯一の情報源。

```ts
type TranscriptItem =
  | { kind: "user_message"; id; text; at }
  | { kind: "agent_message"; id; markdown; at; segments: Segment[] }   // ストリーミングで追記
  | { kind: "agent_thought"; id; text; at }
  | { kind: "tool_call"; id; toolCallId; title; toolKind; status; locations; diffs; terminalOutput; rawInput; rawOutput }
  | { kind: "plan"; entries }
  | { kind: "permission"; id; toolCallId; options; outcome? }
  | { kind: "turn_end"; stopReason; summary }
  | { kind: "error"; message; detail };
```

### 4.2 Agent State(表情の入力)

Session Store から導出する。複数条件が同時に成り立つときは優先度の高い方。

```
priority  state               条件
  6       error               直近の turn が refusal / プロセス異常終了 / JSON-RPC エラー
  5       waiting_permission  未解決の permission.request がある
  4       tool_run            pending / in_progress の tool_call がある
  3       thinking            prompt 送信済みで message.delta 未着、または thought.delta 受信中
  2       speaking            message.delta 受信中
  1       done                直近の turn_end が end_turn(ホールド時間内)
  0       idle                それ以外
```

- `done` は数秒後に `idle` に落ちる
- `thinking` / `tool_run` には最低表示時間(例 600ms)を設けてチラつきを防ぐ
- 語彙は文字列 union にして後から足せるようにする

### 4.3 台詞と本文の分離(Segment)

`agent_message.markdown` をブロック単位でパースし、表示先を付ける。LLM は使わない。

| ブロック | Segment | Novel View での扱い |
|---|---|---|
| 段落、見出し、短い箇条書き(3 項目以内) | `speech` | 台詞として 1 ページ |
| コードブロック | `code` | 「[コードを見る]」リンクを挿入。本体は Log View |
| 表、長い箇条書き、diff | `detail` | 同上 |

ストリーミング中はブロック境界が確定するまで `speech` として扱い、確定時に再分類する。

### 4.4 テキスト送り

- ストリーミング中: 現在ページをタイプライター表示し、ページ末尾で自動で次へ
- ターン終了後: 最後のページで ▼ 点滅。クリック / Enter / Space で閉じる
- 途中クリック: タイプライターをスキップして即時表示
- 「オート」「スキップ」トグル。**既定はオート**(開発ツールなので待たせない)
- Log View は常に全文即時表示

---

## 5. permission

```
Agent → session/request_permission { toolCall, options[] }
  → Session Store に未解決の permission item、Agent State = waiting_permission
  → Novel View: 選択肢(options[].name をそのまま)+「変更内容を見る」
  → Log View: 同じ選択肢ボタン + diff
  → ユーザー選択 → { outcome: { outcome: "selected", optionId } }
  → Esc → { outcome: "cancelled" }
```

- `allow_always` / `reject_always` の記憶は Agent 側(ACP の意味論)。UI は記憶しない
- 選択肢の文言は ACP のまま。キャラの口調で言い換えない(意味がずれると承認事故になる)
- 台詞側には toolCall.title を 1 行。diff があれば「+12 -3 src/auth.ts」程度の要約
- UI 側で勝手に allow を返す自動承認モードは作らない。必要なら Agent 側の mode(`session/set_mode`)や `permissionMode` に委ねる

---

## 6. セキュリティ(自分用でも守るもの)

| 項目 | 方針 |
|---|---|
| 資格情報 | アプリは API キー / OAuth トークンを保存・仲介しない。認証は Agent 自身のフローに委ねる |
| API キーを使う場合 | 環境変数で Agent プロセスに渡す。設定ファイルに書くなら平文である点を自覚する |
| cwd | 明示的に指定したディレクトリのみ |
| ACP `fs/*` / `terminal/*` | MVP では clientCapabilities で無効にし、Agent に任せる(各アダプタが capability 無効時に自前実行へフォールバックするかは Step 0 で確認) |
| WebSocket | localhost のみで listen。外部公開しない |
| Markdown | `rehype-sanitize` 等で sanitize |

---

## 7. Agent abstraction

```ts
interface AgentBackend {
  readonly info: { id: string; displayName: string };
  connect(opts: { cwd: string; persona?: string }): Promise<void>;   // spawn + initialize + authenticate + session/new
  prompt(text: string): Promise<{ stopReason: StopReason }>;
  cancel(): Promise<void>;
  respondPermission(requestId: string, optionId: string | null): void;
  dispose(): Promise<void>;
  onEvent(handler: (e: AgentEvent) => void): () => void;
}
```

- MVP の実装は `AcpBackend` の 1 つ。Agent の違いは設定ファイルの起動コマンドと env だけ
- ACP に乗らない機能が必要になったら、同じ interface で `ClaudeSdkBackend` 等を足す。内部イベントは ACP 語彙のまま

```json
{
  "agent": {
    "command": "npx",
    "args": ["@agentclientprotocol/claude-agent-acp@0.78.0"],
    "env": {}
  },
  "cwd": "C:/work/myproject",
  "character": "./characters/mio",
  "persona": null
}
```

`persona` に文字列を入れると `_meta.systemPrompt.append` で渡す(output style で足りるなら null のまま)。

---

## 8. キャラクター表情とアニメーション

### 8.1 キャラクターフォルダ

素材はリポジトリに含めない。ローカルのフォルダを指す。

```
characters/mio/
  character.json
  normal.png
  thinking.png
  happy.png
  eyes-closed.png    (任意: 瞬き)
  mouth-open.png     (任意: 口パク)
```

```json
{
  "name": "ミオ",
  "expressions": {
    "normal":   "normal.png",
    "thinking": "thinking.png",
    "happy":    "happy.png"
  },
  "overlays": {
    "blink": "eyes-closed.png",
    "mouth": "mouth-open.png"
  },
  "stateMap": {
    "idle":               "normal",
    "thinking":           "thinking",
    "tool_run":           "thinking",
    "speaking":           "normal",
    "waiting_permission": "normal",
    "done":               "happy",
    "error":              "normal"
  },
  "fallback": "normal"
}
```

- `stateMap` は Agent State → 表情名。無い表情は `fallback`
- `stateMap` 省略時はアプリ既定を使う(画像だけ置けばよい)
- `overlays` は任意。無ければアニメーションしない

### 8.2 アニメーション(PNG + CSS で完結)

| 動き | 実装 |
|---|---|
| 表情切替 | 2 枚の `<img>` を重ねてクロスフェード(100〜200ms)。空白フレームを出さない |
| 瞬き | `blink` を 3〜6 秒のランダム間隔で 100ms 重ねる |
| 口パク | タイプライターが文字を出している間だけ `mouth` を 120ms 前後で交互に重ねる。差分は全身画像でも口周りだけの部分画像でもよい(位置合わせは同サイズ前提が楽) |
| 呼吸 | CSS `transform: translateY` の 2〜3px ループ(4 秒) |
| 登場 / 退場 | フェード + 数 px のスライド |

Live2D が要るのは視線追従や滑らかな首振りだけで、目的に入っていない。

### 8.3 表情の決定

- 入力は Agent State のみ。LLM による感情判定はしない
- 将来、テキスト感情推定や Agent 自身の指定を重ねたくなったら `AgentEvent` に `emotion.hint` を足し、`stateMap` より優先するかをそのとき決める

### 8.4 口調(ロールプレイ)

「一人称・語尾・呼び方」程度に留め、コーディングの指示は保つ。優先順位:

1. **Claude Code の output style**(Agent 側の設定、アプリのコード不要)
   - `.claude/output-styles/mio.md`
     ```markdown
     ---
     name: mio
     description: ミオの口調
     keep-coding-instructions: true
     ---
     一人称は「わたし」、語尾は「〜だよ」「〜だね」。ユーザーは「きみ」と呼ぶ。
     コードブロック、ファイルパス、コマンド、エラーメッセージは口調を変えずそのまま書く。
     ```
   - `.claude/settings.local.json` に `{ "outputStyle": "mio" }`
   - アダプタは settingSources を user / project / local で読むので ACP 経由でも効く見込み(**Step 0 で確認**)
   - 注意: ターミナルの `claude` にも効く。分けたいなら user ではなく project レベルに置き、プロジェクトごとに切り替える
2. **`_meta.systemPrompt.append`**(アプリ側から追記)
   - output style が効かない場合、または Agent 側の設定を触りたくない場合
   - 置換(文字列指定)は使わない
3. **Codex の `AGENTS.md`**(Codex に切り替えたとき)

やらないこと: アプリが system prompt を丸ごと差し替える(AvatarCode 方式)。Agent の応答をアプリ側で LLM に書き換えさせる(捏造とコストの両面で不利)。

---

## 9. Novel View と Log View

| | Novel View | Log View |
|---|---|---|
| 目的 | 会話している感覚、状態の把握、承認 | コードレビュー、原因調査、正確な情報 |
| 表示 | 立ち絵、名前欄、台詞(speech)、選択肢、状態インジケータ | Markdown 全文、コード、diff、tool call、terminal、plan、エラー、stderr |
| テキスト送り | あり | なし |
| レイアウト | 常に表示 | 右側パネル(開閉)。台詞内の「[コードを見る]」でジャンプ |
| 優先 | 演出 | **実用**。両立しなければ Log View を正とし、Novel View は要約 |

ノベルゲームの「バックログ」= Log View の全文トランスクリプト。

---

## 10. ディレクトリ構成(案)

```
poc-ai-chara/
  docs/
  scripts/
    acp-smoke.ts            # Step 0: 接続確認(UI なし)
  host/                     # Node ホスト
    agent/
      backend.ts            # AgentBackend interface, AgentEvent
      acp-backend.ts        # @agentclientprotocol/sdk
      process-host.ts       # spawn / stdio / kill
    server.ts               # WebSocket(localhost)
    config.ts
  web/                      # ブラウザ UI(Vite + React)
    state/
      session-store.ts
      agent-state.ts
      segments.ts           # Markdown → speech / code / detail
    character/
      pack.ts               # character.json
      sprite.tsx            # 表情クロスフェード、瞬き、口パク
    ui/
      novel/                # NameBox, MessageWindow, Choices, Typewriter
      log/                  # Transcript, ToolCallCard, DiffView
  characters/               # 自分の素材(git 管理外)
```
