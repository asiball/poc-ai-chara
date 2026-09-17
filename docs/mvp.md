# MVP

目標: **「表情差分 3 枚で Claude Code と会話できる」**。自分用なので、動くことと確かめたいことを優先し、配布や見栄えは後回しにする。

---

## 1. 最初に確かめること

UI を作る前に、いちばん不確かな点を潰す。

| 確認項目 | 方法 | 結果の扱い |
|---|---|---|
| 自分の環境で、サブスクログインのまま claude-agent-acp が ACP で応答するか | Step 0 のスクリプト | NG なら API キーで再試行。両方 NG なら接続方式を再検討 |
| permission 要求(`session/request_permission`)が届き、選択結果で処理が続くか | 同上(ファイル編集を頼む) | 届かない場合は `_meta.claudeCode.options.permissionMode` の既定を確認 |
| `tool_call` の `content` に diff が乗るか | 同上 | 乗らなければ Edit の rawInput から自前で組む |
| output style(`.claude/settings.local.json` の `outputStyle`)が ACP 経由でも効くか | 口調ファイルを置いて Step 0 を再実行 | 効かなければ `_meta.systemPrompt.append` を使う |
| kill で `node` / `claude` の子プロセスが残らないか(Windows) | タスクマネージャ | 残るなら `taskkill /T` |

---

## 2. スコープ

### 2.1 作るもの

| # | 機能 |
|---|---|
| M1 | Node プロセスが `npx @agentclientprotocol/claude-agent-acp@<固定版>` を stdio で起動し、`initialize` → `session/new` まで通す |
| M2 | 未認証なら authMethods を表示し、`terminal` 型のログインコマンドを案内する(アプリはトークンに触れない) |
| M3 | 作業ディレクトリの指定(起動引数または設定ファイル) |
| M4 | ブラウザ UI から `session/prompt`(text のみ)。送信中は入力を無効化。Esc で `session/cancel` |
| M5 | `agent_message_chunk` を段落単位でページ分割し、タイプライター表示。オート送り既定、クリックでスキップ、ターン終了で ▼ |
| M6 | コードブロック・表・長いリストは台詞から除いて「[コードを見る]」リンクにし、ログパネルに全文 Markdown を出す |
| M7 | Agent の状態(idle / thinking / speaking / tool_run / waiting_permission / done / error)→ `normal` / `thinking` / `happy` の 3 枚。無い表情は `normal` にフォールバック |
| M8 | `session/request_permission` を選択肢として表示し、選択結果を返す。数字キー / Enter / Esc |
| M9 | ログパネルに tool call(kind / title / status)、diff、terminal 出力、エラー |
| M10 | 瞬き(目閉じ差分)と口パク(口開き差分、タイプライター中のみ)。素材が無ければ何もしない |
| M11 | 口調: プロジェクトの output style で設定する手順を README に書く。効かない場合は設定ファイルの `persona` を `_meta.systemPrompt.append` で渡す |
| M12 | 名前欄横の状態インジケータ(「実行中 (Bash)」「承認待ち」など) |

### 2.2 作らないもの

- Live2D / VRM、TTS、音声入力
- LLM による感情判定、テキスト感情推定
- 複数 Agent 同時実行、Agent ごとのキャラ割り当て(設定上は可能でも検証しない)
- Codex / Gemini CLI の動作保証
- セッション履歴の一覧・再開 UI
- ACP `fs/*` / `terminal/*` のクライアント側実装(capabilities で無効にし、Agent に任せる)
- 画像添付、@-mention、slash command 補完、plan の描画
- デスクトップアプリ化(Tauri / Electron)、インストーラ、配布
- Windows 以外の動作保証

---

## 3. 実装順

### Step 0: 接続確認スクリプト(UI なし)

