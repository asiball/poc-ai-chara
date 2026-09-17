# 既存プロジェクト調査

調査日: 2026-09-16

> この調査は当初「配布も視野に入れた汎用クライアント」を前提に行った。その後プロジェクトは「自分用」に絞ったため、§3 の GUI クライアント一覧や §4 の利用条件の一部(ブランド規約、配布時の認証方針)は参考情報の位置づけになる。結論(§6)と ACP まわり(§4)は自分用でもそのまま有効。
方法: GitHub リポジトリ(README・ソース)、公式ドキュメント、npm / crates.io、検索結果。
本セッションの実行環境では X / Reddit / Hacker News / YouTube / Zenn / Qiita / note / 一部の公式サイト
(agentclientprotocol.com、zed.dev、developers.openai.com、support.claude.com など)が取得できなかったため、
それらは検索結果のスニペットのみで確認しています。該当箇所は **[一次未確認]** と表記します。

凡例: **[事実]** = 一次情報で確認 / **[推測]** = 一次情報からの推定 / **[一次未確認]** = スニペットのみ / **[確認できず]** = 情報に到達できず
種別: **OSS** / **商用** / **公式実験** / **個人PoC**(記事・バイナリ・動画のみ)

---

## 1. 重点調査 3 プロジェクト

### 1.1 Agent Avatar(joyparkray/agent-avatar) — OSS

| 項目 | 内容 |
|---|---|
| 目的 | "Give your AI coding agent a body on your desktop." Agent の実行状態を Live2D デスクトップペットで可視化する **観察専用**ツール |
| 種別 / ライセンス / Stars / 最終更新 | OSS / MIT / 1 / 2026-09-04(v1.1.0) |
| 対応 Agent | Claude Code、Codex、Hermes、DeepSeek Harness、WorkBuddy |
| 技術 | Tauri 2 + Rust + TypeScript + PixiJS 8 + pixi-live2d-display、connector は Python 3(stdlib のみ)。macOS 14.2+ / Windows 10+ |

**アーキテクチャ [事実]**

```
[Claude Code] --hooks(stdin JSON)--> connectors/claude-code/agent-avatar-hook.py
[Codex CLI]   --hooks(stdin JSON)--> connectors/codex/agent-avatar-hook.py
        |  pascal_events.translate() → bridge/state_machine.py::update()
        v
  $TMPDIR/agent-avatar-state.<harness>.json   (atomic write + flock / msvcrt lock)
        |  5Hz ポーリング(desktop/src-tauri/src/hermes.rs::read_semantic_state)
        v
  desktop/src/semantic.ts → director.ts → live2d.ts(pixi-live2d-display)
```

- 接続方法は **Claude Code / Codex の hooks プラグイン**(`.claude-plugin/plugin.json`、`.codex-plugin/`)。
  登録イベント: SessionStart / SessionEnd / UserPromptSubmit / PreToolUse / PostToolUse / (Claude のみ PostToolUseFailure, PermissionDenied) / SubagentStart / SubagentStop / Stop
- トランスポートは **一時ディレクトリの JSON ファイル**。ソケット・HTTP・IPC は不使用
- Bridge の状態語彙: 8 状態(`idle < reviewing < writing < researching < executing < syncing < awaiting < error`、優先度付き)+ reaction overlay 2 種(`blocked`, `interrupted`)。`activity_from()` がホワイトリスト項目から 80 文字以内の `doing` 文を生成
- キャラクター状態マッピング: モデルフォルダ内の `avatar.json`(`motions` / `expressions` / `reactions` を state 名でキー付け)。無い場合は `model3.json` から合成
- 設計原則(bridge/README.md): 「harness は今何をしているかだけ発行し、表現は avatar 側が決める」「hook は常に exit 0」「Observe, never answer」

**会話機能**: なし [事実]。main.ts にテキスト入力・Agent 出力表示は存在しない。

**強み**
- Claude Code / Codex hooks を実地で踏んだ知見(exit 2 でツールがブロックされる、PowerShell の BOM、subagent イベントの除外、prompt_id と turn_id の違い)
- 並行ツール・順序乱れ・欠落イベントに耐える状態集約ロジックがテスト付きで単一ファイルに収まっている
- `avatar.json` の state → 表現マッピング形式がシンプル

**弱み**
- 会話機能ゼロ。設計思想が「観察専用」なので拡張方向が逆
- ファイルポーリングのため遅延と取りこぼしがある(lock 競合でイベントを捨てる)。Agent の発話テキストやツールの入出力はプロトコルに乗らない
- Live2D(Cubism 4)固定。desktop 内部ではレンダラ差し替えの抽象層が事実上ない
- Star 1、個人開発、リリース直後

**本プロジェクトとの関係**
- 「Agent Avatar は状態可視化が中心、本プロジェクトは会話そのものをノベル UI にする」という仮説は **正しい** [事実]。重複は可視化レイヤのみで、会話クライアント部分の重複はゼロ
- 参考にできる実装: `bridge/state_machine.py` の状態語彙と優先度、`pascal_events.py` の hooks → 内部イベント表、`avatar.json` 形式、`semantic.ts` のヒステリシス / 最低表示時間 / stale 判定
- 参考にできる UX: 「doing」の短文表示(ホワイトリストから生成、捏造しない)
- 再実装しない方がよい: 状態集約ロジック(MIT なので移植可)。ただし **hooks 経由の観察は本プロジェクトには不要**。SDK / ACP のストリームを直接握るクライアントは、同じ状態を自前で算出できる。hooks 方式が必要になるのは「外部ターミナルで動く Claude Code を横から観察する」モードを作る場合のみ

### 1.2 AvatarCode(sat315/avatar-code) — OSS

| 項目 | 内容 |
|---|---|
| 目的 | 「自分だけの AI キャラで Claude Code をリモート操作できる Web クライアント」 |
| 種別 / ライセンス / Stars / 最終更新 | OSS / MIT / 1 / 2026-03-18(以後更新なし) |
| 対応 Agent | Claude Code のみ |
| 技術 | Bridge: Node + `@anthropic-ai/claude-agent-sdk` + Hono + ws / Web: React 19 + Vite + Tailwind + react-markdown / Worker: Cloudflare Workers + D1 + R2 / 画像生成: Gemini API / トンネル: Cloudflare Tunnel |

**アーキテクチャ [事実]**

```
[ブラウザ / スマホ PWA]
   | HTTPS(Bearer)                 | WebSocket /ws?token=BRIDGE_SECRET
   v                               v
[Cloudflare Workers + D1 + R2]   [Cloudflare Tunnel] → [bridge/src/index.ts(ローカル PC :3456)]
(ペルソナ・履歴・衣装を保存)                            | query({ prompt: AsyncIterable, options: { cwd, model,
                                                        |          permissionMode, canUseTool, systemPrompt, resume } })
                                                        v
                                                   [@anthropic-ai/claude-agent-sdk] → ローカルの Claude Code
```

- **permission**: `canUseTool` で `permission_request` を WS 送信し、`approve` / `reject` で Promise を解決。allow / deny の 2 択のみ(allow-always なし)
- **セッション**: WebSocket 1 本につき 1 セッション。SDK の `session_id` を `resume` で再開。Rewind は D1 の行削除のみで SDK 側の履歴は巻き戻さない
- **cwd**: `PROJECTS_DIR` 配下に限定し、パストラバーサルを拒否
- **ファイル操作**: ツリー取得、テキスト系ファイルの読み書き、`git diff` を返す `get_diff`、`DiffModal` で自前描画
- **ペルソナ**: `bridge/personas/<id>.md` を `systemPrompt` に **置換**(デフォルト system prompt への追記ではない。コーディング能力に影響する可能性あり [推測])
- **認証**: コード・README のどこにも `ANTHROPIC_API_KEY` / `claude login` の記述がない。前提条件は「Claude Code がインストール済み」のみ。ローカルの Claude Code の既存ログインを Agent SDK が使う前提と **推測**される
- **ストリーミング**: `includePartialMessages` 未使用。content ブロック単位で届く

