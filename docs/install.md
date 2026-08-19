---
description: "將 book-to-skill 安裝為 Claude Code、GitHub Copilot CLI、Amp 與 Codex 的 agent skill，或是作為獨立的 pip CLI 工具。涵蓋所有宿主環境路徑與可選的擴充擷取器。"
seo_title: "安裝 book-to-skill - 支援 Claude Code, Copilot CLI, Amp 或 pip"
---

## 📥 安裝 (Install)

> **有兩種使用方式，請勿混淆：**
> - **作為一個 agent skill** (在 Claude Code, Copilot CLI, Amp 或 Codex 中使用的 `/book-to-skill` 指令) → **將儲存庫 `git clone` 到您的 skills 資料夾中** (詳見下方)。這樣您才能獲得斜線指令以及完整的書籍轉換流程。
> - **作為獨立的 CLI 工具** (僅限文字擷取器) → 透過 `pip install` 從儲存庫安裝，然後執行 `book-to-skill --help`。這**不會**註冊 agent skill；它只會安裝擷取引擎。請參閱 [CLI 章節](#standalone-cli-pip)。

此 skill 遵循開放的 [Agent Skills](https://github.com/agentskills/agentskills) 標準，因此只需一次安裝，即可在任何相容的宿主環境中執行。

**一個指令，支援所有環境** — 使用 [`skills` CLI](https://skills.sh) 工具會自動解析儲存庫、偵測根目錄的 `SKILL.md`，並將完整的 skill（包含 `scripts/extract.py` 與 `tools/`）安裝到您所選環境的 skills 資料夾中：

```bash
npx skills add virgiliojr94/book-to-skill
```

比較偏好手動安裝？下方列出的每一個針對特定宿主環境的 `git clone` 路徑，執行效果都完全相同。

**GitHub Copilot CLI** (個人 skill):

```bash
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.copilot/skills/book-to-skill
# 接著，在 `copilot` 會話中：
/skills reload
/skills info book-to-skill
```

或是使用 Copilot CLI, Amp 和 Codex 都能自動識別的跨 agent 共用路徑：

```bash
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.agents/skills/book-to-skill
```

**OpenAI Codex** 會讀取 `~/.agents/skills` 並且支援 symbolic links (符號連結)，所以上方的 clone 指令就足夠了。您也可以在本地端結帳 (checkout) 後，用語法連結的方式取代直接複製：

```bash
ln -s /path/to/book-to-skill ~/.agents/skills/book-to-skill
```

**Claude Code**:

將以下指令複製到您的 Claude Code 會話中：

```
Install book-to-skill: https://raw.githubusercontent.com/virgiliojr94/book-to-skill/master/SKILL.md
```

或者使用標準的 `git clone` 手動安裝 (可確保模組化的引擎檔案被正確抓取)：

```bash
git clone https://github.com/virgiliojr94/book-to-skill.git ~/.claude/skills/book-to-skill
```

然後在任何 agent 會話中執行：

```bash
/book-to-skill ~/path/to/your-book.pdf
# 或者
/book-to-skill ~/path/to/your-book.epub
```

### 獨立 CLI 工具 (pip)

透過 `pip` 安裝 CLI 是**獨立且選用的**方式。它僅會將文字擷取引擎安裝為命令列工具，供腳本編寫或使用可選的擷取器；它**不會**註冊 `/book-to-skill` agent skill (請使用上方的 `git clone` 來註冊)。

`book-to-skill` 尚未發布至 PyPI，因此 `pip` 會直接從儲存庫拉取套件：

```bash
pip install "book-to-skill[pdf,epub,docx] @ git+https://github.com/virgiliojr94/book-to-skill.git"
book-to-skill ~/path/to/book.pdf --mode text  # 或者執行: python -m book_to_skill ...
book-to-skill --check                          # 回報目前已安裝哪些擷取器
```

> **`[html]` 模組會比其他模組來得龐大。** 由於它會拉取 `trafilatura`，這將引入一整套 HTML 處理堆疊 (包含 lxml、日期解析器、時區資料庫、URL 分類器 — 總共 17 個套件)，以執行真正的主內容/樣板 (boilerplate) 偵測，而不僅僅是移除 `<script>`/`<style>`。在資源受限的機器上安裝前請特別留意 — 備用的 `bs4` 解析器 (不需額外安裝 `[html]`) 依然可以運作，只是無法移除樣板內容。

---


---

[← 返回 README](../README.md)