- `scripts/acp-smoke.ts` のような 1 ファイル。`@agentclientprotocol/sdk` の `ClientSideConnection` で `npx @agentclientprotocol/claude-agent-acp@X.Y.Z` を子プロセス起動
- `initialize` → 必要なら authMethods を表示して終了 → `session/new({ cwd })` → `session/prompt("このリポジトリの構成を説明して")` → `session/update` を種類ごとに整形してコンソールに出す
- `session/request_permission` が来たら標準入力で番号を選ばせて返す
- 「ファイルを 1 つ作って」と頼み、permission と `tool_call.content.diff` が流れることを見る
- Windows の `npx.cmd` は Node の `spawn` で `shell: true` が必要(CVE-2024-27980 以降)。または `npx` の実体パスを解決して `node` で直起動
- **§1 の確認項目をここで全部潰す。** 演出には着手しない

### Step 1: 接続層とセッション状態

- `AgentBackend` interface と `AcpBackend`(Step 0 のコードを整理)
- Node 側で ACP クライアントを動かし、ブラウザとは WebSocket で `AgentEvent` をやり取りする
- トランスクリプト(Session Store)と、そこから導出する Agent State
- テスト: ACP イベント列 → Session Store / Agent State の期待値

### Step 2: ログパネル(実用側を先に)

- 全文 Markdown(sanitize)、tool call カード、diff、terminal 出力、エラー
- permission の選択肢(まずは普通のボタン)
- ここで「地味だが使えるクライアント」にする

### Step 3: ノベル UI

- 立ち絵、名前欄、メッセージウィンドウ、選択肢、状態インジケータ
- Markdown → 台詞 / コード / 詳細の振り分け、ページ分割、タイプライター、オート送り、スキップ

### Step 4: 表情とアニメーション

- `character.json` 読込、Agent State → 表情、クロスフェード、`done` のホールドと `idle` 復帰
- 瞬き、口パク、待機時の軽い揺れ

### Step 5: 口調

- output style ファイルの雛形(`keep-coding-instructions: true`、一人称・語尾・呼び方だけ)
- 効かない場合のフォールバック(`_meta.systemPrompt.append`)

---

## 4. 受け入れ条件

### 接続

- [ ] Step 0 のスクリプトが、サブスクログイン済みの環境で `end_turn` まで到達する
- [ ] 同じスクリプトが `ANTHROPIC_API_KEY` 環境変数でも到達する
- [ ] 未認証の環境で authMethods が表示され、案内どおりログインすると次回は通る。ログ・設定ファイルにトークン文字列が残らない
- [ ] 終了後に `node` / `claude` の子プロセスが残らない
- [ ] Agent プロセスが落ちても UI が固まらず、エラーが表示され、再接続できる

### 会話

- [ ] 応答が段落ごとにタイプライター表示され、コードブロックは「[コードを見る]」に置き換わり、ログパネルで全文が読める
- [ ] 応答中のクリックでスキップ、ターン終了で ▼
- [ ] Esc で `session/cancel` が送られ、状態が `idle` に戻る
- [ ] Markdown 内の HTML / スクリプトが実行されない

### 表情・アニメーション

- [ ] 送信直後と tool call 中は `thinking.png`、`end_turn` で `happy.png`、数秒後に `normal.png`
- [ ] `character.json` から `happy` を消してもエラーにならず `normal.png` が出る
- [ ] 目閉じ差分があれば数秒おきに瞬きし、口開き差分があればタイプライター中だけ口が動く
- [ ] 切替時に空白フレームが出ない

### permission

- [ ] ファイル編集の要求が選択肢として出て、名前欄横が「承認待ち」になる
- [ ] 「変更内容を見る」でログパネルに diff が出る
- [ ] 数字キーと Enter で選べ、選択後に処理が続く。Esc で `cancelled` が返る
- [ ] 承認待ちの間、入力欄が無効化される

### 口調

- [ ] output style を置くと、ターミナルの `claude` と ACP 経由の両方で口調が変わる(変わらなければその旨をドキュメントに記録し、フォールバックに切り替える)
- [ ] 口調を付けても、ファイル編集・テスト実行などの動作が変わらない

---

## 5. MVP の後で考えること

- Codex(`codex-acp`)で同じ UI が動くか
- `confused` / `surprised` の追加
- Tauri か Electron で包む(UI と接続コードはそのまま)
- `session/load` によるセッション再開、plan の表示
