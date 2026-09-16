# ADR-0002: Tauri 2 + React + TypeScript を採用し、Rust 側は最小に留める

- 日付: 2026-09-16
- 状態: 採用(MVP)。Step 0 のスパイクで問題があれば再検討

## 背景

デスクトップアプリの候補: Tauri 2 / Electron / Web + ローカル Node サーバ。UI フレームワーク候補: React / Vue / Svelte。
比較軸: Windows での扱いやすさ、Agent プロセス起動、stdio 通信、ファイルアクセス、セキュリティ、配布サイズ、開発容易性、Agent の差し替えやすさ。

調査で分かった事実(`docs/research.md` §3、§7):
- ACP Agent(claude-agent-acp、codex-acp)は npm 配布で **Node ≥22 が必要**。どの構成でもユーザー環境の Node か Node 同梱が要る
- 既存の ACP デスクトップクライアントは Tauri 2(acp-ui、Gold Band、Jockey、Clark Code、gemini-cli-desktop)と Electron(AionUi、Superset、Emdash)に二分。公式アプリ(Claude Desktop、Codex app)は Electron と報道
- Tauri で npx を起動する際の Windows 課題(`.cmd`、コンソールウィンドウ、kill、PATH)は acp-ui の `agent.rs` が解決済み(MIT)
- Electron は main プロセスで Agent SDK / ACP SDK を直接 import でき、Node sidecar が不要。配布サイズは 80〜200MB(Tauri は 10MB 前後)

## 決定

| 層 | 選択 |
|---|---|
| シェル | **Tauri 2** |
| UI | **React + TypeScript**(Vite) |
| プロトコル処理 | **TypeScript(WebView 側)**。`@agentclientprotocol/sdk` の `ClientSideConnection` を Tauri event 経由の Web Streams に接続 |
| Rust 側 | **プロセス起動・stdin 書込・stdout 行読取・kill・設定ファイル・フォルダピッカーのみ**(acp-ui の `agent.rs` を土台に) |
| 状態管理 | zustand(小さく、React 非依存の store が作れる) |
| Markdown | react-markdown + remark-gfm + rehype-sanitize |
| diff | 既存ライブラリを使う(候補: react-diff-viewer-continued、または acp-components の DiffViewer)。自前実装しない |

## 理由

1. **配布サイズと Windows 優先**。Tauri は WebView2 を使うため Windows で軽く、インストーラが小さい。Node と Agent は同梱しない方針なので、Electron の「Node 同梱」の利点が薄い
2. **既存の解決策をそのまま使える**。Windows のプロセス起動は acp-ui からコピーできる。TS 側の ACP クライアントは公式 SDK
3. **プロトコル処理を TS に置くことで、シェルを Electron に替えても TS 側が無傷**。Rust 側は数百行に収まる
4. **React**: 参考実装(acp-components、AvatarCode、Gold Band、Clark Code)が React で、部品を流用しやすい。ノベル UI は CSS / DOM で十分(PixiJS 不要)。Svelte / Vue でも作れるが、流用可能な資産の量で React を選ぶ
5. **Rust で ACP クライアントを書かない**。`agent-client-protocol` crate(2.1.0)は使えるが、UI と状態が TS 側にある以上、プロトコルも TS 側に置いた方が境界が 1 つ減る

## 反対意見と扱い

- **Electron なら Agent SDK 直結(ADR-0001 の B)が sidecar なしで済む**
  → MVP は ACP のみ。固有バックエンドが必要になった時点で、Node sidecar(`bun build --compile` 等で単一バイナリ化)か Electron 移行かを再判断する。TS 側の資産は移行しても使える
- **Tauri の WebView は OS 依存でレンダリング差が出る**
  → MVP は Windows(WebView2 = Chromium)に絞る。macOS の WebKit 差は後で対応
- **`cmd /C` 経由の kill 問題(tauri #4949)**
  → `taskkill /T /F` で子プロセスごと終了。設定で node の実体パスを指定できるようにし、`node <script>` の直起動も可能にする
- **Node をユーザーに要求する**
  → 対象ユーザーは Claude Code / Codex を既に使っている開発者で、Node は入っている前提。設定画面で node / npx のパスを上書きできるようにする

## 結果

- Step 0 のスパイクで「Windows で claude-agent-acp を起動し、initialize → session/new → prompt が通り、kill で子プロセスが残らない」ことを確認してから先へ進む
- スパイクで解決不能な問題が出た場合の代替は **Electron + 同じ TS コード**。Web + ローカル Node サーバは「デスクトップアプリ」という要件から外れるため最後の手段
