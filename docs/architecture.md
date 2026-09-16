# アーキテクチャ

対象: MVP(1 Agent、Claude Code、ACP / stdio)。将来拡張(複数 Agent、Codex / Gemini、TTS など)を妨げない範囲で最小に保つ。
判断の記録は `docs/adr/` を参照。

---

## 1. 全体像: Agent / UI の分離

```
+---------------------------------------------------------------------------+
|  Desktop App (Tauri 2)                                                    |
|                                                                           |
|  +---------------------------------------------------------------------+  |
|  |  WebView (TypeScript / React)                                       |  |
|  |                                                                     |  |
|  |   Novel Layer            Developer Layer         Settings           |  |
|  |   立ち絵 / 名前欄 /       Markdown 全文 / diff /   Agent 設定 / cwd / |  |
|  |   メッセージウィンドウ /   tool call / terminal /   キャラクターパック  |  |
|  |   選択肢 / テキスト送り    plan / エラー / ログ                       |  |
|  |        ^                        ^                                   |  |
|  |        |   Character State      |   Session Store(トランスクリプト)  |  |
|  |        |   (表情マッピング)       |                                   |  |
|  |        +----------+-------------+                                   |  |
|  |                   |                                                 |  |
|  |          Agent Session Model(ACP 準拠の内部イベント)                 |  |
|  |                   ^                                                 |  |
|  |          AgentBackend interface                                     |  |
|  |                   |                                                 |  |
|  |          AcpBackend(@agentclientprotocol/sdk ClientSideConnection)  |  |
|  |                   |  Tauri events / invoke(行単位の NDJSON)         |  |
|  +-------------------|-------------------------------------------------+  |
|                      |                                                    |
|  +-------------------v-------------------------------------------------+  |
|  |  Rust core (src-tauri)                                              |  |
|  |   ProcessHost: 子プロセス spawn / stdin 書込 / stdout 行読取 / kill   |  |
|  |   Windows: cmd /C + CREATE_NO_WINDOW、Unix: login shell で PATH 継承 |  |
|  |   fs コマンド(キャラクターパック読込、cwd ピッカー)                    |  |
|  +-------------------|-------------------------------------------------+  |
+----------------------|----------------------------------------------------+
                       | stdio(JSON-RPC 2.0、改行区切り)
                       v
        npx @agentclientprotocol/claude-agent-acp@<pinned>
                       |
                       v  Claude Agent SDK → 無改変の claude バイナリ
        (将来) codex-acp / gemini --acp / その他 ACP Agent
```

原則:
- **Agent 本体は再実装しない**。Agent は別プロセスで動き、ACP で会話する
- **UI は Agent の種類を知らない**。UI が扱うのは ACP 準拠の内部イベント(`AgentEvent`)だけ
- **プロトコル処理は TypeScript 側に置く**。Rust 側は「プロセスを起動し、行を流す」だけにする。将来 Electron 等へ移る場合も TS 側は無傷で済む
- **Novel Layer と Developer Layer は同じ Session Model を読む別ビュー**。どちらか一方にしか無い情報を作らない

---

## 2. 通信方式: ACP

### 2.1 採用理由(要約。詳細は ADR-0001)

- ACP の `session/update` 種別(`agent_message_chunk` / `agent_thought_chunk` / `tool_call` / `tool_call_update` / `plan`)と `session/request_permission` が、ノベル UI の描画プリミティブ(台詞 / 思考 / 演出 / クエストログ / 選択肢)に 1 対 1 で対応する
- Claude Code、Codex、Gemini CLI がすべて ACP Agent として提供されており、Agent 差し替えが設定ファイルの 1 行で済む
- `tool_call.content` に `diff {path, oldText, newText}` が乗るので、diff 表示のために Agent 固有のツール入力を解釈しなくてよい
- Windows での起動問題は acp-ui(MIT)が解決済みで、そのまま流用できる

### 2.2 リスクと対策

