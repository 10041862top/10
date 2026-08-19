<p align="center">
  <img src="docs/assets/banner.webp" alt="Booklin, book-to-skill 的巫師，拿著一本翻開的書，書頁散落成火花並落入有序的網格中" width="100%">
</p>

<h1 align="center">book-to-skill</h1>

<p align="center">
  <a href="README.md"><strong>English</strong></a> ·
  <a href="README.ru.md">Русский</a>
</p>

<p align="center">
  <strong>將任何技術書籍、文件資料夾或資料來源集合，轉換為統一的 agent skill — 準備好在您使用 GitHub Copilot CLI、Amp 或 Claude Code 工作時，進行學習、參考和使用。</strong>
</p>

<p align="center">
  <a href="https://github.com/virgiliojr94/book-to-skill/releases"><img src="https://img.shields.io/github/v/release/virgiliojr94/book-to-skill?style=for-the-badge&color=blueviolet" alt="最新版本"></a>
  <img src="https://img.shields.io/badge/Agent_Skills-Open_Standard-blueviolet?style=for-the-badge" alt="Agent Skills 標準">
  <img src="https://img.shields.io/badge/PDF%20%E2%80%A2%20EPUB%20%E2%80%A2%20DOCX%20%E2%80%A2%20MD%20%E2%80%A2%20HTML%20%E2%80%A2%20RTF%20%E2%80%A2%20MOBI-supported-green?style=for-the-badge" alt="支援的格式">
  <img src="https://img.shields.io/badge/License-MIT-blue?style=for-the-badge" alt="MIT 授權">
  <a href="https://github.com/sponsors/virgiliojr94"><img src="https://img.shields.io/github/sponsors/virgiliojr94?style=for-the-badge&color=ea4aaa&logo=githubsponsors&logoColor=white" alt="贊助"></a>
</p>

<p align="center">
  <a href="https://trendshift.io/repositories/27038?utm_source=repository-badge&amp;utm_medium=badge&amp;utm_campaign=badge-repository-27038" target="_blank" rel="noopener noreferrer"><img src="https://trendshift.io/api/badge/repositories/27038" alt="virgiliojr94%2Fbook-to-skill | Trendshift" width="250" height="55"/></a>
  <a href="https://trendshift.io/repositories/27038?utm_source=trendshift-badge&amp;utm_medium=badge&amp;utm_campaign=badge-trendshift-27038" target="_blank" rel="noopener noreferrer"><img src="https://trendshift.io/api/badge/trendshift/repositories/27038/daily?language=Python" alt="virgiliojr94%2Fbook-to-skill | Trendshift (每日, Python)" width="250" height="55"/></a>
</p>

<p align="center">
  <a href="#-why">為什麼</a> ·
  <a href="#-what-it-generates">生成內容</a> ·
  <a href="#-beyond-books">超越書籍</a> ·
  <a href="docs/how-it-works.md">運作原理</a> ·
  <a href="docs/usage.md">使用方式</a> ·
  <a href="docs/install.md">安裝</a> ·
  <a href="docs/faq.md">常見問題</a> ·
  <a href="docs/performance.md">效能</a> ·
  <a href="docs/architecture.md">架構</a> ·
  <a href="CHANGELOG.md">更新日誌</a>
</p>

<p align="center">
  <strong>回答一個問題所需的 token 數量，比將整本書丟入 context 要少 24×–51×</strong>，這是根據真實書籍測量的結果（<a href="docs/performance.md#the-discovery-loop-tax">測量方法</a>）。
</p>

**運作原理（3 個步驟）：**

1. **指向** 它到一個檔案、資料夾或 glob — `/book-to-skill ./my-book.pdf`
2. **它提煉** 書籍成為一個 skill — 框架、決策規則、反模式（anti-patterns）以及每章獨立的檔案。是結構，而不是摘要。
3. **您的 agent 按需載入它** — 詢問 `/my-book replication`，它會閱讀正確的章節並從真實內容中回答，不會產生幻覺。

---

## 🤔 為什麼

<img align="right" width="200" src="docs/assets/booklin.png" alt="Booklin — book-to-skill 的吉祥物，一個拿著書的紫色巫師">

您買了一本很棒的技術書。您讀了一次。三個月後，您忘了第 7 章的存在。

