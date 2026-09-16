# MVP

目標: **「表情差分 3 枚で Claude Code と会話できる」**を、Windows で動くデスクトップアプリとして成立させる。

---

## 1. スコープ

### 1.1 対象機能

| # | 機能 | 内容 |
|---|---|---|
| M1 | アプリ起動 | Tauri 2 アプリとして起動。Windows を第一対象、macOS は動作確認程度 |
| M2 | Agent 接続 | `npx @agentclientprotocol/claude-agent-acp@<pinned>` を stdio で起動し、ACP `initialize` → `session/new` まで完了する |
| M3 | 認証誘導 | 未認証なら ACP の authMethods を表示し、`terminal` 型は別ターミナルでログインコマンドを起動して終了を待つ。アプリはトークンに触れない |
| M4 | cwd 選択 | フォルダピッカーで作業ディレクトリを選ぶ。前回値を記憶 |
| M5 | メッセージ入力 | 入力欄から `session/prompt`(text のみ)。送信中は入力を無効化。Esc / ボタンで `session/cancel` |
| M6 | 台詞表示 | `agent_message_chunk` を段落単位にページ分割し、タイプライター表示。オート送り既定。クリックでスキップ。ターン終了で ▼ |
| M7 | 台詞 / 詳細の分離 | コードブロック・表・長いリストは台詞から除き「[コードを見る]」リンクにする。Developer Layer に全文 Markdown を表示 |
| M8 | 表情切替 | Agent State(idle / thinking / speaking / tool_run / waiting_permission / done / error)→ `normal` / `thinking` / `happy` の 3 枚。無い表情は `normal` にフォールバック |
| M9 | permission | `session/request_permission` を選択肢として表示。options をそのまま並べ、選択結果を返す。キーボード操作対応 |
| M10 | tool call 表示 | Developer Layer に tool call カード(kind / title / status / locations)。`content.diff` は diff ビュー、terminal 出力はテキスト表示 |
| M11 | エラー表示 | プロセス異常終了、JSON-RPC エラー、`refusal` を Developer Layer に生表示。Novel Layer には短い定型台詞 |
| M12 | キャラクターパック | フォルダを指定して `character.json` + PNG 3 枚を読む。サンプルパックは画像を含まない雛形のみ同梱 |
| M13 | 設定 | `agents.json`(起動コマンド・引数・env)、node / npx のパス上書き、キャラクターパックのパス、オート送り ON/OFF |
| M14 | 状態インジケータ | 名前欄横に「実行中 (Bash)」「承認待ち」「完了」など、Agent State と現在の tool title を表示 |

### 1.2 非対象(MVP から明示的に外す)

- Live2D / VRM / 瞬きなどのアニメーション
- TTS、音声入力
- LLM による感情判定、テキスト感情推定、Agent 自身による表情指定
- 複数 Agent の同時実行、Agent 間レビュー
- Codex / Gemini CLI の動作保証(`agents.json` に書けば起動はできるが、MVP では検証しない)
- セッション履歴の一覧・再開 UI(`session/load` は将来)
- ACP `fs/*` / `terminal/*` のクライアント側実装(capabilities で無効にする)
- 画像添付、@-mention、slash command 補完
- plan(`session/update: plan`)の描画(受信して保存はする)
- リモート Agent(WebSocket transport)
- キャラクターの口調変更、system prompt 注入
- 自動アップデート、コード署名、インストーラの配布
- macOS / Linux の動作保証

---

## 2. 実装順

リスクの高い部分(Agent プロセス起動と ACP 接続)を先に潰し、演出は最後に載せる。

### Step 0: 接続スパイク(捨てるコード、1〜2 日)

- Tauri 2 の最小アプリから Rust `process_host.rs` で `cmd /C npx @agentclientprotocol/claude-agent-acp@X` を起動し、stdout の行を WebView に流す
- TS 側で `@agentclientprotocol/sdk` の `ClientSideConnection` を Tauri event 経由の Web Streams に接続し、`initialize` → `session/new` → `session/prompt("hello")` → `session/update` をコンソールに出す
- 確認事項: Windows での起動、認証フロー(`terminal` 型)、kill で子プロセスが残らないか、`clientCapabilities` で fs / terminal を無効にしたときアダプタが Bash / Edit を自前実行するか
- **ここで詰まったら ADR-0002 の代替案(Electron)を再検討する**。演出には一切着手しない

### Step 1: 接続層と Session Store

- `AgentBackend` interface と `AcpBackend`
- `agents.json` 読込、cwd ピッカー、接続 / 切断
- Session Store(トランスクリプト正規化)、Agent State 導出
- 単体テスト: ACP イベント列 → Session Store / Agent State の期待値

