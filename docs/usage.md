---
description: "執行 /book-to-skill 的所有方式：單一檔案、資料夾、glob 模式或路徑列表，包含擷取模式、技能命名以及實際指令範例。"
seo_title: "使用方式 - 將書籍、資料夾或 Glob 轉換為 Agent Skill"
---

## 🚀 使用方式 (Usage)

```
/book-to-skill <檔案或資料夾路徑/glob模式>... [skill-名稱]
```

支援的文件格式：PDF, EPUB, DOCX, TXT, Markdown, reStructuredText, AsciiDoc, HTML, RTF, MOBI/AZW/AZW3。

**範例：**

```bash
# 將多個檔案合併處理成一個統一的 skill
/book-to-skill ~/papers/paper1.pdf ~/notes/export.txt unified-research

# 處理資料夾內所有支援的檔案
/book-to-skill ~/workspace/project-docs/ project-knowledge

# 處理符合 glob 模式的檔案
/book-to-skill "~/books/*.epub" my-library

# 將新素材更新/合併到現有的 skill 資料夾中
/book-to-skill ~/articles/new-paper.pdf ~/.claude/skills/project-knowledge
```

建立好 skill 後，就可以像使用其他的 agent skill 一樣呼叫它：

```bash
/designing-data-intensive-apps                  # 載入核心心智模型
/designing-data-intensive-apps replication      # 尋找並解釋某個主題
/designing-data-intensive-apps ch05             # 深入第 5 章
/designing-data-intensive-apps "what chapters do you have?" # 列出所有章節
```

在 GitHub Copilot CLI 中，寫入檔案後您可能需要執行 `/skills reload`，新的 skill 才會出現在 `/skills list` 中。而 Claude Code 和 Amp 則會在下一次會話時自動載入。

---

## 💬 其他人的應用方式

有人將一本關於開發者體驗 (DevEx) 的書轉化為適用於 300 多位工程師的問卷調查。也有因為掃描版 PDF 導致程式當機，進而促成了 [#130](https://github.com/virgiliojr94/book-to-skill/pull/130) 的提早中止修復 (early-abort fix) — 而回報這個問題的人甚至不是軟體工程師。

這些案例以及如何新增您的故事，都可以在 [book-to-skill-use-cases](https://github.com/virgiliojr94/book-to-skill-use-cases) 找到：將您的心得寫成 Gist 發布在自己的帳號上，然後在索引中加入一行即可，無需特定範本，也沒有 CI 檢查。歡迎分享您的轉換數據、應用成果，以及工具的不足之處。

---

## 📤 發布您生成的 skill (選用)

轉換完成後，轉換器會提示您將生成的 skill 發布為獨立的 GitHub 儲存庫。系統會另外詢問可見度 (visibility)，且 `gh repo create` 指令預設使用 **`--private`** (私有)；只有當您輸入精確的單字 `public` 時，才會建立公開的儲存庫。關於版權的描述並不算可見度的回答 — 說「這是公有領域 (public domain)」只是在描述書籍，並非儲存庫，系統依然會建立私有儲存庫。章節檔案是從來源素材合成出來的，因此由第三方版權書籍生成的 skill 必須保持私有 (請參閱 [版權與合理使用 (Copyright & fair use)](../README.md#%EF%B8%8F-copyright--fair-use))。發布後的 skill 只需一行指令即可安裝在任何 Agent Skills 宿主環境中：

```bash
npx skills add https://github.com/<you>/<your-book-slug> --skill <your-book-slug>
```

這需要安裝並驗證過 [`gh` CLI](https://cli.github.com)。生成的儲存庫會內建 README，且 skill 資料夾會成為 git 工作目錄，因此日後合併更新時可直接推送到同一個遠端儲存庫。

---

[← 返回 README](../README.md)