常見的解決方案沒有幫助：
- 📄 "讓我直接搜尋 PDF" → 您得到的是一堆頁面列表，而不是答案
- 🧠 "我會詢問 agent 關於這本書的內容" → 它要麼產生幻覺，要麼說它沒有該內容
- 📝 "我會在閱讀時做筆記" → 最終您得到一個 200 行的文件，然後再也沒打開過

**book-to-skill 藉由將書籍轉換成結構化的 skill，讓您的 agent 按需載入，從而解決這個問題。**

安裝後，您只需輸入 `/your-book-slug replication`，agent 就會閱讀正確的章節並從實際內容中回答。沒有幻覺。不需要在 PDF 中翻找。這本書成為了您 workflow 的一部分。

適用於任何支援開放 [Agent Skills](https://github.com/agentskills/agentskills) 標準的宿主程式 — GitHub Copilot CLI、Amp 和 Claude Code 都能讀取相同的 `SKILL.md` 格式。

---

## 📦 生成內容

執行 `/book-to-skill your-book.pdf`（或資料夾、glob、檔案列表），會在您 agent 的 skills 目錄中建立一個完整的 skill（Copilot CLI 為 `~/.copilot/skills/<slug>/`，Amp 或跨 agent 為 `~/.agents/skills/<slug>/`，Claude Code 為 `~/.claude/skills/<slug>/`）：

| 檔案 | 目的 | 大小 |
|------|---------|------|
| `SKILL.md` | 核心心智模型 + 章節索引 | 約 4,000 個 token |
| `chapters/ch01-*.md` … | 每章一個檔案，按需載入 | 每個約 1,000 個 token |
| `glossary.md` | 每個關鍵術語，按字母排序並附帶章節參考 | 約 1,500 個 token |
| `patterns.md` | 所有技術、演算法和設計模式 | 約 2,000 個 token |
| `cheatsheet.md` | 決策表和快速參考規則 | 約 1,000 個 token |

**章節檔案是按需載入的** — 除非您詢問該主題，否則它們不會計入 skill 的預算中。

---

## 🏢 超越書籍

名字裡雖然有 "book"（書籍），但輸入可以是任何結構化的散文。同樣的擷取方式也適用於您擁有且經常重複閱讀的知識：

- **內部文件** — 架構決策記錄、runbook、入職指南。將整個 `docs/` 資料夾摺疊成一個 skill，並在您寫程式時向它提問。
- **品牌與設計系統** — 語音指南、語氣文件、元件原則。將品牌手冊轉換成您的團隊可以查詢的 skill，而不是去略讀一份 60 頁的 PDF。
- **研究集群** — 一疊論文加上您自己的筆記，合併成一個統一的 skill，並隨著新資料的到來而更新（請參閱 [更新 / 合併](#-usage)）。
- **規格與標準** — RFC、API 合約，以及您會參考但永遠背不下來的合規文件。

如果您經常重新打開某份文件到希望自己能把它背下來的程度，那它就是一個合適的候選者。

---


## 🧾 發現迴圈稅 (The Discovery Loop Tax)

一個閱讀 PDF 的 agent 不只是閱讀 — 它還會*導覽*：它重新獲取目錄，往回找，然後在每一次對話回合中重新處理這一切。book-to-skill 在轉換時**一次性**支付了這個結構化成本，所以查詢的成本會與答案成正比 — 比起把書丟進 context 中，使用的 **token 少了 24×–51×**，這是根據真實書籍測量的結果。

📊 **完整的測量方法、數據和各書籍的表格 → [docs/performance.md](docs/performance.md#the-discovery-loop-tax)**

---

## ⚙️ 運作原理

分為兩半：一個決定性的 Python **提取器**（文件 → 乾淨的文字 + metadata），和一個由規格驅動的 **生成器**（您的 agent 遵循 `SKILL.md` 將其轉換為結構化的 skill）。按需載入的章節檔案可保持載入的 skill 體積小巧。

🔧 **完整演練（步驟 0–10、提取模式、token 預算） → [docs/how-it-works.md](docs/how-it-works.md)**

---

## 🚀 使用方式

`/book-to-skill <path|folder|glob> [skill-name]` — 加上僅分析（analyze-only）、從分析生成（generate-from-analysis）以及更新/合併（update/fold-in）模式。轉換後，轉換器可以將該 skill 發佈到 GitHub（預設為私有），這樣任何宿主都可以使用 `npx skills add` 來安裝它。

▶️ **所有模式和範例 → [docs/usage.md](docs/usage.md)**

💬 **實際應用 → [使用案例](https://github.com/virgiliojr94/book-to-skill-use-cases)** — 一本 DevEx 的書變成了 300 多位工程師的問卷調查；一個掃描版 PDF 處理停滯成為了 [#130](https://github.com/virgiliojr94/book-to-skill/pull/130)。加入您的案例：帳號存在於您自己的 Gist 中，索引只需要一行 PR。

---

## 📥 安裝

```bash
# 單一指令，支援任何宿主 — 透過跨 agent 的 skills CLI：
npx skills add virgiliojr94/book-to-skill

# 或者手動安裝 — 複製 (clone) 到您的 skills 資料夾中（註冊 /book-to-skill）：
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.claude/skills/book-to-skill
# (Copilot CLI: ~/.copilot/skills/ · Amp/cross-agent: ~/.agents/skills/)
```

📥 **所有宿主、可選提取器與獨立的 CLI → [docs/install.md](docs/install.md)**

---

## ❓ 常見問題

常見問題 — "為什麼不直接丟進 PDF？"、成本、隱私、非書籍輸入、多檔案書籍。

❓ **解答 → [docs/faq.md](docs/faq.md)**

---

<details>
<summary>🔧 <strong>系統需求</strong></summary>


提取器會針對每種格式依序嘗試工具，並使用第一個可用的工具。如果沒有安裝任何東西，它會告訴您該執行哪個指令。純文字、Markdown、reStructuredText 和 AsciiDoc 不需要額外的依賴套件。

> **用一個指令檢查您的設定：** `python3 scripts/extract.py --check` 會印出每個格式安裝了哪些提取器，以及安裝任何缺少的工具所需的確切指令 — 不需要提供檔案。

**PDF — 依書籍類型選擇：**

| 書籍類型 | 工具 | 安裝指令 | 速度 |
|-----------|------|---------|-------|
| 偏重文字（散文，少量表格） | `pdftotext` (poppler) | `sudo apt install poppler-utils` | ⚡ 瞬間 |
| 偏重文字後備 | `pypdf` | `pip3 install pypdf` | ⚡ 瞬間 |
| 偏重文字後備 | `pdfminer.six` | `pip3 install pdfminer.six` | ⚡ 瞬間 |
| **技術類（程式碼、表格、公式）** | **`docling`** | `pip3 install docling` | 每頁約 1.5 秒 |

> 在提取開始之前，skill 會詢問您這本書是**技術類**還是**偏重文字**，並自動選擇合適的工具。Docling 會保留 markdown 表格和程式碼區塊；pdftotext 對於純散文書籍來說速度更快。

> **掃描的 PDF 需要先進行 OCR。** 如果 PDF 只是一頁一頁的圖片且沒有文字層 — 例如拍攝或掃描的書 — 這些工具就沒有東西可以提取。提取器會檢查前幾頁並立即停止並附上說明，而不是處理整本書卻產生一個空的 skill。請您自行執行 OCR，然後轉換結果：
>
> ```bash
> ocrmypdf input.pdf output.pdf
> ```

**EPUB:**

| 工具 | 安裝指令 | 品質 |
|------|---------|---------|
| `ebooklib` + `beautifulsoup4` | `pip3 install ebooklib beautifulsoup4` | ⭐⭐⭐ 最佳 |
| 標準函式庫 `zipfile` | 內建 — 無需安裝 | ⭐⭐ 隨時可用 |

**其他格式:**

| 格式 | 工具 | 安裝指令 |
|--------|------|---------|
| DOCX | `python-docx` (後備：標準函式庫 ZIP/XML) | `pip3 install python-docx` |
| HTML | `beautifulsoup4` (後備：標準函式庫 `html.parser`) | `pip3 install beautifulsoup4` |
| RTF | `striprtf` (後備：正規表達式) | `pip3 install striprtf` |
| MOBI / AZW / AZW3 | Calibre `ebook-convert` (外部應用程式，非 pip) | https://calibre-ebook.com/download |
| TXT / Markdown / reStructuredText / AsciiDoc | 內建 | — |

---


</details>

<details>
<summary>📁 <strong>存放庫結構</strong></summary>


```
book-to-skill/
├── SKILL.md              # Skill 定義 + 逐步指示（生成器規格）
├── scripts/
│   ├── extract.py        # 輕量級的入口包裝器
│   └── extractor/        # 模組化提取套件
│       ├── config.py     # 擴展名、路徑、依賴常數
│       ├── dependencies.py  # 探測可選依賴 + --check
│       ├── exceptions.py # ExtractionError（每個來源的錯誤，支援批次處理）
│       ├── utils.py      # CLI 解析、多來源解析、章節偵測、執行器
│       └── parsers/      # 針對特定格式的解析器（pdf, epub, docx, html, rtf, calibre, text）
├── tools/
│   ├── discovery_tax.py  # 測量 token 成本 vs context 傾倒 / 發現迴圈
│   └── validate_skill.py # 根據宿主規則檢查生成的 SKILL.md（--lens claude|copilot|amp）
├── tests/                # pytest 測試套件（提取、偵測、發現稅）
├── docs/
│   ├── performance.md    # 測量的效能基準、發現稅、成本
│   └── architecture.md   # pipeline + 元件圖
├── CHANGELOG.md          # 發佈歷史（語義化版本）
├── CONTRIBUTING.md       # 開發設定、PR 慣例、發佈流程
├── SECURITY.md           # 漏洞回報
└── README.md             # 本檔案
```

---


</details>

---
## ⚖️ 版權與合理使用

book-to-skill **不隨附任何書籍內容** — 連一頁都沒有。它是一個轉換器，您可以將其指向您已擁有的檔案。

- **處理過程在本地端進行。** 提取和分析在您的機器上執行。您的檔案絕不會被此工具上傳。（如果您的 agent 所在的模型在雲端執行，那麼您提供給它的文字會遵循該提供商的一般資料條款 — 就像任何 prompt 一樣。）
- **您使用自己的副本。** 帶上一本您買的書、您公司擁有的文件，或您有權閱讀的論文。
- **輸出的結果是您的筆記。** 生成的 skill 是一種結構化、合成的衍生作品 — 框架名稱、定義、重點摘要 — 而不是內文的複製品。該 skill 明確規定絕不複製原始段落（請參閱 Quality Rule #7）。請將其視為手寫的學習筆記：屬於您的，僅供個人使用。
- **請勿重新散佈。** 發佈或分享受版權保護作品的生成 skill 可能會侵犯版權所有者的權利。請將第三方書籍的 skill 保持私有。內部文件、您自己的寫作，以及開放授權的資料，可以在其授權條款的範圍內進行分享。

如有疑問，請遵循來源文件的授權或條款。這個專案是一個工具；如何使用由您決定。

---

## 💖 贊助者

<img align="right" width="150" src="docs/assets/booklin-celebrating.png" alt="Booklin 慶祝中">

book-to-skill 是免費的且採用 MIT 授權，由個人時間維護。如果它為您節省了 token 或學習時間，請考慮贊助其維護費用：PR 審查、多國語言修復、版本發佈與文件。

**[成為贊助者 → github.com/sponsors/virgiliojr94](https://github.com/sponsors/virgiliojr94)**

每一位贊助者都會列在 [BACKERS.md](BACKERS.md) 中。感謝您讓開源、隱私優先的工具能繼續存活。 ✨

## 授權條款

MIT — 適用於此存放庫中的轉換器（程式碼 + skill 定義），**不適用於**您用它處理的任何書籍或文件。

## Star History

<a href="https://www.star-history.com/?repos=virgiliojr94%2Fbook-to-skill&type=date&legend=top-left">
 <picture>
   <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=virgiliojr94/book-to-skill&type=date&theme=dark&legend=top-left" />
   <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=virgiliojr94/book-to-skill&type=date&legend=top-left" />
   <img alt="Star History Chart" src="https://api.star-history.com/chart?repos=virgiliojr94/book-to-skill&type=date&legend=top-left" />
 </picture>
</a>
