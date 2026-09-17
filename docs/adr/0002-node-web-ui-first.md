# ADR-0002: Node + ブラウザ UI から始め、デスクトップシェルは後回しにする

- 日付: 2026-09-17(2026-09-16 の「Tauri 2 + React + TypeScript」案を置き換え)
- 状態: 採用(MVP)

## 背景

当初は Tauri 2 を第一候補にしていたが、プロジェクトの目的を「自分用、まず動かして確かめる」に絞り直した。
調査で分かった事実:
- ACP Agent(claude-agent-acp、codex-acp)は npm 配布で Node ≥22 が必要。どの構成でも Node は要る
- Tauri では Rust 側に spawn / stdio / kill を書く必要があり、Windows の `npx.cmd`、コンソールウィンドウ、kill 伝播、PATH の課題を Rust で扱うことになる(acp-ui が解決済みではあるが、コピーと理解のコストはかかる)
- Node なら `@agentclientprotocol/sdk` を直接使え、spawn も Node の `child_process` で完結する
- 既存の ACP クライアントは Tauri と Electron に二分され、公式アプリは Electron。どちらでも後から包める

## 決定

| 層 | 選択 |
|---|---|
| Agent 接続 | **Node + TypeScript**。`@agentclientprotocol/sdk` の `ClientSideConnection`、`child_process.spawn` |
| UI | **ブラウザ(Vite + React + TypeScript)**。Node ホストと localhost の WebSocket で通信 |
| デスクトップ化 | **後回し**。必要になったら Tauri(Node ホストを同梱)か Electron(main に移す)。UI と接続コードは変えない |
| 状態管理 | zustand |
| Markdown | react-markdown + remark-gfm + rehype-sanitize |
| diff | 既存ライブラリ(react-diff-viewer-continued 等)。自前実装しない |

## 理由

1. **最短で「サブスクのまま ACP が通るか」を確かめられる**。Step 0 は Node のスクリプト 1 本で済み、UI フレームワークの判断すら要らない
2. **Windows の起動問題を JavaScript 側だけで扱える**(`shell: true` か実体パス直起動)
3. **ブラウザ UI は演出を作りやすい**。ノベル UI は CSS / DOM で足り、DevTools で調整できる
4. **後戻りが無い**。Tauri / Electron のどちらで包んでも、Node ホストと React UI はそのまま

## 反対意見と扱い

- **最初からデスクトップアプリの見た目が欲しい**
  → ブラウザをアプリモード(`--app=`)で開けばウィンドウ 1 枚に見える。本格的に包むのは MVP 後
- **Tauri なら配布サイズが小さい**
  → 配布しない。将来配布するならそのとき選ぶ
- **React でなく Svelte / Vue でもよい**
  → 参考実装(acp-components、AvatarCode、Gold Band)が React で部品を流用しやすいので React。強い理由ではない

## 結果

- `scripts/acp-smoke.ts` → `host/` → `web/` の順(`docs/mvp.md`)
- デスクトップ化の判断は MVP 後に別 ADR
