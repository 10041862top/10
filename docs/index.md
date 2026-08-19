---
description: "將任何書籍、PDF 或 EPUB 轉換為 Claude Code、GitHub Copilot CLI 與 Amp 專用的結構化 agent skill — 按需載入框架與決策規則。"
seo_title: "book-to-skill - 將任何書籍轉化為 AI Agent Skill"
hide:
  - navigation
  - toc
---

<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "SoftwareApplication",
  "name": "book-to-skill",
  "applicationCategory": "DeveloperApplication",
  "operatingSystem": "Linux, macOS, Windows",
  "description": "將書籍和文件（PDF、EPUB、DOCX、HTML、Markdown、RTF、MOBI/AZW）轉換為 Claude Code、GitHub Copilot CLI 和 Amp 專用的結構化、按需載入的 agent skill。",
  "url": "https://booktoskill.is-a.dev/",
  "codeRepository": "https://github.com/virgiliojr94/book-to-skill",
  "license": "https://opensource.org/licenses/MIT",
  "offers": { "@type": "Offer", "price": "0", "priceCurrency": "USD" },
  "author": { "@type": "Person", "name": "virgiliojr94", "url": "https://github.com/virgiliojr94" }
}
</script>

# book-to-skill

<p style="font-size: 1.25rem; max-width: 42rem;">
將任何書籍或文件轉化為結構化、按需載入的 agent skill — 包含具名框架、決策規則與反模式 (anti-patterns)。<strong>擷取結構，而不只是摘要。</strong>
</p>

[開始使用](guide.md){ .md-button .md-button--primary }
[技能參考 (Skill reference)](skill-reference.md){ .md-button }
[GitHub](https://github.com/virgiliojr94/book-to-skill){ .md-button }

---

## 為什麼選擇 book-to-skill

<div class="grid cards" markdown>

-   :material-file-document-multiple:{ .lg .middle } __支援多種格式 (Multi-format)__

    ---

    支援 PDF, EPUB, DOCX, HTML, Markdown, RTF, MOBI/AZW (透過 Calibre)。內容擷取完全在本地端執行，並具備優雅的標準函式庫備援 — 無須上傳，沒有平台鎖定 (lock-in)。

-   :material-brain:{ .lg .middle } __注重結構，非僅摘要__

    ---

    提取具名框架、心智模型 (mental models)、決策規則和反模式 (anti-patterns) — 精準捕捉作者的方法論與專有名詞，而非單純的讀書心得。

-   :material-flash:{ .lg .middle } __按需載入章節__

    ---

    將檔案按章節分割，只有在相關主題被提及時才會載入。因此，一本 200 頁的書所消耗的 token 是根據問題來計算，而非總頁數。

-   :material-robot-happy:{ .lg .middle } __支援多 Agent__

    ---

    單一的 `SKILL.md` 即可透過開放的 Agent Skills 標準，在 Claude Code、GitHub Copilot CLI 以及 Amp 上執行。

</div>

## 安裝 (Install)

**作為一個 agent skill** (提供您在 Claude Code, Copilot CLI, Amp 中的 `/book-to-skill` 指令):

```bash
npx skills add virgiliojr94/book-to-skill
# 或者手動安裝：
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.claude/skills/book-to-skill
# 接著，在您的 agent 對話中：
/book-to-skill /path/to/book.pdf [skill-name]
```

**作為獨立的 CLI 工具** (僅限文字擷取器，選用):

```bash
# 尚未發布至 PyPI — pip 會直接從儲存庫拉取
pip install "book-to-skill[pdf,epub,docx] @ git+https://github.com/virgiliojr94/book-to-skill.git"
book-to-skill /path/to/book.pdf --mode text
```

## 了解更多

<div class="grid cards" markdown>

-   :material-sitemap:{ .lg .middle } __[系統架構 (Architecture)](architecture.md)__

    ---

    說明決定性擷取器 (deterministic extractor) 與規格驅動的生成器如何整合運作。

-   :material-speedometer:{ .lg .middle } __[效能 (Performance)](performance.md)__

    ---

    實測的探索迴圈稅 (Discovery Loop Tax) 與實際每次轉換的 token 成本。

-   :material-book-open-page-variant:{ .lg .middle } __[技能參考 (Skill Reference)](skill-reference.md)__

    ---

    完整的 `SKILL.md` 規格說明：每個步驟、深度預算以及品質規則。

-   :material-heart:{ .lg .middle } __[贊助 (Sponsor)](https://github.com/sponsors/virgiliojr94)__

    ---

    book-to-skill 是免費且基於 MIT 授權的開源專案。您的贊助將支持程式碼審查、版本發布與問題修復。

</div>
