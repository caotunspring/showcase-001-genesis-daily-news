# showcase-001-genesis-daily-news

**AIA × Claude Code · 工程力 receipt**
**Owner:** Mark Chen <mark@aipm.com.tw>
**Companion to:** [`showcase-003-daily-news`](https://github.com/aipmtw/showcase-003-daily-news) (the live product)

---

## 一句話

**這份 `build.md` 是把 003 daily-news 從 zero 蓋到 live 的 single source of truth。任何後續的 Claude Code session,clone 這份檔案、照做,40 分鐘內可以把同一個產品再蓋一次,deterministic、零驚喜。**

## 這個 repo 不是

- 不是另一個 daily-news 部署。實際 daily-news 跑在 `showcase-003-daily-news.vercel.app`,這 repo 沒有 `next dev`。
- 不是一份回顧文(後驗看心得)。是 forward-runnable 的工程手冊。
- 不是行銷文案。沒有「我們做了什麼」的炫技,只有「下一個工程師要怎麼做」的指令。

## 這個 repo 是

- 一份 17 節的 `build.md` —— 每一節都是 paste-and-run 的 shell + SQL + TypeScript。
- 一份 `timing/2026-04-26-rebuild.md` —— Mark 親手 walk 過一遍的時計與 checkpoint。
- 一份 `evidence/` —— 鏡頭檔、CLI 紀錄、`https://routines-fleet.aipm.com.tw` 的 routine_runs 表截圖,證明 003 是真跑出來的。

## 與其他 showcases 的關係

| | 在 AIA 投稿中扮演什麼 |
|---|---|
| **001** (this) | 工程力 receipt —— 「003 是被這份檔案蓋出來的,可重現」 |
| **002** (TBD) | (decision pending — 詳見 `mark-ai-talk/0426-001-002-direction.md`) |
| **003** | 主送件作品 —— Claude Code Routine + Vercel + Supabase 真的每天 08:00 跑 |
| **004** | 5/18 demo 用的「sibling proof」—— 同一份 build.md 換 domain,9 篇 supply-chain 新聞 |

## Inheritance / 多租戶 Supabase 架構

001 的 `build.md` 假設你會把新 showcase 部署到 `caotunspring's Org` 底下的 **`showcases-shared`** Supabase project(ref `vdjsjdkswhbtvpsmujlg`),透過 `project text not null` discriminator + 複合 unique constraint 達到多租戶。如果你要全新一個 Supabase project,§4 有 alternative 路徑。

## 怎麼用這份 repo

1. 開 `build.md`
2. 從 §0 開始逐節跑
3. 每跑完一節在 `timing/<date>-rebuild.md` 加一行 `Checkpoint NN — <description> — <elapsed>`
4. 最後的 checkpoint 應該是 `Checkpoint 17 — Live URL responding 200 with today's items — XX min total`

如果某節壞了 / 卡住,在 `evidence/` 開一個 `<date>-issue-<N>.md`,寫下發生什麼、怎麼解的。下一個 session 看得到。

## License

MIT — Mark Chen 2026.