**チャット UI**: Discord ライク 3 ペイン。Markdown、Prism ハイライト、ThinkingBlock、ToolActivityCard(Edit は疑似 diff、Bash は出力 500 文字)、画像添付、PWA。

**キャラクター表現**: **静止画 1 枚** [事実]。表情差分なし、状態やテキストによる表情切替なし。「ワードローブ」= Gemini 画像生成で衣装差分を増やす、「自撮り」= 指示文から画像生成。

**強み**
- Claude Agent SDK を WebSocket ブリッジにする最小実装として非常に読みやすい(`query` + 入力ジェネレータ + `canUseTool` の Promise 保留)
- permissionMode・resume・cwd 制限・ペルソナ注入という「会話クライアントに必要な要素」が一通り揃う
- モバイル / リモート運用の実例

**弱み**
- Claude Code 専用。認証方式が文書化されていない
- キャラクター表現が静止画のみ。ノベル文法(名前欄、テキスト送り、表情差分、選択肢)はない
- Cloudflare 依存、テストなし、6 か月更新停止

**本プロジェクトとの関係**
- 重複度: **高**。「キャラ付き UI から Claude Code を操作する会話クライアント」という目的は同一
- ただし差分は明確: (1) 表情差分と状態連動がない、(2) ノベル文法がない、(3) Claude Code 専用で ACP 非対応、(4) Cloudflare 前提の Web アプリでデスクトップアプリではない、(5) 更新停止
- **率直な評価**: 「静止画キャラ付き Claude Code チャット」だけなら AvatarCode で足りる。本プロジェクトの存在意義は、ADV 文法(台詞 / 選択肢 / 表情差分 / 詳細レイヤ分離)と Agent 差し替えにある。それを作らないなら新規開発の理由はない
- 参考にできる実装: `bridge/src/index.ts` の WS メッセージ設計(`start` / `input` / `approve` / `reject` / `interrupt` / `get_diff`)、`ToolActivityCard.tsx` のツール別要約、cwd トラバーサル防御
- 再実装しない方がよい: Cloudflare Workers / D1 の履歴層、Gemini 衣装生成

### 1.3 ACP UI(formulahendry/acp-ui) — OSS

| 項目 | 内容 |
|---|---|
| 目的 | "A modern, cross-platform client for the Agent Client Protocol (ACP) on desktop, mobile, and the web" |
| 種別 / ライセンス / Stars / 最終更新 | OSS / MIT / 477 / 2026-05-25(v0.1.16) |
| 対応 Agent(既定設定) | GitHub Copilot、Claude Code、Gemini CLI、Qwen Code、Auggie、Qoder、Codex CLI、OpenCode、OpenClaw、Kiro CLI、Hermes(11 種) |
| 技術 | Tauri 2 + Vue 3 + Pinia + `@agentclientprotocol/sdk`(型のみ利用)、Rust `agent.rs` / `config.rs`。Windows / macOS / Linux / Android、Web 版(WebSocket のみ) |

**アーキテクチャ [事実]**

```
[Vue UI] → stores/session.ts → lib/acp-bridge.ts(AcpClientBridge implements Client、自前 JSON-RPC)
                                       | AcpTransport { send, onMessage, onClose, close }
                    ┌──────────────────┴──────────────────┐
           transport/stdio.ts                      transport/websocket.ts
           invoke spawn_agent / send_to_agent      NDJSON over WS(Bearer は subprotocol に折り込み)
           listen agent-message / agent-stderr
                    |
           src-tauri/src/agent.rs
           Windows: cmd /C <command> <args> + CREATE_NO_WINDOW
           Unix:    $SHELL -l 経由で PATH を継承
```

- 起動コマンド例: `npx @agentclientprotocol/claude-agent-acp@latest`、`npx @zed-industries/codex-acp@latest`、`npx @google/gemini-cli@latest --experimental-acp`
- 設定: `agents.json`(`{ transport?, command?, args?, env, url?, headers? }`)
- ACP 実装状況: `initialize` / `authenticate` / `session/new`(cwd, mcpServers: []) / `session/load` / `session/prompt`(text のみ) / `session/cancel` / `session/set_mode` / `session/set_model`。Client 側は `session/request_permission`、`fs/read_text_file`、`fs/write_text_file`(Desktop のみ)。**`terminal/*` は未実装**
- `session/update` 処理: user_message_chunk / agent_message_chunk / agent_thought_chunk / tool_call / tool_call_update(status・title のみ) / current_mode_update / available_commands_update。**`plan` 未処理、tool_call の content(diff / terminal)は保存・表示しない**
- permission UI: ACP の `options[]`(allow_once / allow_always / reject_once / reject_always)をそのままボタン化
- model / mode 選択、cwd ピッカー、session 履歴の保存と `session/load` 再開、全 JSON-RPC の Traffic Monitor

**強み**
- 11 Agent を同一 UI で扱える実証。Tauri + Rust で **Windows の npx 起動問題(.cmd、コンソールウィンドウ、PATH)を解決済み**
- 公式 SDK の型に沿った実装で、ACP 主要 RPC の使い方の実例として読みやすい
- リモート構成(WebSocket + stdio-to-ws)の実例

**弱み**
- terminal / diff / plan 未対応。チャット UI は素朴(`marked` + v-html)。画像添付なし
- Application Insights テレメトリ組み込み(流用時は削除が必要)
- 2026-05-25 以降コミットなし

**本プロジェクトとの関係**
- 重複度: 中。「複数 Agent と会話する開発クライアント」の骨格が重複。演出層はゼロ
- 参考にできる実装: `agents.json` スキーマと各 Agent の起動コマンド、`agent.rs` のプロセス生成、`handleSessionUpdate` の分岐、permission options → ボタン変換、`SavedSession` + `session/load`
- 再実装しない方がよい: ACP の JSON-RPC 層(公式 SDK の `ClientSideConnection` + `ndJsonStream` を使う)、OS 別シェル解決ロジック(`agent.rs` をそのまま流用)

---

## 2. キャラクター型 AI の先行事例(UX 参考)

### 2.1 RyzaChat:AI — 商用(ゲーム)

| 項目 | 内容 |
|---|---|
| 正式名称 | 『RyzaChat:AI ライザと創るあなただけのひと夏の夢物語』 [一次未確認] |
| 提供元 | SpiralAI(コーエーテクモゲームス / ガストからライセンス・監修) [一次未確認] |
| 配信 | 2026-08-25 配信開始(iOS / Android、日本国内。当初 8/18 予定をサーバー増強で延期) [一次未確認] |
| 価格 | 3 日間無料 → PRO 月額 980 円 / 年額 6,000 円。PRO でも会話は 15 回 / 10 日、超過分はトークン従量課金(複数記事で一致) [一次未確認] |
| 技術方針 | 画像・動画生成 AI は不使用(監修イラスト)、独自 LLM、声優収録音声ベースの音声合成 [一次未確認] |