### Step 2: Developer Layer(実用 UI を先に)

- 全文トランスクリプト(Markdown、sanitize)
- tool call カード、diff ビュー、terminal 出力、エラー表示
- permission ダイアログ(選択肢ボタン、キーボード)
- この時点で「地味だが使えるクライアント」として成立させる

### Step 3: Novel Layer

- 立ち絵、名前欄、メッセージウィンドウ
- Segment 分割(speech / code / detail)、ページ分割、タイプライター、オート送り、スキップ
- 選択肢 UI(permission)
- 状態インジケータ

### Step 4: キャラクターパックと表情

- `character.json` 読込・検証、フォールバック
- Agent State → 表情、クロスフェード、`done` のホールドと `idle` への復帰、最低表示時間

### Step 5: 設定と仕上げ

- 設定画面(agents.json 編集、パス上書き、パック選択、オート送り)
- Windows ビルド(NSIS)、README の手順を実機で検証

---

## 3. 受け入れ条件

### 接続

- [ ] Windows 11 のクリーン環境(Node 22 と `claude` インストール済み、`claude auth login` 済み)で、アプリ起動 → cwd 選択 → 「接続済み」表示まで 10 秒以内に到達する
- [ ] 未認証の環境で、アプリが authMethods を表示し、ログインコマンドを別ターミナルで起動し、完了後に再接続できる。アプリの設定ファイル・ログにトークン文字列が一切残らない
- [ ] `ANTHROPIC_API_KEY` を `agents.json` の env に設定した場合も接続できる
- [ ] アプリ終了後に `node` / `claude` の子プロセスが残らない(タスクマネージャで確認)
- [ ] Agent プロセスが異常終了した場合、UI がフリーズせずエラーを表示し、再接続ボタンが機能する

### 会話

- [ ] 「このリポジトリの構成を説明して」と送ると、Agent の応答が段落ごとにタイプライター表示され、コードブロックは「[コードを見る]」に置き換わり、Developer Layer で全文が読める
- [ ] 応答中にクリックするとタイプライターがスキップされ、ターン終了で ▼ が点滅する
- [ ] 送信中に Esc を押すと `session/cancel` が送られ、Agent State が `idle` に戻る
- [ ] Markdown に含まれる HTML / スクリプトが実行されない

### 表情

- [ ] 送信直後に `thinking.png`、tool call 実行中も `thinking.png`、`end_turn` で `happy.png`、約 4 秒後に `normal.png` に戻る
- [ ] `character.json` の `expressions` から `happy` を削除しても、アプリがエラーにならず `normal.png` が表示される
- [ ] 表情切替時に画像のチラつき(空白フレーム)が出ない

### permission

- [ ] Agent がファイル編集を要求すると、台詞ウィンドウに ACP の options(例: Allow / Always allow / Reject)が選択肢として表示され、名前欄横が「承認待ち」になる
- [ ] 「変更内容を見る」で Developer Layer に diff が表示される
- [ ] 数字キーと Enter で選択でき、選択後に Agent が処理を続行する
- [ ] Esc でキャンセルすると `outcome: cancelled` が返り、Agent がそれを拒否として扱う
- [ ] permission 待ちの間、ユーザー入力欄は無効化され、選択肢以外の操作で状態が壊れない

### Developer Layer

- [ ] tool call ごとに kind / title / status が表示され、completed / failed で見た目が変わる
- [ ] `content.diff` を含む tool call で、追加 / 削除行が色分けされた diff が表示される
- [ ] terminal を含む tool call で、コマンド出力が表示される
- [ ] 全文トランスクリプトと台詞の内容が食い違わない(台詞は全文の部分集合)

### 設定・配布

- [ ] `agents.json` の args のバージョンを変えると、次回接続でそのバージョンが起動する
- [ ] キャラクターパックのフォルダを切り替えると、再起動なしで立ち絵が変わる
- [ ] NSIS インストーラのサイズが 30MB 未満(Node と Agent は含まない)

---

## 4. MVP 後の最初の拡張候補(順不同、YAGNI を守る)

- Codex(`codex-acp`)での動作検証と、Agent ごとのキャラクター割り当て
- `confused` / `surprised` 表情の追加(`error` / `waiting_permission` に割り当て)
- `session/load` によるセッション再開
- plan の描画(クエストログ)
- ACP `terminal/*` の実装(ターミナル出力を Developer Layer にストリーム表示)
- 瞬き、背景画像、トランジション
