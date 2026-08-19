---
description: "book-to-skill 的架構設計：一個決定性的 Python 擷取器加上一個規格驅動的 agent 生成器。包含處理管線、元件對應表，以及兩者背後的設計取捨。"
seo_title: "系統架構 - book-to-skill 如何進行擷取與生成"
---

# 系統架構 (Architecture)

book-to-skill 分為兩大部分：一個**決定性的擷取器 (deterministic extractor)** (使用 Python) 和一個**規格驅動的生成器 (spec-driven generator)** (遵循 `SKILL.md` 指示的 agent)。擷取器將任何文件轉換為乾淨的文字與詮釋資料 (metadata)；agent 則將這些內容轉換為結構化的 skill。

```
            ┌─────────────────────────── 擷取器 (EXTRACTOR, Python, 決定性) ──┐
  文件       │  scripts/extract.py (墊片)  →  book_to_skill/                    │
 (pdf/epub/ │    ├─ cli.py · utils.py   CLI 解析 · 多來源處理 · 執行器          │
  docx/...) │    ├─ config.py           支援的副檔名 · 路徑 · 依賴套件          │
     │      │    ├─ dependencies.py     可選依賴探測 · --check 報告             │
     ▼      │    ├─ sanitize.py         清除不可見/零寬度的 Unicode 字元        │
 ───────────│    └─ parsers/            pdf · epub · docx · html · rtf ·      │
            │                             calibre · text (最佳工具與備案)       │
            │  輸出 → <tempdir>/book_skill_work/                              │
            │    full_text.txt   (合併所有來源，並帶有來源標記)                   │
            │    metadata.json   (頁數、字數、tokens、章節、目錄)                 │
            └────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
            ┌─────────────────────────── 生成器 (GENERATOR, agent, 遵循 SKILL.md) ┐
            │  步驟 1.5  詢問內容類型 → BOOK_TYPE (技術 technical | 純文字 text)  │
            │  步驟 2/2.5 擷取 · 成本估算 · 確認                                  │
            │  步驟 2.6  針對大型書籍的 REPL 探測 (使用 grep/sed，避免完整重讀)   │
            │  步驟 3    分析結構 (標題、作者、章節、目錄)                        │
            │  步驟 4    目的 → DEPTH (參考 reference | 學習 study)               │
            │  步驟 7    生成各章節摘要 (預算 = BOOK_TYPE × DEPTH)                │
            │  步驟 8    術語表 · 模式 · 備忘錄 (決策層)                          │
            │  步驟 9/9.5 SKILL.md 主檔 + 索引                                  │
            └────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
                <SKILLS_HOME>/<slug>/  ← 根據不同宿主環境選擇：
                  ~/.copilot/skills/   GitHub Copilot CLI
                  ~/.agents/skills/    Copilot CLI 或 Amp (跨 agent)
                  ~/.claude/skills/    Claude Code
                  .github|.claude|.agents/skills/  專案本地端
                  SKILL.md         核心框架 + 章節與主題索引 (約 4K)
                  chapters/*.md    按需載入，僅在需要時讀取
                  glossary.md      術語表
                  patterns.md      技術/模式
                  cheatsheet.md    決策規則 / 決策樹 / 權衡取捨 / 辨識特徵 (tells)
```

## 設計原則

1. **擷取結構，而非摘要** — 具名框架、決策規則、反模式；絕對不直接輸出原始段落。
2. **編譯期處理勝過執行期** — 一次性支付導航與結構化的成本；在查詢時只載入相關章節。請參閱 [效能與成本 (Performance & Cost)](performance.md)。
3. **按需載入章節** — 保持 `SKILL.md` 的輕量；章節檔案只有在被讀取時才會消耗 tokens。
4. **前置 `SKILL.md` 核心內容** — 最重要的內容放在最前面 (若內容過長，系統會從結尾開始截斷)。
5. **優雅降級 (Graceful degradation)** — 每種格式都有標準函式庫的備案；如果某個來源處理失敗會跳過，而不會導致整個流程中斷。