**UX 上の観点(記事から整理。スクリーンショットは未確認)**
- 会話だけでゲームが進む「RPG モード」と雑談用「チャットモード」。ボタン・コマンド操作なし
- 会話内容に応じて表情・仕草が変化。生成画像ではなく用意された差分・モーションを LLM 出力に紐づけていると **推測**
- 「キャラクターと会話している」と感じさせる要素: 公式ライセンス + 本人声優、会話が即ゲーム進行に接続される(会話に「結果」がある)、表情・仕草の同期
- 主要な不満は「会話回数制限」「記憶の一貫性」。**会話回数をメーター化して見せることが体験を壊した実例**

**本プロジェクトとの違い**: RyzaChat は「LLM に人格を付けて会話するゲーム」。本プロジェクトは既存 Agent の出力を演出する開発ツールで、人格生成はしない。転用できるのは「発話 → 世界側の結果 → 表情差分で返す」ループ設計と、レート制限の見せ方への警戒。

### 2.2 Codex Pets — 公式(恒久機能)

- 2026 年 5 月初旬に Codex app / ChatGPT desktop app 向けに導入 [一次未確認、複数二次記事で一致]
- フローティングオーバーレイで **Running / Needs input / Ready(for review) / Blocked** を表示。ペット下から入力・音声でチャット開始可。Codex CLI は `/pet`(iTerm2 3.6+ / Kitty / Sixel 対応ターミナル必須、tmux 不可)。IDE 拡張は非対応
- ビルトイン 8 種、`/hatch` で画像から AI 生成。形式は `openai/skills` の hatch-pet スキルが規定 [事実]: `pet.json` + `spritesheet.webp`(1536×1872、8 列×9 行)、**9 状態 = idle / running-right / running-left / waving / jumping / failed / waiting / running / review**。アニメのタイミングはアプリ側ハードコード(openai/codex #20863)
- コミュニティ: Petdex(crafter-station/petdex、4.1k★、MIT、Codex / Claude Code / Gemini CLI 等対応)、awesome-codex-pets(版権キャラのスプライトが多数。権利面は要注意)
- **本プロジェクトとの関係**: Agent 状態の粒度は粗い 4 状態。会話 UI ではない。「状態語彙は 4〜6 段階で十分」という実証として参考。Codex pet 形式を読めるようにする価値は低い(スプライト形式が本プロジェクトの立ち絵と合わない)

### 2.3 Claude Code `/buddy` — 公式実験(一時)

- 2026-03-31 に npm `@anthropic-ai/claude-code` v2.1.88 へソースマップが混入して全ソースが流出し、Tamagotchi 風コンパニオン「BUDDY」が発見される [一次未確認、複数報道]
- 2026-04-01 v2.1.89 で `/buddy` 提供(「/buddy is here for April 1st」)。2026-04-09 v2.1.97 で changelog 記載なしに削除。Issue #45596「Bring Back Buddy」は open、#47683 は closed as not planned [事実]。2026-09-16 時点で公式 changelog に復活の記載なし [事実]
- 仕様: 18 種、レアリティ、3 フレーム ASCII アニメが入力ボックス横に常駐し作業に吹き出しで反応。`~/.claude.json` の `companion` に保存
- **April Fools 企画として設計され約 1 週間で撤去された一時実験**。ただし「Sub-agent ごとの buddy」「文脈反応」などの要望が出ており、ユーザーが Agent 状態の擬人化を求めている証跡
- 公式ハードウェア参考実装 `anthropics/claude-desktop-buddy` [事実]: Claude Desktop / Cowork / Claude Code の BLE API リファレンス。ESP32 上に ASCII ペットを表示し、**承認待ちで焦れる / デバイスから Allow・Deny できる**。「公式サポート機能ではない」と明記
- 復活 OSS: ramarivera/coding-buddy(457★)、Ido-Levi/claude-code-tamagotchi(436★)、TeXmeijin/claude-code-mascot-statusline(日本人作者、コンテキスト消費で毛色が変わる)ほか
- **"Clawd"**(起動時のオレンジ色 ASCII 生物): 製品内・公式ドキュメントに名称記載なし。Issue #8536(文書化要望)open。名称が公式かは **確認できず**

### 2.4 コミュニティ・個人事例(コーディング Agent に実際に接続しているもの)

接続方式の凡例: hooks = Claude Code / Codex hooks、log = `~/.claude/projects` 等の JSONL 監視、PTY = 擬似端末で CLI を駆動、MCP = MCP サーバとして Agent から呼ばれる。

#### 2.4.1 デスクトップペット / アバター(状態可視化が主目的)

| 事例 | 種別 | 実装度 | 接続 | 表現 | 重複度 / 所見 |
|---|---|---|---|---|---|
| clawd-on-desk(chrono-meta) | OSS(AGPL-3.0) | 高、3 OS | hooks + HTTP permission hook + log | ピクセルアート、12+ 状態(subagent 専用状態あり)、Codex pet パック読込可 | 中。**ペットの吹き出しから Allow / Deny(Ctrl+Shift+Y/N)** = キャラ経由の権限承認の先例 |
| Copiwaifu(Panzer-Jack) | OSS(MIT、42★) | 高、Tauri、mac/Win | hooks → ローカル HTTP :23333。Claude Code / Copilot / Codex / Gemini / OpenCode | **Live2D**。idle / thinking / tool_use / error / complete / needs_attention をモーションに紐付け。「AI Talk」= セッションメタ情報から LLM が短い吹き出し文を生成 | 中〜高。状態 → 表現マッピング、完了時の一言生成が参考 |
| CodeWaifu(flowinginthewind700) | OSS(MIT) | 中〜高 | hooks → loopback relay。Codex / Claude Code | Live2D、Matcha-TTS、リップシンク | 中〜高。**Codex スレッドへはメッセージ投入可、Claude Code は注入 API が無くクリップボード止まり** = 双方向化の壁を明記 |
| cc-mascot(kazakago) | OSS(Apache-2.0) | 高、mac/Win | log 監視(chokidar)。Claude Code / Codex / Gemini CLI | **VRM 3D**、テキストから感情推定 → 表情、VOICEVOX / AivisSpeech でリップシンク | 中〜高。日本語 TTS + 感情推定 → 表情の実装参考 |
| mascot-haihu-v2(Hexagon0423) | OSS(MIT、素材別) | 中、Windows 専用 | Claude Code / Claude Desktop / ChatGPT Desktop 用コネクタ | ずんだもん Live2D、表情プリセット多数、素材は各自入手 | 中。素材ライセンスを本体から分離する運用例 |
| V1R4(Kunnatam) | OSS(MIT、26★) | 高 | hooks が応答内の `<tts>` タグを抽出 → ローカル TTS | VRM、Kokoro TTS、状態別グロー | 中。「Agent に発話用タグを吐かせる」プロンプト設計 |
| Pipsqueak / Pets-for-Claude-Code(tristanmuzzu) | OSS(MIT) | 中、Tauri | hooks + transcript 読取 | ピクセル。**Claude が書いた直近の一文をナレーションとして表示** | 中〜高。「Agent の地の文をそのまま台詞にする」発想はノベル UI に直結 |
| claude-buddy(tumourlove) | OSS(ISC) | 中、Windows Electron | log 監視 | ピクセル Clawd がアイソメ部屋でツール種別ごとの家具へ移動 | 中。「ツール → 場所 / 動作」のメタファー |
| varie-claude-avatar、Claude-pet(DecoudJuan)、claude-pet(IMMINJU)、agent-pet(lijuno)、pets-driven、CoPet、ccpet(musoukun、バイナリのみ)、claude-sidekick、ascii-avatar | OSS / 個人PoC | 中 | hooks(HTTP / TCP / socket / ファイル) | Spine / ピクセル / 絵文字 / ASCII | 低〜中。状態可視化のみ |
| desktop-mascot-mcp(rennosuke-haresu) | OSS(MIT) | 中 | **MCP** `speak(text, emotion, animation)` | VRM + VOICEVOX | 中。「Agent 自身がツール呼出で表情・台詞を指定」方式の例 |
| **Clawd Face**(anam-org/clawd-face) | OSS(MIT)「vibe-coded PoC」 | 中 | **PTY で Claude Code を駆動**。音声 → 文字起こし → Claude Code → 応答をアバターが発話 | 実写系ビデオアバター + ターミナル / diff パネル | **高**(双方向・会話で Agent を操作する開発クライアント)。ただしノベル風ではなくビデオ会議風 |
| 「動く秘書 Ria」(note)、zundamon-notify(LCL ブログ)、VOICEVOX 通知系(Qiita / Zenn 多数) | 個人PoC(記事) | 低〜中 | hooks | Live2D / 立ち絵 + 吹き出し / 音声のみ | 低〜中 [一次未確認] |

#### 2.4.2 ゲーム / RPG 的可視化(multi-agent 含む)

| 事例 | 種別 | 内容 | 接続 | 重複度 |
|---|---|---|---|---|
| Claude Quest(Michaelliv、196★、MIT) | OSS | ターミナル内ピクセル RPG。Read = 呪文、Bash = 攻撃、エラー = 被ダメ、**マナバー = 残りコンテキスト** | log | 中。「Agent 動作 → 演出語彙」「コンテキスト → 資源ゲージ」 |
| Agent-Quest(FulAppiOS、138★、MIT) | OSS | 中世村ダッシュボード。各セッション = 勇者、Library(Read)/ Forge(Edit)/ Arena(Bash)へ移動。Claude Code + Codex 同時 | log + hook | 中。multi-agent の「パーティ」表現 |
| Slime(bitqs、MIT) | OSS | statusline HUD + PixiJS アリーナ。prompt = ボス、tokens = 資源 | hooks | 低〜中 |
| agent-town(geezerrrr、239★) | OSS | ピクセルオフィスを Agent が歩き回る | OpenClaw gateway | 低〜中 |
| **kourai-khryseai**(ajbarea、1★、MIT) | OSS(学術ポスターあり) | A2A プロトコルの 10 エージェントが実際に計画・実装・テスト・コミット。UI は CLI / Pygame(JRPG 風ポートレート + ストリーミング吹き出し + TTS)/ **Ren'Py ビジュアルノベル(好感度、待機中 Agent の雑談、選択肢、セーブ / ロード、`vn-bridge` で接続)** | 自前 A2A + MCP(Claude Code / Codex ではない) | **最高**。「ノベルゲーム UI でコーディングマルチエージェントと会話」を Ren'Py で実装した唯一の確認例。ただし Claude Code / Codex を直接扱わない独自 Agent |
| 「将軍システム」(Zenn / note / Qiita) | 個人記事 | 将軍・家老・足軽の階層メタファーで Claude Code マルチエージェント運用。UI は tmux でキャラ表示なし | Agent Teams | 低(命名・役割設計の参考)[一次未確認] |

#### 2.4.3 ビジュアルノベル + LLM(コーディング Agent 非接続、UI 参考)

- RenAI-Chat(Rubiksman78、CC0): Ren'Py で VN 風にチャットボットと会話、TTS / STT、表情
- ai-galgame(ZzzhangLK): LLM 生成の VN、ダイアログボックス + 選択肢
- SillyTavern Visual Novel Mode + Character Expressions: 28 感情スプライトを応答の感情分類で自動切替 [一次未確認]
- 「Claude Code でノベルゲームを作る」記事は多いが、「Claude Code をノベルゲーム UI から操作する」記事は **確認できず**

### 2.5 汎用キャラクター AI デスクトップ(比較用)

| プロジェクト | ★ | アバター | コーディング Agent 連携 |
|---|---|---|---|
| Open-LLM-VTuber | 13.8k | Live2D | MCP ツール呼出あり。Claude Code / Codex 連携なし |
| AIRI(moeru-ai) | 49.2k | Live2D + VRM | Roadmap に「Claude Code hooks support」が未チェック項目として存在 [一次未確認] |
| Amica(semperai) | 1.6k | VRM、Tauri | 「Agent Frontend」を名乗るが Claude Code / Codex / MCP の記載なし |
| ChatVRM(pixiv、archived) | – | VRM | なし |

この層は全て「LLM に人格を付けて会話」型。Agent 状態の可視化やコーディングタスクの実行には接続していない。

---

## 3. 実用的なコーディング Agent GUI / ACP クライアント

### 3.1 Claude Code GUI

| Project | 種別 | 接続方式 | Permission | Diff / Tool | Stack | Win | License | Stars | メンテ |
|---|---|---|---|---|---|---|---|---|---|
| Claude Code Desktop(公式) | 公式 | 非公開(Electron と報道 [一次未確認]) | Manual / Accept edits / Plan / Auto / Bypass | インラインコメント付き diff viewer、統合ターミナル | – | ○ | 商用 | – | 活発 |
| Claude Code VS Code 拡張(公式) | 公式 | CLI を同梱しサブプロセス駆動(`claudeProcessWrapper`)[事実] | side-by-side diff で許可、ルール編集 | ○ | – | ○ | 商用 | – | 活発 |
| Remote Control(公式) | 公式 | ローカル `claude` に claude.ai / スマホから接続、outbound HTTPS のみ | スマホから承認 | diff ペイン | – | ○ | Pro/Max/Team、API key 非対応 | – | 活発 |
| opcode(旧 Claudia) | OSS | `claude -p --output-format stream-json --dangerously-skip-permissions` [事実] | **常に skip** | checkpoint 間 diff | Tauri 2 + React + Rust | ○ | AGPL-3.0 | 22.4k | 停滞(2025-10) |
| Claude Code UI(siteboon/claudecodeui) | OSS | Node サーバで Agent SDK `query()` [事実] | `canUseTool` → WS `permission_request`、全ツール既定 OFF | tool 表示、CodeMirror、git | React + Node + WS、Electron コンパニオン | ○ | AGPL-3.0 | 13.7k | 活発 |
| Happy Coder(slopus/happy) | OSS | Claude = Agent SDK、Codex = app-server JSON-RPC、Gemini = 独自 [事実] | スマホへ転送、push 通知 | チャット中心 | Expo + Node + relay | – | MIT | 23.8k | 活発 |
| claude-code-webui(sugyan) | OSS | ローカル `claude` CLI ラップ | ダイアログ | – | Deno + React | – | MIT | 1.1k | アーカイブ(2026-05) |
| ccgui / desktop-cc-gui | OSS | 各 CLI 向け runtime adapter、portable-pty | 追加ディレクトリ付与 | tool call 展開、Git Diff | Tauri 2 + React | ○ | MIT | 4.2k | 活発 |

### 3.2 Codex GUI

| Project | 種別 | 接続方式 | 承認 | Stack | Win | License | Stars |
|---|---|---|---|---|---|---|---|
| Codex app(公式) | 公式 | app-server(Electron と報道 [一次未確認]) | ○ | – | ○(2026-03) | 商用 | – |
| CodexGui(wieslawsoltes) | OSS | app-server stdio + ws、NSwag 生成 DTO | pending approval | .NET + Avalonia | ○ | MIT | 16 |
| Codex-Official-App-Server-Web(Shermanxwz) | OSS | app-server ゲートウェイ | ○ | Node | – | MIT | 1 |
| codex-desktop(jake-enastudio) | OSS | `codex exec --json` | なし | Electron + node-pty | ○ | 独自 | 0 |

### 3.3 ACP クライアント / マルチ Agent クライアント

| Project | 種別 | 対応 Agent | Permission / Diff | Stack | Win | License | Stars | 備考 |
|---|---|---|---|---|---|---|---|---|
| Zed | OSS(エディタ) | Claude Agent / Codex / Gemini CLI / OpenCode / Copilot / Cursor + Registry | Allow once / Deny once / Always for tool / pattern、hunk 単位 keep / reject | Rust + GPUI | – | GPL/AGPL | – | ACP の本家クライアント |
| JetBrains AI Assistant | 商用 | 2026.1 から ACP Registry(約 50 Agent) [一次未確認] | – | Kotlin | ○ | 商用 | – | ACP 共同ガバナンス |
| agent-shell.el(xenodium) | OSS | 15+ | バッファ内ボタン、diff buffer | Elisp(`acp.el` は UI 非依存) | – | GPL-3.0 | 1.9k | UI 非依存 ACP クライアントの設計例 |
| vscode-acp(formulahendry) | OSS | 11 プリセット | `acp.autoApprovePermissions` | TS | ○ | MIT | 379 | |
| **ACP UI**(formulahendry) | OSS | 11 | approve / deny、tool call、Traffic Monitor | **Tauri 2 + Vue 3** | ○ | MIT | 477 | §1.3 参照 |
| AionUi(iOfficeAI) | OSS | 20+(ACP + MCP) | Agent ごとの承認ダイアログ | Electron | ○ | Apache-2.0 | **32.9k** | 最大級の ACP デスクトップ |
| Gold Band(diodeme) | OSS | Claude Code / Codex | permission mode、raw frame 検査 | Tauri 2 + React | ○ | AGPL-3.0 | 85 | |
| Jockey(recailai) | OSS | claude-code / codex をロールとして ACP 接続 | – | Tauri 2 + SolidJS | ○ | MIT | 25 | マルチ Agent 会話 |
| Clark Code(clark-labs-inc) | OSS | ACP + ローカルモデル | 明示的承認、tool call / plan / diff | Tauri 2 + React、Rust crate 分割(`agent-core`, `provider-acp`) | ○ | Apache-2.0 | 7 | クレート分割が参考 |
| acp-components(zvzuola) | OSS(ライブラリ) | 任意 | Permission / ToolCall / DiffViewer / Plan 部品 | `@acp-components/core`(zustand + transport)+ `@acp-components/react` | – | MIT | 57 | UI 部品ライブラリ、Tauri サンプル同梱 |
| acpx(openclaw) | OSS(CLI / ライブラリ) | Pi / OpenClaw / Codex / Claude Code | `--approve-*` ポリシー | TS、`acpx/runtime` 公開 | – | MIT | 3.3k | headless セッション管理 |
| Toad(batrachianai) | OSS | 12 | diff | Python + Textual(TUI) | ×(WSL) | AGPL + 商用 | 3.4k | |
| Vibe Kanban(BloopAI) | OSS | Claude / Codex / Gemini / Copilot ほか | executor 毎に独自(Claude = `claude -p --permission-prompt-tool=stdio`、Gemini = ACP) | Rust + React | ○ | Apache-2.0 | 28.1k | 開発元閉鎖、コミュニティ維持 |
| Superset / Emdash / MonoCode / Conductor | OSS / 商用 | 20+ | PTY で TUI をそのまま表示 | Electron / Tauri / native | 一部 | 各種 | 14.3k / 5.8k / – / – | **PTY 型はノベル UI に不向き**(構造化イベントが取れない) |

観察: マルチ Agent 系は「PTY で TUI をそのまま表示」型と「構造化プロトコル(ACP / stream-json / app-server)で UI を自前描画」型に二分される。ノベルゲーム風 UI は後者が必須。

---

## 4. Agent 接続方式・認証・利用条件(2026-09-16 時点)

### 4.1 ACP(Agent Client Protocol)

- **安定版 protocolVersion = 1**。v2 は 2026-07-20 に Draft 公開 [事実]
- ガバナンス: Zed と JetBrains の共同(Lead Core Maintainers = Zed / JetBrains 各 1 名)、Apache-2.0 [事実]
- Transport: stdio(改行区切り JSON-RPC 2.0)が主。Streamable HTTP は draft、WebSocket は v1 仕様に未記載(RFD Active)[事実]
- 主要メソッド [事実]
  - Agent 側: `initialize` / `authenticate` / `session/new` / `session/prompt`(必須)、`session/load` / `session/set_mode` / `session/set_config_option` / `session/list` / `session/resume` / `session/fork`(任意)、通知 `session/cancel`
  - Client 側: `session/request_permission`(必須)、`fs/read_text_file` / `fs/write_text_file` / `terminal/create` / `terminal/output` / `terminal/wait_for_exit` / `terminal/kill` / `terminal/release`(任意)、通知 `session/update`
  - `session/update` の種別: `user_message_chunk` / `agent_message_chunk` / `agent_thought_chunk` / `tool_call` / `tool_call_update` / `plan` / `available_commands_update` / `current_mode_update` / `config_option_update` / `usage_update`
  - `tool_call`: `kind`(read / edit / delete / move / search / execute / think / fetch / other)、`status`(pending / in_progress / completed / failed)、`content`(通常ブロック、**diff `{path, oldText, newText}`**、terminal)、`locations`、`rawInput` / `rawOutput`
  - `session/request_permission`: options の `kind` = `allow_once` / `allow_always` / `reject_once` / `reject_always`、outcome = `selected {optionId}` / `cancelled`
  - `session/prompt` の `stopReason`: `end_turn` / `max_tokens` / `max_turn_requests` / `refusal` / `cancelled`
  - 認証: authMethods は `type: "agent"`(プロトコル内で `authenticate`)と `type: "terminal"`(クライアントが対話プロセスを起動し終了を待つ)
- 公式 SDK [事実]: TypeScript `@agentclientprotocol/sdk` 1.4.0(Web Streams 対応で Node / ブラウザ両対応)、Rust `agent-client-protocol` 2.1.0、Python / Java / Kotlin
- レジストリ: agentclientprotocol/registry(約 50 Agent)。JetBrains 向け派生あり [事実]

### 4.2 Claude Code

| 経路 | 内容 |
|---|---|
| **ACP アダプタ** `@agentclientprotocol/claude-agent-acp`(旧 `@zed-industries/claude-code-acp`) | v0.78.0(2026-09-15)、Claude Agent SDK 0.3.270 上に実装、Node ≥22、Apache-2.0。permission / tool call(diff 付き)/ terminal / edit review / slash commands / modes / MCP / session load・resume・fork / subagent transcript 対応 [事実]。認証は `type: "terminal"` で `claude-ai-login`(`claude auth login --claudeai`)と `console-login`(API 課金)を提示 [事実]。**Anthropic 非公式**(保守は agentclientprotocol org)。Anthropic の Issue #6686「Add support for ACP」は closed as not planned、CLI に `--acp` フラグなし [事実] |
| **Claude Agent SDK** `@anthropic-ai/claude-agent-sdk` | Claude Code CLI をサブプロセスとして spawn。`canUseTool(toolName, input)` で permission を処理(allow / deny / updatedInput / remember)、`permissionMode`、`cwd`、`resume` / `forkSession`、`hooks`、`mcpServers`、`settingSources`(既定 user / project / local = CLAUDE.md・hooks・.mcp.json・skills を CLI と同様に読む)、`includePartialMessages` [事実]。diff は Edit ツール入力から自前解釈が必要 |
| **CLI headless** `claude -p --output-format stream-json --input-format stream-json` | `--permission-prompt-tool <MCP tool>`、`--permission-mode`、`--resume`、`--add-dir`、`--bare` 等 [事実]。**stdin の双方向仕様と permission の control メッセージは未文書化**(Issue #24594 closed as not planned)[事実]。Node / Python 以外から使う場合のみ検討 |

**認証・利用条件 [事実、code.claude.com/docs/en/legal-and-compliance および agent-sdk/overview]**
- 「Unless previously approved, Anthropic does not allow third party developers to offer claude.ai login or rate limits for their products, including agents built on the Claude Agent SDK. Use the API key authentication methods instead.」
- 「Anthropic does not permit third-party developers to offer Claude.ai login into their own applications, or to route requests through Free, Pro, or Max plan credentials on behalf of their users. Moreover, developers may not collect, store, or intermediate Claude.ai credentials or session tokens — sign-in to a Claude account must complete through Anthropic's own flow.」
- 「Nor does it prevent an end user from signing in to the unmodified Claude Code binary with their own Claude subscription」
- 「Advertised usage limits for Pro and Max plans assume ordinary, individual usage of Claude Code and the Agent SDK.」
- ブランド: 「Claude Code」「Claude Code Agent」の名称使用不可、「Claude Agent」「Powered by Claude」は可
- 時系列 [一次未確認、複数報道で一致]: 2026-02-20 法務ページに上記節追加。2026-04-04 Anthropic の Boris Cherny が「サブスクは第三者ツール経由の利用をカバーしない」と告知。2026-05〜06「Agent SDK credit」(サブスクの Agent SDK 利用分を別プールに)を告知、**2026-06-16 に「この課金変更はまだ施行しない。ACP 利用、`claude -p`、Agent SDK、Agent SDK ベースの第三者アプリはサブスクで従来通り動作する。将来変更時は事前通知する」と発表**

**結論**: 技術的には SDK / ACP 経由でサブスク OAuth が動作し、2026-06-16 時点で Anthropic も「従来通り動作」と告知している。一方で「第三者開発者が自社アプリに Claude.ai ログインを提供 / 資格情報を仲介」は明示的に不許可。**個人が自作クライアントを自分のサブスクで使うことが前者か後者かは公式に明文化されておらず、確認できない**。課金モデルは再変更が予告されている。

### 4.3 OpenAI Codex

| 経路 | 内容 |
|---|---|
| **ACP アダプタ** `@agentclientprotocol/codex-acp` | v1.12.0、TypeScript、`@openai/codex ^0.154.0` を同梱、Apache-2.0。旧 Rust 版 `zed-industries/codex-acp` は 2026-07-22 アーカイブ [事実]。app-server を起動して ACP に変換。approval / sandbox mode、diff、terminal、subagent、MCP、slash commands 対応。認証は ChatGPT login / `CODEX_API_KEY` / `OPENAI_API_KEY` [事実]。**OpenAI 非公式** |
| **`codex app-server`**(公式) | JSON-RPC(stdio または `--listen ws://`)。`thread/start`(cwd, model, approval_policy, sandbox)/ `thread/resume` / `thread/fork` / `turn/start` / `turn/interrupt`、承認要求 `item/commandExecution/requestApproval`(accept / acceptForSession / decline)と `item/fileChange/requestApproval`、通知 `turn/diff/updated`、`item/agentMessage/delta` ほか。`account/login/start` は `{type:"chatgpt"}` / `{type:"apiKey"}` 両対応 [事実、ソース確認]。公式 Codex app / VS Code 拡張の基盤。`codex mcp-server` は 2026-09-05 に削除 [一次未確認] |
| **Codex SDK** `@openai/codex-sdk` / `codex exec --json` | `codex exec` を JSONL でラップ。`approvalPolicy`、`sandboxMode`、`workingDirectory`、`resumeThread`。**承認は非対話**、`file_change` は path / kind のみで diff 本文なし [事実] |

**認証・利用条件**
- README: 「Sign in with ChatGPT … as part of your Plus, Pro, Business, Edu, or Enterprise plan」[事実]
- Sam Altman 2026-05-01(X): 「you can sign in to openclaw with your chatgpt account now and use your subscription there!」[事実、一次投稿] → 第三者ハーネスでのサブスク利用を CEO が公言
- help.openai.com「Using Codex with your ChatGPT plan」: 「reselling access or using ChatGPT to power third-party services violates policy」[一次未確認]
- openai/codex Discussion #8338: OpenAI スタッフは「Apache ライセンスなので fork・改変は自由」と回答、第三者クライアントでのサブスク利用可否には直接回答なし [事実]
- OpenCode では 2026-03 に ChatGPT サブスク認証オプションが消失(anomalyco/opencode #16316)、理由の公式説明なし [事実]

**結論**: OpenAI は Anthropic より寛容だが、「ユーザー自身のサブスクを自作クライアントで使う」ことを明文で許可した規約は **確認できず**。「第三者サービスを ChatGPT で動かす / 再販」は禁止。

### 4.4 Gemini CLI

- `gemini --acp` で ACP 対応(`--experimental-acp` は旧フラグ)。`setSessionMode` で auto-approve 切替、fs はクライアント経由 [事実]
- 認証: Google アカウント OAuth(個人無料枠)、`GEMINI_API_KEY`、Vertex AI。第三者クライアントでの制限記述は **確認できず**

### 4.5 比較表(Agent × 経路)

| 観点 | Claude: ACP | Claude: Agent SDK | Codex: ACP | Codex: app-server | Gemini: ACP |
|---|---|---|---|---|---|
| Permission approval | ○ `request_permission` | ○ `canUseTool` | ○ | ○(2 系統の requestApproval) | ○ |
| Tool call 可視化 | ○ | ○ | ○ | ○ | ○ |
| Diff 取得 | ○ `content.diff` | △ Edit 入力を自前解釈 | ○ | ○ `turn/diff/updated` | ○ |
| Session resume | ○ load / resume / fork | ○ | ○ | ○ | ○ load |
| cwd 指定 | ○ | ○ | ○ | ○ | ○ |
| 設定継承(CLAUDE.md / AGENTS.md / hooks) | ○(SDK 既定)[推測: アダプタが上書きしない限り] | ○ `settingSources` | ○ CODEX_HOME 共有 | ○ | ○ |
| サブスク認証(技術) | ○ terminal auth | ○ | ○ ChatGPT login | ○ | ○ |
| サブスク認証(公式方針) | 第三者提供は不許可。個人自作は明文なし。課金変更は延期中 | 同左 | CEO は寛容。明文許可は未確認 | 同左 | 記述なし |
| メンテ主体 | agentclientprotocol org(非公式) | Anthropic 公式 | agentclientprotocol org(非公式) | OpenAI 公式 | Google 公式 |

---

## 5. 総合比較表

| Project | 種別 | Character UI | Novel UI | Claude Code | Codex | ACP | Chat | Agent State | Tool/Permission | Multi Agent |
|---|---|---|---|---|---|---|---|---|---|---|
| Agent Avatar | OSS | Live2D | × | ○(hooks) | ○(hooks) | × | × | ○(8 状態) | ×(観察のみ) | ○(5 harness、切替) |
| AvatarCode | OSS | 静止画 | × | ○(Agent SDK) | × | × | ○ | △(文字列) | ○(allow/deny) | × |
| ACP UI | OSS | × | × | ○ | ○ | ○ | ○ | △(tool status) | ○(ACP options) | ○(切替) |
| RyzaChat:AI | 商用 | 差分+音声 | ○(会話 UI) | × | × | × | ○ | – | – | × |
| Codex Pets | 公式 | スプライト | × | × | ○ | × | △(ペットから開始) | ○(4 状態) | ×(通知のみ) | ○(tray) |
| Claude `/buddy` | 公式実験(削除) | ASCII | × | ○ | × | × | × | △ | × | × |
| clawd-on-desk | OSS | ピクセル | × | ○ | ○ | × | × | ○(12+) | △(Allow/Deny のみ) | ○ |
| Copiwaifu | OSS | Live2D | × | ○ | ○ | × | × | ○ | × | ○ |
| cc-mascot | OSS | VRM + TTS | × | ○ | ○ | × | × | ○(感情推定) | × | ○ |
| Clawd Face | OSS(PoC) | ビデオアバター | × | ○(PTY) | × | × | ○(音声) | △ | △ | × |
| kourai-khryseai | OSS | JRPG / Ren'Py | **○** | ×(独自 Agent) | × | × | ○ | ○ | △ | ○ |
| Claude Code UI(siteboon) | OSS | × | × | ○ | ○ | × | ○ | ○ | ○ | ○ |
| Happy Coder | OSS | × | × | ○ | ○ | △ | ○ | ○ | ○ | ○ |
| AionUi | OSS | × | × | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| Zed | OSS | × | × | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| **本プロジェクト(計画)** | OSS | 立ち絵 + 差分 | **○** | ○(ACP) | ○(ACP) | ○ | ○ | ○ | ○ | 将来 |

---

## 6. 差別化分析

### 6.1 このプロジェクトを作る意義はあるか

**結論: ある。ただし新規に書く価値があるのは UI 層だけで、接続層は既存 OSS を土台にすべき。**

- 「Agent 状態 → キャラクター表示」は Codex Pets の公式化を含めて飽和している。ここで戦う理由はない
- 「キャラ付き UI から Claude Code を操作する」は AvatarCode と Clawd Face が存在するが、いずれも演出は静止画 / ビデオで、ノベル文法を持たず、Claude Code 専用で、更新が止まっているか PoC 止まり
- 「ノベルゲーム文法で Agent と会話する」は kourai-khryseai が Ren'Py で実装しているが、独自 Agent であり Claude Code / Codex を扱えない
- **「ADV 文法 × 実在の Claude Code / Codex × permission・diff を扱える実用クライアント」の組み合わせは確認できなかった**

### 6.2 「既存プロジェクトを組み合わせれば済む」か

済まない。理由:
- ACP UI / AionUi はチャット UI であり、台詞 / 選択肢 / 表情差分 / 詳細レイヤ分離という UX を後付けするには UI 層を書き直すことになる
- Agent Avatar / Copiwaifu などのペット系は観察専用で、Agent への入力経路を持たない(CodeWaifu が明記する通り、hooks 方式では Claude Code に注入できない)
- AvatarCode は最も近いが、Web + Cloudflare 構成、Claude 専用、表情なし、更新停止

ただし **部品としては大いに流用できる**:
- ACP 接続層: `@agentclientprotocol/sdk`(公式)、acp-ui の `agent.rs`(Windows 対応の spawn)
- 状態語彙と表情マッピング形式: Agent Avatar の `state_machine.py` / `avatar.json`
- Agent SDK ブリッジの設計: AvatarCode の `bridge/src/index.ts`(ACP を使わない場合の代替)
- UI 部品: acp-components(Permission / ToolCall / DiffViewer / Plan)

### 6.3 「人格付与」と「ADV 文法による表現」の分離

| | 人格付与 | ADV 文法による表現 |
|---|---|---|
| 何をするか | system prompt でキャラの口調・性格を注入する | Agent の出力・状態をそのまま台詞 / 表情 / 選択肢に翻訳する |
| 既存事例 | RyzaChat、SillyTavern、AvatarCode のペルソナ | kourai-khryseai(独自 Agent) |
| リスク | コーディング能力の劣化(AvatarCode は system prompt を置換している)、Agent の CLAUDE.md / AGENTS.md との衝突 | 長文 Markdown をテキスト送りにすると読みにくい。分離ルールの設計が要る |
| 本プロジェクト | **MVP では行わない**(将来、Agent 固有の設定に干渉しない範囲で `--append-system-prompt` 相当を検討) | **中心コンセプト** |

### 6.4 UX 上の参考点(ノベルゲーム的文法への翻訳)

1. **状態語彙は 4〜6 段階が実用的**。Codex Pets(4)、Claude `/buddy`、多数のペットが収束した粒度は「作業中・思考中・入力 / 承認待ち・完了・失敗」。立ち絵差分にそのまま対応できる
2. **「入力待ち」を最優先で目立たせる**。claude-desktop-buddy は承認待ちで「焦れる」、Codex Pets は Needs input を通知。ノベル UI なら「▼ 点滅 + 名前欄ハイライト + 選択肢提示」
3. **台詞と本文を分離する**。Pipsqueak は「Claude が実際に書いた直近の一文」を台詞化(捏造なし)。Copiwaifu はメタ情報のみを LLM に渡して短文生成(コード本文を渡さない)。**本文(ログ)と台詞(要約)を分離し、テキスト送りは台詞側にだけ適用する**
4. **表情差分の駆動方法は 3 通り**: (a) イベント → 固定マッピング(大半)、(b) テキスト感情推定(cc-mascot、SillyTavern)、(c) Agent 自身がツール呼出で指定(desktop-mascot-mcp)。MVP は (a)、将来 (b)(c) を追加可能にする
5. **permission は選択肢**。clawd-on-desk / claude-desktop-buddy が「キャラ経由の Allow / Deny」を先行実装。ACP の options はそのままノベルの選択肢になる
6. **資源ゲージ**: Claude Quest のマナ = コンテキスト残量。ただし RyzaChat の「会話回数メーター」は体験を壊した。ACP の `usage_update` を表示するなら控えめに
7. **待ち時間の埋め方**: pets-driven / kourai-khryseai の「手が空いた Agent が雑談する」。複数 Agent 対応時の候補
8. **法務・素材**: Live2D は商用時に別ライセンス、awesome-codex-pets には版権キャラが多数。**素材は同梱せず BYO(bring your own)** にする

---

## 7. 参考 URL(確認日 2026-09-16)

✓ = 本文取得・確認済 / △ = 取得不可、検索スニペットのみ

**重点 3 プロジェクト**
- ✓ https://github.com/joyparkray/agent-avatar (ARCHITECTURE.md, bridge/README.md, bridge/state_machine.py, bridge/pascal_events.py, connectors/*/hooks.json, docs/MODELS.md, desktop/src/*.ts, desktop/src-tauri/src/hermes.rs)
- ✓ https://github.com/sat315/avatar-code (bridge/src/index.ts, web/src/hooks/useBridgeWs.ts, web/src/components/*.tsx, worker/src/db/schema.sql, CLAUDE.md, SETUP.md) / △ https://zenn.dev/sat315/articles/d2cb2198257aa5
- ✓ https://github.com/formulahendry/acp-ui (src/lib/acp-bridge.ts, src/lib/transport/*, src/stores/session.ts, src-tauri/src/agent.rs, src-tauri/src/config.rs, src/components/PermissionDialog.vue) / ✓ https://acp-ui.github.io

**ACP**
- ✓ https://github.com/agentclientprotocol/agent-client-protocol (docs/protocol/v1/*.mdx, docs/get-started/*.mdx, docs/community/governance.mdx, docs/updates.mdx)
- ✓ https://github.com/agentclientprotocol/typescript-sdk / ✓ https://github.com/agentclientprotocol/rust-sdk / ✓ https://github.com/agentclientprotocol/registry
- ✓ https://github.com/agentclientprotocol/claude-agent-acp (README, package.json, src/acp-agent.ts, docs/permission-extension.md, CHANGELOG)
- ✓ https://github.com/agentclientprotocol/codex-acp / ✓ https://github.com/zed-industries/codex-acp (archived)
- ✓ https://github.com/google-gemini/gemini-cli (docs/cli/acp-mode.md, docs/get-started/authentication.mdx)
- △ https://agentclientprotocol.com / △ https://zed.dev/blog/anthropic-subscription-changes

**Claude Code**
- ✓ https://code.claude.com/docs/en/agent-sdk/overview / …/quickstart / …/typescript / …/permissions / …/sessions / …/claude-code-features
- ✓ https://code.claude.com/docs/en/legal-and-compliance / …/authentication / …/headless / …/cli-reference / …/desktop / …/vs-code / …/remote-control / …/changelog
- ✓ https://github.com/anthropics/claude-code/issues/6686 / …/issues/24594 / …/issues/45596 / …/issues/47683 / …/issues/8536 / …/issues/24926
- ✓ https://github.com/anthropics/claude-desktop-buddy
- ✓ https://github.com/agentclientprotocol/claude-agent-acp/issues/658
- △ https://support.claude.com/en/articles/15036540-use-the-claude-agent-sdk-with-your-claude-plan
- △ 流出・buddy 報道: https://futurism.com/artificial-intelligence/leaked-claude-code-tamagotchi / https://news.ycombinator.com/item?id=47590913 / https://www.claudeupdates.dev/version/2.1.89

**Codex**
- ✓ https://github.com/openai/codex (README, codex-rs/app-server-protocol/src/protocol/common.rs, codex-rs/app-server/README.md, codex-rs/protocol/src/protocol.rs, codex-rs/exec/src/cli.rs, sdk/typescript/README.md)
- ✓ https://github.com/openai/codex/discussions/8338 / ✓ https://github.com/openai/codex/issues/20863 / …/issues/27335
- ✓ https://x.com/sama/status/2050357911915028689
- ✓ https://github.com/openai/skills/blob/main/skills/.curated/hatch-pet/SKILL.md
- ✓ https://github.com/anomalyco/opencode/issues/16316
- △ https://developers.openai.com/codex/pets / …/app-server / …/changelog / △ https://help.openai.com/en/articles/11369540 / △ https://openai.com/index/unlocking-the-codex-harness/

**RyzaChat:AI**
- △ https://ryzachat-ai.go-spiral.ai/en / △ https://www.famitsu.com/article/202608/85432 / △ https://automaton-media.com/articles/newsjp/ryazai-20260804-458244/ / △ https://www.4gamer.net/games/029/G102959/20260825010/ / △ https://prtimes.jp/main/html/rd/p/000000092.000120221.html

**キャラクター系 OSS / 個人事例**
- ✓ https://github.com/chrono-meta/clawd-on-desk / ✓ https://github.com/Panzer-Jack/Copiwaifu / ✓ https://github.com/flowinginthewind700/codewaifu / ✓ https://github.com/kazakago/cc-mascot / ✓ https://github.com/Hexagon0423/mascot-haihu-v2 / ✓ https://github.com/Kunnatam/V1R4 / ✓ https://github.com/varie-ai/varie-claude-avatar / ✓ https://github.com/DecoudJuan/Claude-pet / ✓ https://github.com/IMMINJU/claude-pet / ✓ https://github.com/lijuno/agent-pet / ✓ https://github.com/tristanmuzzu/Pets-for-Claude-Code / ✓ https://github.com/tumourlove/claude-buddy / ✓ https://github.com/young1the/pets-driven / ✓ https://github.com/ChanceYu/CoPet / ✓ https://github.com/musoukun/ccpet-release / ✓ https://github.com/juan-rome/claude-sidekick / ✓ https://github.com/Angelopvtac/ascii-avatar / ✓ https://github.com/rennosuke-haresu/desktop-mascot-mcp / ✓ https://github.com/Crackxsy/nox / ✓ https://github.com/anam-org/clawd-face
- ✓ https://github.com/Michaelliv/claude-quest / ✓ https://github.com/FulAppiOS/Agent-Quest / ✓ https://github.com/bitqs/slime / ✓ https://github.com/geezerrrr/agent-town / ✓ https://github.com/ajbarea/kourai-khryseai
- ✓ https://github.com/crafter-station/petdex / ✓ https://github.com/BeiXiao/awesome-codex-pets
- ✓ https://github.com/ramarivera/coding-buddy / ✓ https://github.com/Ido-Levi/claude-code-tamagotchi / ✓ https://github.com/TeXmeijin/claude-code-mascot-statusline
- ✓ https://github.com/Rubiksman78/RenAI-Chat / ✓ https://github.com/ZzzhangLK/ai-galgame
- ✓ https://github.com/Open-LLM-VTuber/Open-LLM-VTuber / ✓ https://github.com/moeru-ai/airi / ✓ https://github.com/semperai/amica
- △ 日本語記事: https://zenn.dev/musoukun/articles/f71c3933ff3d68 / https://note.com/leiablack/n/ncbf5742bf152 / https://qiita.com/kazakago/items/287f91082ab59f581c09 / https://techblog.lclco.com/entry/2026/03/09/130422 / https://zenn.dev/shio_shoppaize/articles/5fee11d03a11a1 / https://dev.to/panzer_jack/copiwaifu-a-live2d-desktop-pet-that-syncs-with-claude-code-codex-copilot-and-more-19gp / https://anam.ai/blog/giving-claude-code-a-face

**GUI / ACP クライアント**
- ✓ https://github.com/getAsterisk/opcode / ✓ https://github.com/siteboon/claudecodeui / ✓ https://github.com/slopus/happy / ✓ https://github.com/sugyan/claude-code-webui / ✓ https://github.com/zhukunpenglinyutong/desktop-cc-gui
- ✓ https://github.com/wieslawsoltes/CodexGui / ✓ https://github.com/Shermanxwz/Codex-Official-App-Server-Web / ✓ https://github.com/jake-enastudio/codex-desktop
- ✓ https://github.com/formulahendry/vscode-acp / ✓ https://github.com/diodeme/Gold-Band / ✓ https://github.com/recailai/jockey / ✓ https://github.com/clark-labs-inc/clark-code / ✓ https://github.com/zvzuola/acp-components / ✓ https://github.com/openclaw/acpx / ✓ https://github.com/batrachianai/toad / ✓ https://github.com/xenodium/agent-shell / ✓ https://github.com/iOfficeAI/AionUi / ✓ https://github.com/Piebald-AI/gemini-cli-desktop
- ✓ https://github.com/BloopAI/vibe-kanban / ✓ https://github.com/generalaction/emdash / ✓ https://github.com/superset-sh/superset / ✓ https://github.com/nMaroulis/awesome-agent-client-protocol
- ✓ https://github.com/zed-industries/zed (docs/src/ai/external-agents.md, agent-panel.md, tool-permissions.md)

**Tauri / プロセス起動**
- ✓ https://github.com/tauri-apps/tauri-docs (v2 develop/sidecar.mdx, learn/sidecar-nodejs.mdx) / ✓ https://github.com/tauri-apps/plugins-workspace/issues/1632 / …/issues/2135 / ✓ https://github.com/tauri-apps/tauri/issues/4949 / …/issues/11513 / ✓ https://github.com/tauri-apps/fix-path-env-rs / ✓ https://github.com/rust-lang/rust/issues/94743 / ✓ https://nodejs.org/en/blog/vulnerability/april-2024-security-releases-2 / ✓ https://github.com/electron/electron/blob/main/docs/api/utility-process.md