| リスク | 対策 |
|---|---|
| ACP アダプタ(claude-agent-acp / codex-acp)は Anthropic / OpenAI 非公式で、破壊的変更がある | `npx <pkg>@<version>` でバージョンを固定。アップグレードは手動 |
| Agent 固有機能(Claude の hooks 設定、Codex の execpolicy amendment 等)が ACP に乗らない | MVP では不要。必要になったら `AgentBackend` の別実装(Agent SDK 直結 / app-server 直結)を追加する。UI 側は変更しない |
| ACP v2 への移行 | 内部イベントは ACP v1 の語彙に依存するが、`AcpBackend` 内で吸収する。UI は `AgentEvent` しか見ない |
| ACP アダプタが Node ≥22 を要求する | ユーザー環境の Node を使う(設定で node / npx のパスを指定可能)。Node 同梱(sidecar)は MVP では行わない |

### 2.3 ACP メッセージと UI の対応

| ACP | 内部イベント | Novel Layer | Developer Layer |
|---|---|---|---|
| `agent_message_chunk` | `message.delta` | 台詞(段落単位でページ送り、タイプライター) | Markdown 全文 |
| `agent_thought_chunk` | `thought.delta` | 表情 `thinking`、台詞ウィンドウには出さない | 折りたたみ「考えていること」 |
| `tool_call`(pending / in_progress) | `tool.start` | 表情 `thinking`、状態インジケータ(kind と title) | tool call カード(kind / title / locations / rawInput) |
| `tool_call_update`(completed / failed) | `tool.update` | failed なら表情 `confused`(MVP は normal にフォールバック) | カード更新、`content.diff` は diff ビュー、terminal は出力ビュー |
| `plan` | `plan.update` | 表示しない | クエストログ風の TODO リスト |
| `session/request_permission` | `permission.request` | **選択肢**(options をそのまま並べる)+「変更内容を見る」リンク | 対象 tool call の詳細と diff |
| `session/prompt` 応答 `stopReason` | `turn.end` | `end_turn` → 表情 `happy`、`cancelled` → `normal`、`refusal` → `confused` | 完了サマリ(変更ファイル数、tool call 数) |
| `available_commands_update` | `commands.update` | 入力欄の `/` 補完 | 同左 |
| `current_mode_update` / `config_option_update` | `config.update` | 名前欄横のモード表示 | 設定パネル |
| `usage_update` | `usage.update` | 表示しない(RyzaChat の教訓) | 控えめな使用量表示 |
| プロセス終了 / JSON-RPC エラー | `session.error` | 表情 `confused`(MVP は normal)+ 短い台詞「うまく繋がらなかったみたい」 | 生のエラーと stderr |

---

## 3. プロセス構成

### 3.1 起動シーケンス

```
1. アプリ起動
2. 設定読込: agents.json(コマンド・引数・env)、キャラクターパック、前回の cwd
3. ユーザーが cwd を選択(絶対パス。ACP は相対パス不可)
4. Rust ProcessHost が Agent プロセスを spawn
     Windows: cmd /C npx @agentclientprotocol/claude-agent-acp@X.Y.Z   (CREATE_NO_WINDOW)
     Unix:    $SHELL -l -c "npx @agentclientprotocol/claude-agent-acp@X.Y.Z"
   stdout → 行単位で Tauri event `agent:line`、stderr → `agent:stderr`、終了 → `agent:closed`
5. TS AcpBackend が initialize(protocolVersion 1, clientCapabilities, clientInfo)
6. 認証が必要なら authMethods を表示
     type "terminal": 別ターミナルで `claude auth login --claudeai` などを起動し、終了を待って再接続
     (アプリはトークンに触れない。無改変バイナリ自身のログインフローに委ねる)
7. session/new({ cwd, mcpServers: [] }) → sessionId
8. UI が「接続済み」表示。表情 normal
```

### 3.2 プロセス管理の注意点(調査で判明した既知課題)