## 關鍵元件

| 路徑 | 職責 |
|------|----------------|
| `scripts/extract.py` | 輕量級的入口墊片 (shim) → 指向 `book_to_skill.cli` (保留此檔案以確保舊版指令仍能運作) |
| `book_to_skill/cli.py`, `utils.py` | CLI 參數解析、多來源解析、章節與目錄偵測、執行器 |
| `book_to_skill/parsers/` | 每種格式一個模組 (`pdf`, `epub`, `docx`, `html`, `rtf`, `calibre`, `text`) |
| `book_to_skill/config.py` | 支援的副檔名、輸出路徑、依賴套件對應表 |
| `book_to_skill/dependencies.py` | 可選依賴探測機制 + `--check` 功能 |
| `book_to_skill/sanitize.py` | 清除擷取文字中的零寬度/Unicode tag block 字元 (請參閱安全防護章節) |
| `tools/discovery_tax.py` | 測量 context-dump (直接丟入內容) 相比於 discovery loop (探索迴圈) 的 token 成本差異 |
| `tools/validate_skill.py` | 根據宿主規則檢查生成的 SKILL.md (`--lens claude\|copilot\|amp`) |
| `tools/scan_generated_skill.py` | 對生成的 skill 進行建議性的 prompt-injection 掃描 (請參閱安全防護章節) |
| `SKILL.md` | 生成器的規格文件 (步驟 0–10 + 合併工作流程) |

## 安全防護 (Security)

不可信任的文件會流入 agent 的 context，接著變成生成的 skill，之後又會載入到其他 agents 中 — 這是一條「文件→context」的供應鏈。因此安全防護是分層進行的：

- **擷取清理 (Extraction sanitization)** (`book_to_skill/sanitize.py`) — 在計算指標或寫入 `full_text.txt` 之前，會先從每個解析器的輸出中清除零寬度字元 (`U+200B/200C/200D/2060/FEFF`) 和 Unicode tag block (`U+E0000–E007F`)，確保潛藏在文件中看不見的指令永遠不會進入 agent。它會回報被移除的字元數量；若來源清空後沒有可見內容，則會直接拒絕該來源。
- **DOCX XXE / Billion-Laughs 防護** (`parsers/docx.py`) — 在解析之前，拒絕任何宣告了 DTD 或實體 (entities) 的 XML 檔案。
- **子行程參數注入防禦 (Subprocess argument-injection)** — 檔案路徑在傳遞給 `pdftotext` / `pdfinfo` / `ebook-convert` 之前會轉換為絕對路徑，確保以 `-` 開頭的檔名不會被誤認為指令參數。
- **生成的 Skill 掃描** (`tools/scan_generated_skill.py`) — 這是在生成器中的建議步驟 (步驟 9.5)，用於標記生成的 `SKILL.md`、`chapters/*.md`、`glossary.md`、`patterns.md` 和 `cheatsheet.md` 中潛在的覆蓋指令短語、模型控制標籤、殘留的隱形 Unicode、權限擴大的前置設定 (frontmatter) 以及類似資料外洩的內容。掃描結果只會指出違反的規則名稱與檔案位置 — 絕對不會印出比對到的具體文字。
- **CI** — 包含 CodeQL、Bandit (擋下 HIGH 級別漏洞)、Zizmor，以及 PR 上的依賴套件 CVE 審查。

## 擴充 (Extending)

- **新增格式** → 新增 `book_to_skill/parsers/<fmt>.py`，在 `config.py` 中註冊其副檔名，在 `dependencies.py` 設定依賴探測，並在 `utils.extract_single_file` 加入分支邏輯。
- **新增生成行為** → 修改 `SKILL.md` 中相關的步驟；請保持精簡，並提供證據來支持您的變更 (請參閱 CONTRIBUTING.md)。
