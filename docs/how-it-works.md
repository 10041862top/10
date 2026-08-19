---
description: "book-to-skill 的完整運作流程，從步驟 0-10：文件擷取、章節偵測、框架挖掘、深度預算，以及 agent skill 是如何被組合出來的。"
seo_title: "book-to-skill 運作原理 - 從書籍到 Agent Skill，逐步解析"
---

<img align="right" width="180" src="assets/booklin-casting.png" alt="Booklin 施放 book-to-skill 魔法">

## ⚙️ 運作原理 (How it works)

```
單一檔案 · 一個資料夾 · glob 模式 · 路徑列表
     │
     ▼
步驟 1.5 — "是技術書籍還是純文字書籍？"
     │
     ├── 技術類 (technical) → Docling  (將表格與程式碼區塊轉為 markdown，每頁約 1.5 秒)
     └── 純文字 (text)      → pdftotext → pypdf → pdfminer  (瞬間完成)
     │
     ▼
scripts/extract.py <paths…> --mode <technical|text>
  根據來源：PDF → pdftotext/Docling · EPUB → ebooklib → 標準函式庫 zipfile · DOCX/HTML/RTF/…
  (遇到損壞的來源會跳過並發出警告；其餘的會繼續處理)
     │
     ├── /tmp/book_skill_work/full_text.txt   (合併所有來源，並帶有來源標記)
     └── /tmp/book_skill_work/metadata.json   (彙總統計資料 + 個別來源陣列)
               │
               ▼
          Claude 分析結構
          (標題、作者、章節、目錄 — 涵蓋所有來源)
          ── 或者，如果目標是現有的 skill：則將新內容合併 (模式 4)
               │
               ▼
          生成各章節摘要 (每個約 800–1,200 tokens)
          technical 模式 → 包含「程式碼範例 (Code Examples)」與「參考表格 (Reference Tables)」區塊
          生成術語表 (glossary)、模式 (patterns)、備忘錄 (cheatsheet)
          生成包含核心心智模型的 SKILL.md 主檔
               │
               ▼
          Skill 會寫入以下其中一個位置：
            ~/.copilot/skills/<slug>/   (GitHub Copilot CLI)
            ~/.agents/skills/<slug>/    (Copilot CLI 或 Amp，跨 agent 共用)
            ~/.claude/skills/<slug>/    (Claude Code)
          /tmp/book_skill_work/         🗑️  清理暫存檔
```

**擷取效能基準測試 (Extraction benchmark)** (103 頁技術書籍，僅使用 CPU)：

| 擷取方式 | 耗時 | Tokens | 表格數量 | 程式碼區塊 |
|--------|------|--------|--------|-------------|
| pdftotext | 0.1s | 27K | 0 | 0 |
| Docling | 164s | 27K (+1.2%) | 48 | 36 |

**實際轉換數據 (Real conversions)** (測量項目：頁數、擷取出的 tokens、自動偵測章節數，
在 Claude Sonnet 4.5 每百萬 token \$3/\$15 的估計單次轉換成本)：

| 書籍 | 格式 | 頁數 | Tokens | 章節數 | 預估成本 |
|------|--------|------:|-------:|---------:|------:|
| Think Python 2 | PDF | 244 | 119K | 19 | \$0.88 |
| Working Backwards | PDF | 371 | 175K | 10 | \$0.96 |
| Pro Git | PDF | 501 | 229K | — † | \$1.23 |
| Moby-Dick | EPUB | — | 301K | — † | \$1.42 |

† 章節自動偵測需要明確的 `Chapter N` / `Capítulo N` 標題。《Pro Git》使用段落標題，而《白鯨記 (Moby-Dick)》使用章節*名稱* / 羅馬數字，因此兩者皆無法自動分段 — 但擷取與轉換仍然可以運作，只需手動指定段落即可。一個完整的 skill 轉換成本大約是 **每本書 \$1**；這遠比每次 agent 對話時重新閱讀整份 PDF 便宜得多。

<details>
<summary>設計原則 (點擊展開)</summary>

1. **資訊密度重於完整性 (Density over completeness)** — 一份 1,000-token 的摘要勝過 10,000-token 的摘錄。
2. **實踐者口吻 (Practitioner voice)** — 用「在遇到 Y 情況時使用 X」，而不是「這本書解釋了 X」。
3. **前置核心 SKILL.md (Front-loaded SKILL.md)** — 濃縮資訊確保前 ~5,000 tokens 包含最重要的內容。
4. **按需載入章節 (On-demand chapters)** — 主題索引會告訴 Claude 該讀哪個檔案；只有在需要時才載入特定章節。
5. **不使用原始文字 (Never raw text)** — 必須對來源進行綜合、摘要，並從中萃取出有用的訊號。

</details>

---


---

[← 返回 README](../README.md)