| 課題 | 対処 |
|---|---|
| Windows で `npx` は `npx.cmd` で、Rust `Command` から直接実行できない | `cmd /C` 経由(acp-ui `agent.rs` と同じ) |
| `cmd /C` 経由だと kill が子プロセスに届かない(tauri #4949) | `taskkill /T /F /PID` で子ごと終了。または設定で node の実体パスを指定させ `node <script>` を直起動 |
| コンソールウィンドウが一瞬出る | `CREATE_NO_WINDOW (0x08000000)` |
| macOS / Linux の GUI アプリは PATH が最小で nvm / volta の node が見えない | login shell 経由、または `fix-path-env` |
| stdout が改行で終わらないと届かない(tauri-plugin-shell #1632) | ACP は改行区切り NDJSON なので問題なし。自前 Rust spawn なら `BufRead::lines` |
| 長命プロセス | tauri-plugin-shell の JS `Command.create` は capabilities で command を allowlist する必要があり、任意コマンドに向かない。**Rust 側の `#[tauri::command]` で spawn し event で流す**(acp-ui / opcode 方式) |

### 3.3 セッションのライフサイクル

- アプリの 1 ウィンドウ = 1 Agent プロセス = 1 ACP セッション(MVP)
- cwd を変更したら `session/new` を作り直す(プロセスは再利用可)
- ウィンドウを閉じたら `session/cancel` → プロセス kill
- 将来: 複数 Agent は `ProcessHost` を複数持ち、`sessionId` でルーティングする。UI のキャラクターは `agentId` に紐付ける

---

## 4. 状態管理

### 4.1 Session Store(トランスクリプト)

ACP イベントを時系列に正規化した配列。Novel / Developer 両レイヤの唯一の情報源。

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

Session Store から導出する(保存しない)。優先度付きで、複数条件が同時に成り立つときは高い方を採る。

```
priority  state               条件
  6       error               直近の turn が refusal / プロセス異常終了 / JSON-RPC エラー
  5       waiting_permission  未解決の permission.request がある
  4       tool_run            status が pending / in_progress の tool_call がある
  3       thinking            prompt 送信済みで、message.delta がまだ来ていない、または thought.delta 受信中
  2       speaking            message.delta 受信中
  1       done                直近の turn_end が end_turn(ホールド時間内)
  0       idle                それ以外
```

- `done` は一定時間(例 4 秒)後に `idle` に落ちる(Agent Avatar の `EXPRESSION_REVERT_MS` と同じ発想)
- `thinking` / `tool_run` は最低表示時間(例 600ms)を設け、チラつきを防ぐ
- 状態語彙は将来 `cancelled`、`subagent_run` などを追加できるよう文字列 union にする

### 4.3 台詞と本文の分離(Segment)

`agent_message.markdown` をブロック単位でパースし、各ブロックに表示先を付ける。LLM は使わない。

| ブロック | Segment 種別 | Novel Layer での扱い |
|---|---|---|
| 段落、見出し、短い箇条書き(3 項目以内) | `speech` | 台詞として 1 ページ。タイプライター表示 |
| コードブロック | `code` | 「[コードを見る]」リンクを台詞に挿入。本体は Developer Layer |
| 表、長い箇条書き、diff | `detail` | 同上 |
| 空行のみ | 無視 | – |

ストリーミング中はブロックの境界が確定するまで `speech` として扱い、確定時に再分類する(コードブロックの開始 ``` を検出した時点で `code` に切り替える)。

### 4.4 テキスト送りの挙動

- ストリーミング中: 現在のページをタイプライターで表示。ページ末尾に達したら **自動で次ページへ**(ユーザーが「待って」と思う操作は不要)
- ターン終了後: 最後のページで ▼ を点滅。クリック / Enter / Space で閉じる
- 途中クリック: タイプライターをスキップして即時表示(ノベルゲームの標準挙動)
- 「オート」「スキップ」トグルを設定に置く。**既定はオート**(開発ツールなので、待たせない)
- Developer Layer は常に全文を即時表示。テキスト送りは Novel Layer 限定

---

## 5. permission

### 5.1 フロー

```
Agent → session/request_permission { sessionId, toolCall, options[] }
  ↓
Session Store に permission item を追加(未解決)
  ↓ Agent State = waiting_permission
Novel Layer: 台詞ウィンドウに選択肢を表示(options の name をそのまま)
              「変更内容を見る」→ Developer Layer に該当 tool call と diff を開く
Developer Layer: tool call カードに同じ選択肢ボタン
  ↓ ユーザーが選択
AcpBackend → response { outcome: { outcome: "selected", optionId } }
  ↓
permission item を解決済みに。Agent State は次の状態へ
```

- `allow_always` / `reject_always` は Agent 側が記憶する(ACP の意味論)。UI は記憶しない
- ユーザーがキャンセル(Esc)したら `{ outcome: "cancelled" }`
- ターン全体のキャンセルは `session/cancel`。未解決の permission は Agent 側が閉じる
- キーボード: 1 / 2 / 3 で選択肢、Enter で確定、Esc でキャンセル
- **permission を出さずに自動許可するモードは MVP に置かない**。将来置く場合も Agent 側の mode(`session/set_mode`)に委ね、UI 側で勝手に allow を返さない

### 5.2 permission の見せ方の原則

- 選択肢の文言は ACP の `options[].name` をそのまま使う(アダプタが「Allow」「Always allow」等を返す)。キャラクターの口調で言い換えない(意味がずれる)
- 台詞側には「何をしたいか」(toolCall.title)を 1 行で出す。詳細は必ず Developer Layer で確認できる
- diff がある場合は台詞ウィンドウ内に「+12 -3 / src/auth.ts」程度の要約を出す

---

## 6. セキュリティ

| 項目 | 方針 |
|---|---|
| 資格情報 | アプリは API キー / OAuth トークンを保存・仲介しない。認証は Agent(無改変の `claude` / `codex` バイナリ)自身のフローに委ねる。ACP `terminal` 型認証は別ターミナルを開いて終了を待つだけ |
| API キーを使う場合 | ユーザーが環境変数(`ANTHROPIC_API_KEY` 等)を Agent プロセスに渡す。`agents.json` の `env` に書く場合は平文保存になることを設定画面で明示する |
| cwd | ユーザーが明示的に選んだディレクトリのみ。ACP `fs/*` を実装する場合は cwd 配下(+ `additionalDirectories`)に限定し、パストラバーサルを拒否する(AvatarCode と同じ) |
| ACP `fs/read_text_file` / `fs/write_text_file` / `terminal/*` | **MVP では clientCapabilities で無効(false)にする**。Agent 側が自前で読み書き・実行する。将来、エディタ連携やターミナル表示が欲しくなったら有効化する(要検証: 各アダプタが capability 無効時に正しくフォールバックするか) |
| 子プロセスのコマンド | `agents.json` で任意コマンドを起動できるため、設定ファイルはユーザーのアプリデータ配下に置き、外部から流し込まれる経路(URL スキーム等)を作らない |
| WebView | 外部 URL を開かない。Markdown レンダリングは sanitize する(`rehype-sanitize` 等) |
| テレメトリ | 送らない |
| ネットワーク | アプリ自身は通信しない。通信するのは Agent プロセスのみ |

---

## 7. Agent abstraction

```ts
interface AgentBackend {
  readonly info: { id: string; displayName: string };
  connect(opts: { cwd: string }): Promise<void>;          // spawn + initialize + authenticate + session/new
  prompt(text: string): Promise<{ stopReason: StopReason }>;
  cancel(): Promise<void>;
  respondPermission(requestId: string, optionId: string | null): void;
  setMode?(modeId: string): Promise<void>;
  dispose(): Promise<void>;
  onEvent(handler: (e: AgentEvent) => void): () => void;   // §2.3 の内部イベント
}
```

- MVP の実装は `AcpBackend` の 1 つ。Agent の違いは `agents.json` の起動コマンドと env だけ
- 将来 ACP に乗らない機能が必要になった場合、`ClaudeSdkBackend`(Node sidecar + Agent SDK)や `CodexAppServerBackend` を同じ interface で追加する。内部イベントは ACP 語彙のままにし、各 Backend が自分のプロトコルからそこへ写像する
- `agents.json` は acp-ui と互換の形にしておく(既に 11 Agent 分の起動コマンドが揃っているため)

```json
{
  "agents": {
    "claude": {
      "command": "npx",
      "args": ["@agentclientprotocol/claude-agent-acp@0.78.0"],
      "env": {},
      "character": "mio"
    },
    "codex": {
      "command": "npx",
      "args": ["@agentclientprotocol/codex-acp@1.12.0"],
      "env": {},
      "character": "rin"
    }
  }
}
```

---

## 8. キャラクター表情管理

### 8.1 キャラクターパック

素材は同梱せず、ユーザーがフォルダを指定する(BYO)。

```
characters/mio/
  character.json
  normal.png
  thinking.png
  happy.png
```

```json
{
  "id": "mio",
  "name": "ミオ",
  "version": 1,
  "expressions": {
    "normal":   "normal.png",
    "thinking": "thinking.png",
    "happy":    "happy.png"
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

- `stateMap` は Agent State(§4.2)→ 表情名。**表情名が `expressions` に無い場合は `fallback` を使う**。これにより、将来 `confused` / `surprised` を追加したとき、素材が無いパックでも壊れない
- `stateMap` を省略した場合はアプリ既定のマッピングを使う(パック作者は画像だけ用意すればよい)
- 画像は PNG(透過)。サイズは任意で、アプリ側が高さ基準でフィットさせる
- 将来の拡張フィールド(MVP では読まない): `blink`(瞬き差分と間隔)、`voice`、`background`、`transitions`

### 8.2 表情の決定は UI 側で機械的に行う

- 入力は Agent State のみ。**LLM による感情判定は行わない**。テキスト感情推定や Agent 自身による表情指定は、将来 `AgentEvent` に `emotion.hint` を追加して重ね掛けする形で入れられる(stateMap より優先するかはそのとき決める)
- 表情の切替はクロスフェード(100〜200ms)。瞬きなどのアニメーションは MVP では入れない

### 8.3 台詞の一人称・口調

- MVP では Agent の出力をそのまま台詞にする。キャラクターの口調に書き換えない(捏造と能力劣化を避ける)
- 将来「口調」を付けたい場合は、Agent 側の設定(CLAUDE.md / AGENTS.md や `--append-system-prompt` 相当)に委ねる。アプリが system prompt を置換することはしない(AvatarCode の方式は採らない)

---

## 9. Novel UI と Developer UI の分離

| | Novel Layer | Developer Layer |
|---|---|---|
| 目的 | 会話している感覚、状態の把握、承認 | コードレビュー、原因調査、正確な情報 |
| 表示 | 立ち絵、名前欄、台詞(speech segment)、選択肢、状態インジケータ | Markdown 全文、コード、diff、tool call、terminal 出力、plan、エラー、stderr |
| テキスト送り | あり | なし(即時全文) |
| レイアウト | 常に表示 | 右サイドパネル(開閉)または全画面。台詞内の「[詳細を見る]」「[コードを見る]」で該当箇所へジャンプ |
| データ源 | Session Store | Session Store(同一) |
| 優先 | 演出 | **実用**。両立しない場合は Developer Layer を優先し、Novel Layer は要約に留める |

「バックログ」(ノベルゲームのログ画面)は Developer Layer の全文トランスクリプトそのものとする。ノベルゲームの文法に慣れたユーザーには「ログを開く = 詳細を見る」で自然に繋がる。

---

## 10. ディレクトリ構成(案)

```
poc-ai-chara/
  docs/
  src/                      # WebView (TypeScript / React)
    agent/
      backend.ts            # AgentBackend interface, AgentEvent
      acp-backend.ts        # ACP 実装(@agentclientprotocol/sdk)
      transport-tauri.ts    # Tauri event ↔ Web Streams
    state/
      session-store.ts      # トランスクリプト
      agent-state.ts        # 優先度付き状態導出
      segments.ts           # Markdown → speech / code / detail
    character/
      pack.ts               # character.json 読込・検証
      expression.ts         # state → expression
    ui/
      novel/                # Sprite, NameBox, MessageWindow, Choices, Typewriter
      dev/                  # Transcript, ToolCallCard, DiffView, PlanView, ErrorView
      settings/
  src-tauri/
    src/
      process_host.rs       # spawn / write / kill(acp-ui agent.rs を土台に)
      config.rs             # agents.json
      main.rs
  characters/               # サンプルパック置き場(素材は同梱しない。README のみ)
```
