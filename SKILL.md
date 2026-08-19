---
name: book-to-skill
description: "將書籍與文件（PDF、EPUB、DOCX、HTML、Markdown、純文字、RTF、使用 Calibre 的 MOBI/AZW）轉換為結構化的 agent skills，提取框架 (frameworks)、心智模型 (mental models)、原則、技巧以及反模式 (anti-patterns)。當使用者想要透過 GitHub Copilot CLI、Amp 或 Claude Code 學習文件，在工作時套用作者的框架，或從檔案建立可重複使用的知識庫時，請使用此 skill。"
---

<!--
跨 agent 筆記（僅供參考；主機 agent 會忽略）：
  - 支援的 skill 根目錄：GitHub Copilot CLI (~/.copilot/skills, ~/.agents/skills,
    .github/skills, .claude/skills, .agents/skills)、Amp (.agents/skills,
    ~/.config/agents/skills, ~/.config/amp/skills)、Claude Code (~/.claude/skills)。
  - `allowed-tools` 被刻意省略以保持中立：Copilot CLI 使用
    `shell`/MCP-server 名稱，Claude 使用 `Bash`/`Read`/`Write`/`Glob`/`Grep`，Amp
    加入 `shell_command`。此 skill 需要 shell（執行 extract.py）與檔案
    讀/寫 — 各主機在首次使用時會提示這些權限。
  - 參數提示：<path-to-document-folder-or-glob>... [skill-name-slug]
-->

# Book-to-Skill 轉換器

透過提取結構（而非產生摘要），將書面知識轉換為可執行的 agent skills。

## 理念

書籍包含了結晶化的專業知識：歷經多年發展的框架、原則與技巧。此 skill 將這些知識提取為 GitHub Copilot CLI、Amp、Claude Code 或其他相容 agent 可以重複利用的格式。

**提取結構，而非摘要。** skill 不是讀書心得報告。它是一個工具包，包含：
- 具名的框架（具有明確應用場景的心智模型）
- 可執行的原則（指導決策的規則）
- 技巧（循序漸進的方法）
- 反模式（應該避免什麼以及原因）
- 語氣校準（作者如何思考與溝通）

**保留作者的精準度。** 框架通常有特定的命名原因。「The 5 Whys」不能替換為「ask why multiple times」。請捕捉確切的表述。

**適當的分層深度。** 簡單的書籍 → 簡單的 skills。具有 10 個以上框架的複雜書籍 → 帶有參考檔案與按需載入章節的 skills。

---

## 運作模式

提供四種路徑。根據使用者的要求進行路由：

### 1. 完整轉換 (預設)
**觸發條件：** 使用者提供一個或多個文件/目錄/glob 路徑，且沒有特殊指示
**動作：** 執行下方所有步驟 (步驟 0–9)
**輸出：** 完整的 skill，包含 SKILL.md、chapters/、術語表 (glossary)、模式 (patterns)、備忘錄 (cheatsheet)

### 2. 僅分析 (Analyze Only)
**觸發條件：** 使用者說「分析」、「只需提取」，或「我想在生成前先檢閱」
**動作：** 執行步驟 0–3，然後產生結構化的提取報告（找到的框架、原則、技巧）。停止 — **不要**生成 skill 檔案。
**輸出：** 供使用者檢閱的分析報告

### 3. 從先前的分析生成
**觸發條件：** 使用者有現有的分析筆記，或先前已執行過僅分析
**動作：** 跳過步驟 0–3，使用提供的分析作為輸入，執行步驟 4–9
**輸出：** 從提供的分析生成的 skill 檔案

### 4. 更新 / 融入 (現有的 Skill)
**觸發條件：** 使用者提供一個或多個新來源路徑，並表明他們想要更新現有的 skill（無論是指向現有 skill 資料夾、提供 `SKILLS_HOME` 中已存在的 skill slug，或是明確要求更新）。
**動作：** 執行步驟 0（範圍外檢查）、步驟 1（驗證輸入）、步驟 1.5（識別書籍類型），以及步驟 2（提取新檔案）。接著跳至步驟 5（識別/偵測現有 skill 路徑）並執行 **更新 / 融入 Workflow** 以將新內容合併到現有的 skill 檔案中。
**輸出：** 更新後的現有 skill，包含新增/修改的章節摘要與合併的索引/術語表。

---

## Skill 位置

此轉換器可以從多個 skill 系統執行。在尋找此轉換器的輔助指令碼或寫入生成的書籍 skill 時，優先選擇以下位置：

1. GitHub Copilot CLI 個人 skills: `~/.copilot/skills/`
2. 跨 agent 個人 skills (Copilot, Amp, Codex): `~/.agents/skills/`
3. Claude Code 個人 skills: `~/.claude/skills/`
4. 專案本機 Copilot skills: `.github/skills/`
5. 專案本機 Claude skills: `.claude/skills/`
6. 專案本機 Amp / Copilot skills: `.agents/skills/`
7. Amp 全域 skills: `~/.config/agents/skills/`
8. Amp 舊版全域 skills: `~/.config/amp/skills/`

對於**生成的**書籍 skills，請挑選使用者主機 agent 實際能發現的目的地 (請見步驟 5)。如果存在多個有效的根目錄，請詢問使用者一次，並在工作階段中記住答案 — 不要默默地使用預設值。

---

## 步驟 0 — 範圍外檢查

如果未提供任何參數，請停止並回應：
> "book-to-skill requires a supported document path, folder, or glob pattern. Usage: `book-to-skill <path-to-document-folder-or-glob>... [skill-name-slug]`"

在整個 workflow 中：
- 識別輸入路徑與可選的 skill slug。
- 如果最後一個參數不是存在的檔案、資料夾、或符合任何檔案的 glob，且它看起來像一個 skill slug（例如，小寫連字號、英數字），則將其視為 `SKILL_NAME`。
- 將所有其他參數視為 `INPUT_PATHS` 列表。
- 如果任何輸入路徑是現有的 skill 目錄（包含 `SKILL.md` 和 `chapters/` 子資料夾），或如果 `SKILL_NAME` 與 `SKILLS_HOME` 中現有的 skill slug 相符，則將此執行標記為 **更新/融入 (Update/Fold-in)** 操作 (模式 4)。

---

## 步驟 1 — 驗證輸入

驗證 `INPUT_PATHS` 中至少有一個支援的檔案、目錄或 glob 模式。
對於目錄和 glob，將其展開以尋找符合的支援檔案（`.pdf`、`.epub`、`.docx`、`.txt`、`.md`、`.markdown`、`.rst`、`.adoc`、`.html`、`.htm`、`.rtf`、`.mobi`、`.azw`、`.azw3`）。

如果找不到支援的檔案，請停止並提供明確的錯誤訊息。

---

## 步驟 1.5 — 識別內容類型

在提取之前，請詢問使用者：

> "What kind of content do these sources have? This helps me choose the best extraction method.
>
> 1. **Technical** — 有程式碼區塊、表格、公式、圖表（例如程式設計書籍、學術論文、架構指南）
> 2. **Text-heavy** — 大部分是散文，很少或沒有表格/程式碼（例如管理、生產力、敘事性非小說類）
> 3. **Not sure** — 我將使用快速方法，如果品質似乎有限，會提醒您"

將答案儲存為 `BOOK_TYPE`：
- 選項 1 → `BOOK_TYPE=technical`
- 選項 2 → `BOOK_TYPE=text`
- 選項 3 → `BOOK_TYPE=text`

**如果 `BOOK_TYPE=technical`**，在繼續前通知使用者：
> "📐 Technical mode selected — 使用 Docling 進行結構感知提取（保留為 markdown 的表格、程式碼區塊、公式）。這大約需要每頁 1.5 秒，所以較長的來源預計需要幾分鐘時間。現在開始…"

**如果 `BOOK_TYPE=text`**，通知：
> "📄 Text mode selected — 為每種檔案類型使用最快適合的提取器。純文字/Markdown/HTML 通常在幾秒鐘內準備就緒；PDF 會在可用時使用 pdftotext。"

---

## 步驟 2 — 從來源文件提取文字

執行提取指令碼，傳遞輸入路徑：

```bash
SCRIPT_PATH=""
for candidate in \
  "$HOME/.copilot/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.agents/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.claude/skills/book-to-skill/scripts/extract.py" \
  ".github/skills/book-to-skill/scripts/extract.py" \
  ".claude/skills/book-to-skill/scripts/extract.py" \
  ".agents/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.config/agents/skills/book-to-skill/scripts/extract.py" \
  "$HOME/.config/amp/skills/book-to-skill/scripts/extract.py"
do
  if [ -f "$candidate" ]; then
    SCRIPT_PATH="$candidate"
    break
  fi
done

if [ -z "$SCRIPT_PATH" ]; then
  echo "Could not find scripts/extract.py for book-to-skill" >&2
  exit 1
fi

PYTHON_BIN="${PYTHON_BIN:-python3}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi

"$PYTHON_BIN" "$SCRIPT_PATH" $INPUT_PATHS --mode <BOOK_TYPE> --install-missing ask
```

在提取之前，指令碼會檢查偵測到的格式所需的選用 Python 套件。如果缺少更好的提取器，它會提示使用者提供可用的後備選項。非互動式工作階段預設為後備選項，除非明確將安裝模式設為 `yes`。

**提示 — 環境預檢：** 執行 `"$PYTHON_BIN" "$SCRIPT_PATH" --check` 列印每種格式已安裝提取器的報告，以及安裝缺少項目的確切指令，而不處理任何檔案。當使用者報告設定或品質問題時很有用。

這會建立：
- `<tempdir>/book_skill_work/full_text.txt` — 合併所有來源提取的文字，並具有清晰視覺區分的邊界。
- `<tempdir>/book_skill_work/metadata.json` — 整體合併的大小、字數、頁數、token 數量、捨棄的 EPUB 圖片數量，以及詳細處理過 `sources` 的列表。

讀取 `<tempdir>/book_skill_work/metadata.json` 以檢查結果。

---

## 步驟 2.5 — 預檢成本估算

讀取 `<tempdir>/book_skill_work/metadata.json` 並**在進行任何生成之前**向使用者展示估算：

```
📖 偵測到來源: <total_sources> 來源
<列出 metadata 清單中的每個來源檔案名稱和格式>
<如果 images_dropped > 5: 警告有 N 個來源圖片未被讀取>
📄 合併的頁面/章節: ~<N> | 字數: ~<N> | 總 tokens: ~<N>K

💰 估計的 token 成本 (完整轉換 / 更新):
   輸入  (讀取 + prompts): ~<N>K tokens
   輸出 (生成/更新的 skill 檔案):  ~<N>K tokens
   總計:                           ~<N>K tokens

   成本: 將上述 token 數乘以您的模型目前的
   每 100 萬 token 輸入/輸出費率（價格和模型名稱經常變動 —
   請勿寫死；引用今天的費率並將其標記為估算值）。

   ⏱  預估時間: ~<N> 分鐘

📁 將生成/更新的檔案:
   SKILL.md + chapter 檔案 + glossary + patterns + cheatsheet

➡  繼續完整轉換 / 更新？ (或輸入 "analyze only" 先行預覽)
```

**如何估算：**
- 輸入 tokens ≈ metadata 中的 `estimated_tokens` × 1.3 (每章節傳遞的 prompts 額外負擔)
- 輸出 tokens ≈ 章節數 × 每章預算 + 4,000 (SKILL.md) + 4,500 (glossary + patterns + cheatsheet)
  - 根據 `BOOK_TYPE` 決定每章預算中位數 (`DEPTH` 在步驟 4 決定，可能會提高它)：`text` ≈ 1,000，`technical` ≈ 1,800。如果使用者已經指出是僅參考還是深入學習，則使用步驟 7 矩陣的對應列。
- 成本：報告 token 數量並將其乘以使用者目前的每 100 萬 token 輸入/輸出費率。**不要**寫死具體金額 — 模型名稱和價格會變動；如果您顯示，請標記為估算並註明日期。

等待使用者確認後再繼續。如果他們說 "analyze only"，則切換到模式 2。

---

## 步驟 2.6 — 大型書籍（> 5 萬 tokens）的 REPL 風格存取

受到遞迴語言模型 (RLM) 典範的啟發：將 `full_text.txt` 視為可查詢的語料庫，而不是一次性讀取。將整個檔案載入上下文中，會耗盡您稍後生成所需的預算。

對於超過約 5 萬 tokens 的書籍，偏好使用程式設計方式探測，而不是沒有邊界的 `Read(full_text.txt)`：

```bash
# 在任何 Read 之前檢查大小
wc -w "$FULL_TEXT_PATH"

# 尋找章節偏移量而不載入整個檔案
grep -n -E "^\s*(Chapter|CHAPTER)\s+[0-9]+" "$FULL_TEXT_PATH" | head -40

# 僅拉取您需要的章節（包含開始到結束行）
sed -n '<start>,<end>p' "$FULL_TEXT_PATH"

# 在在 SKILL.md 宣告之前驗證框架確實被提及
grep -c -i "westrum\|dora" "$FULL_TEXT_PATH"

# 具有偏移量/限制的目標 Read 避免傾印完整檔案
# Read(file_path=full_text.txt, offset=<line>, limit=<lines>)
```

針對步驟 3（結構分析）、步驟 7（每章摘要）與步驟 8（glossary / patterns 提取）使用這種方法。在不到 5 萬 tokens 的書籍上，單次 `Read` 就可以了。

為什麼這很重要：一本 200 頁的書約有 7.5 萬個 tokens。每章重讀一次（28 次）花費約 200 萬個輸入 tokens；使用 grep + sed 僅提取相關的片段可保持生成成本與輸出成正比，而不是與來源成正比。

---

## 步驟 3 — 分析書籍結構

讀取提取的 `full_text.txt` 的前 8,000 個字元來識別：
- 書籍**標題 (title)** 與**作者 (author(s))**
- **章節結構**（尋找 "Chapter N"、"PART I"、編號標題、目錄）
- **核心主題**與學科領域
- 大致章節數量

然後閱讀目錄部分（如果有的話）以映射所有章節。

**如果模式為「僅分析 (Analyze Only)」：** 現在產生提取報告並停止。結構：
```
## 提取報告 — <Title>

### 作者核心框架 (Author's Core Frameworks)
- **<Framework Name>**: <它是什麼以及何時應用>

### 關鍵原則 (Key Principles)
- <Principle>: <可執行的規則>

### 技巧與方法 (Techniques & Methods)
- <Technique>: <循序漸進或操作指南>

### 反模式 (Anti-patterns)
- <要避免什麼>: <為什麼>

### 建議的 Skill 名稱
`{author-lastname}-{core-concept}` — 例如 `cialdini-influence`

### 偵測到的章節 (Chapters Detected)
| # | 標題 | 主要框架 |
```

---

## 步驟 4 — 詢問目的 (僅限完整轉換)

在生成之前，請詢問使用者：

> "What should this skill help you do? (Pick one or more)
> 1. Apply the author's frameworks while working
> 2. Think with the author's mental models
> 3. Reference specific chapters and concepts
> 4. All of the above"

使用答案來權衡在 SKILL.md 核心 (Core) 部分要強調的內容。

**根據答案衍生 `DEPTH`（沒有額外的 prompt）：**
- 答案**僅為**選項 3 (參考) → `DEPTH=reference` — 精簡、快速查詢的章節。
- 答案包含選項 1、2 或 4 → `DEPTH=study` — 更深入的章節，包含更多實際詳細資訊、範例與推論。

`DEPTH` 和 `BOOK_TYPE` 共同在步驟 7 中設定每章 token 預算。**不要**單獨詢問「學習與參考 (study vs reference)」的問題 — 它是在此處推斷出來的。（在跳過步驟 4 的模式 2/3 中，預設 `DEPTH=study`。）

---

## 步驟 5 — 決定 skill 名稱

如果提供了 `SKILL_NAME`，則將其用作 skill slug。
否則，提出兩個選項讓使用者選擇：
- **按作者-概念**: `{author-lastname}-{core-concept}` (例如 `cialdini-influence`, `meadows-systems`)
- **按標題**: 書名的連字號小寫 (例如 `designing-data-intensive-apps`)

如果該書具有強烈的方法論特徵，則預設採用作者-概念格式。

選擇目的地 skill 根目錄 (`SKILLS_HOME`)。探測使用者檔案系統以尋找現有 skill roots，並由**使用者正在執行的主機**挑選：

| Host agent | 個人 skill root (依序探測) | 專案本機 root |
|---|---|---|
| **GitHub Copilot CLI** | `~/.copilot/skills` → `~/.agents/skills` | `.github/skills` → `.claude/skills` → `.agents/skills` |
| **Amp** | `~/.agents/skills` → `~/.config/agents/skills` → `~/.config/amp/skills` | `.agents/skills` |
| **Claude Code** | `~/.claude/skills` | `.claude/skills` |
| **OpenAI Codex** | `~/.agents/skills` (原生發現; 遵循符號連結) | `.agents/skills` |

選擇規則：
1. 如果主機的候選根目錄中在磁碟上**只有一個**存在，則不詢問直接使用它。
2. 如果**全都不**存在（新機器），則詢問使用者要建立哪一個根目錄 — 呈現適合主機的選項，並在工作階段中記住該選擇。不要默默地挑選。
3. 如果使用者明確要求專案本機輸出，則偏好專案本機列。
4. 如果您無法識別主機，則詢問：「您正在哪個 agent 中執行這個 — GitHub Copilot CLI、Amp、Codex，還是 Claude Code？」

將 `SKILLS_HOME` 設定為選擇的根目錄，並檢查 `$SKILLS_HOME/<skill_name>/` 是否已存在。
如果存在，請提示使用者選擇：
1. **更新 / 融入 (Update / Fold-in)** (模式 4) — 將新檔案/內容整合至現有 skill 元件中。
2. **覆寫 (Overwrite)** — 刪除並從頭開始重新生成 skill。
3. **重新命名 (Rename)** — 附加 `-2` 或使用其他自訂 slug。

如果使用者選擇 **更新 / 融入 (Update / Fold-in)**，請立即繼續進行步驟 2.5 之後的 **更新 / 融入 Workflow** 部分（跳過步驟 3、4、6、7、8、9）。

---

## 步驟 6 — 建立 skill 目錄結構

```bash
mkdir -p "$SKILLS_HOME/<skill_name>/chapters"
```

---

## 步驟 7 — 生成章節摘要

**TOKEN 預算規則 — 關鍵 (自適應)：**

每章的預算隨著 `BOOK_TYPE` 與 `DEPTH` 進行縮放。技術章節需要空間放置程式碼與表格；學習深度需要空間放置推論過程。從此矩陣中挑選預算：

| | `DEPTH=reference` | `DEPTH=study` |
|---|---|---|
| `BOOK_TYPE=text` | 800–1,200 tokens | 1,000–1,800 tokens |
| `BOOK_TYPE=technical` | 1,200–1,800 tokens | 2,000–3,000 tokens |

- 這些是單檔的目標，而不是硬性上限 — 密集的章節可能會超出，稀疏的章節可能會低於目標。密度仍勝於長度（品質規則 #3）：絕不為了達到數字而填補內容。
- 檔案是按需載入的，所以較大的章節只有在實際閱讀該章節時才會花費 tokens。
- 介於兩個儲存格之間（例如混合內容書籍）若有疑問，使用較低的預算，並讓深度來自精確度，而不是份量。

**`DEPTH=study` 必須以內容來爭取，而不是用更大的數字。** 標準小節範本 (Core Idea → Connects To) 自然會將密集散文章節的字數落在約 700–900 tokens 左右。若要*真實地*達到學習預算（不是透過填補內容），研究深度章節必須增加具體的材料：
- **從該章節重現一個實際範例或工件**（例如範例新聞稿、範例對話、填寫好的範本、作者演練的決策），置於 `## Worked Example` 標題之下。這是最大的槓桿，也是學習者回頭尋求的主要內容。
- **將每個框架的「如何做 (How)」展開**為明確的步驟或標準，而非單行敘述。
- **針對前 1–2 個框架加入簡短的「為什麼有效 / 失敗模式」附註**。

如果一章確實沒有實際範例且難以擴充，就讓它低於學習預算的下限，而非強行填補內容 — 並註明該章的核心概念較為單薄。相較之下，`reference` 深度的章節會刻意省略實際範例，僅保留準備好可供決策的精華。

針對步驟 3 辨識出的**每個**章節/主要區段：

讀取擷取後的 `full_text.txt` 的對應區段 (使用字元位移或針對章節標題使用 grep)。

使用下方結構建立 `$SKILLS_HOME/<skill_name>/chapters/ch<NN>-<slug>.md`。

**根據 `BOOK_TYPE` 調整重點：**
- `technical` → 優先處理 "Code Examples", "Reference Tables", 與 "Commands & APIs" 區段；保留精確的語法。
- `text` → 優先處理 "Frameworks Introduced", "Mental Models", 與 "Key Takeaways"；跳過空白的技術區段。

```markdown
# Chapter N: <Full Title>

## Core Idea
<1–2 句話：本章教授最重要的一件事>

## Frameworks Introduced
- **<Framework Name>**: <精確的表述 — 保留作者的命名>
  - 何時使用：<具體情況>
  - 如何操作：<步驟或標準>

## Key Concepts
- **<Term>**: <1 句話精確定義>
(本章 5–10 個最重要的術語)

## Mental Models
<2–4 個框架或思考工具。寫作格式如 "使用 X 當 Y 發生時" 或 "將 X 視為 Y">

## Anti-patterns
- **<要避免的事項>**: <為何失敗>

## Code Examples *(僅技術書籍 — 如果 BOOK_TYPE=text 則省略)*
<!-- 複製該章最具指導意義的程式碼片段。務必原樣保留縮排。 -->
```<language>
<本章關鍵程式碼範例>
```
- **它展示了什麼**: <單行說明>

## Reference Tables *(僅技術書籍 — 如果 BOOK_TYPE=text 則省略)*
<!-- 在 markdown 中重現該章的任何比較矩陣、參數表或決策表。 -->

## Worked Example *(僅限 DEPTH=study — 如果 DEPTH=reference 則省略)*
<!-- 重現或重建作者演練的一個具體範例：
     範例文件、對話、填妥的範本、對比前後、
     或從頭到尾完整的決策過程。這是學習章節值得其預算的原因。
     忠於來源；絕不要複製大量原始段落 —
     請精簡地重建範例。 -->

## Key Takeaways
1. <可執行的洞見>
2. <可執行的洞見>
3. <可執行的洞見>
(從業者必須記住的 3–7 個收穫)

## Connects To
- **Ch N**: <為何此章節相關聯>
- **<Concept>**: <它所連結的外部概念或標準>
```

---

## 步驟 8 — 產生支援檔案

### glossary.md
建立 `$SKILLS_HOME/<skill_name>/glossary.md`:
- 來自書籍的每個重要術語，按字母排序
- 格式：`**Term** — 定義 (Ch N)`
- 最大 1,500 tokens

### patterns.md
建立 `$SKILLS_HOME/<skill_name>/patterns.md`:
- 來自書籍的所有具體技巧、設計模式、演算法
- 格式：`## Pattern Name\n**When to use**: ...\n**How**: ...\n**Trade-offs**: ...`
- 最大 2,000 tokens

### cheatsheet.md
建立 `$SKILLS_HOME/<skill_name>/cheatsheet.md`:

**這是 skill 中最具差異化的一層 — 請將其視為推論的輔助工具，而非關鍵字列表。** 任何人都能夠在術語表中 grep 搜尋詞彙。備忘錄記錄的是作者的**判斷力**：他們會做出什麼決定以及原因。這是將「我懂得這些詞」轉化為「我會像作者那樣行動」的檔案。

優先順序：
1. **決策規則 (Decision rules)** — 「當 X 時，做 Y，因為 Z」。作者應用的 if/then 邏輯，並以讀者不需重讀書籍即可套用的方式陳述。
2. **決策樹 (Decision trees) / 流程圖 (flowcharts)** (作為巢狀項目符號或小型表格) — 用於多於兩個分支的選擇。
3. **權衡矩陣 (Trade-off matrices)** — 在作者關心的維度上對相互競爭的選項進行評分，讓讀者可在自身限制下進行選擇。
4. **閾值 (Thresholds) & 預設值 (defaults)** — 作者承諾的具體數字、比例或經驗法則（例如「將函數保持在 20 行以內」，「當錯誤預算 < 10% 時發出警報」）。
5. **跡象 (Tells) & 氣味 (smells)** — 用於辨識狀況的快速啟發法 (heuristics)（「如果您看到 X，您可能遇到了麻煩 Y」）。

避免：僅有 詞彙→定義 的列表（那是術語表），以及散文段落（那是章節）。每一行都應該幫助讀者**決定**某些事。

- 大多數格式化為精簡的表格和決策規則；您會希望將其放在單一列印頁面上並在工作時放在身邊的內容。
- 最大 1,200 tokens。

---

## 步驟 9 — 產生主檔案 SKILL.md

**關鍵 TOKEN 預算：將 SKILL.md 內文保持在 4,000 tokens 以下。**
壓實會從尾部截斷 — 請將最重要的內容放在前面。

建立 `$SKILLS_HOME/<skill_name>/SKILL.md`:

```markdown
---
name: <skill_name>
description: "來自 <Author(s)> 所著 \"<Full Title>\" 的知識庫。當為了 <key topics, 3–6 terms> 應用 <author> 的框架、學習書籍或參考其概念時使用。"
---

<!-- argument-hint: [topic, framework name, or chapter number] -->

# <Full Title>
**Author**: <Author(s)> | **Pages**: ~<N> | **Chapters**: <N> | **Generated**: <YYYY-MM-DD>

## How to Use This Skill

- **Without arguments** — 載入核心框架以供參考
- **With a topic** — 詢問關於 `replication`、`pricing` 或其他建立索引的主題；我將找到並閱讀相關章節
- **With chapter** — 要求 `ch05`；我將載入該特定章節
- **Browse** — 詢問 "what chapters do you have?" 以查看完整索引

當您詢問未涵蓋在下方 Core Frameworks 中的主題時，我將在
回答前閱讀相關章節檔案。

---

## Core Frameworks & Mental Models
<!-- ~2,000 tokens：作者最重要且具名的框架與原則。
     保留確切名稱。寫作格式如 "使用 X 當 Y 發生時", "偏好 X 勝於 Y 因為 Z"。
     這是一個工具包，而不是摘要。 -->

<在此產生 2,000 tokens 最關鍵的框架和見解>

---

## Chapter Index

| # | Title | Key Frameworks |
|---|-------|----------------|
| [ch01](chapters/ch01-<slug>.md) | <Title> | <framework1>, <framework2> |
| [ch02](chapters/ch02-<slug>.md) | <Title> | <framework1>, <framework2> |
...

## Topic Index

<!-- 字母順序。主要詞彙/框架 → 涵蓋它們的章節。 -->
- **<Term>** → ch<N>[, ch<N>]
- **<Term>** → ch<N>

## Supporting Files

- [glossary.md](glossary.md) — 包含所有關鍵詞彙及定義
- [patterns.md](patterns.md) — 所有的技巧與設計模式
- [cheatsheet.md](cheatsheet.md) — 快速參考表格和決策指南

---

## Scope & Limits

此 skill 僅涵蓋書籍內容。若要在您的 codebase 中進行實務部署，
請與專案特定的工具結合使用。針對超出本書範圍的主題，請查看相關 skills
或直接詢問 agent。
<如果 images_dropped > 5: 宣告有 N 個來源圖片未被讀取>
```

---

## 步驟 9.5 — 掃描生成的 skill

在回報成功、在另一個工作階段中載入 skill，或發佈之前，請執行諮詢性質的安全性掃描：

```bash
SKILL_CONVERTER_ROOT="$(cd "$(dirname "$SCRIPT_PATH")/.." && pwd)"
"$PYTHON_BIN" "$SKILL_CONVERTER_ROOT/tools/scan_generated_skill.py" "$SKILLS_HOME/<skill_name>"
```

如果掃描器結束碼非零，請停止並要求人類檢閱其檔案/行發現的問題。請勿默默重寫產生的檔案，且在問題解決或明確接受前，請勿載入或發佈該 skill。

---

## 步驟 10 — 清理並報告

```bash
PYTHON_BIN="${PYTHON_BIN:-python3}"
if ! command -v "$PYTHON_BIN" >/dev/null 2>&1; then
  PYTHON_BIN="python"
fi

"$PYTHON_BIN" - <<'PY'
import os
import shutil
import tempfile
from pathlib import Path
shutil.rmtree(
    os.environ.get("BOOK_SKILL_WORKDIR", Path(tempfile.gettempdir()) / "book_skill_work"),
    ignore_errors=True,
)
PY
```

接著向使用者報告：

```
✅ Skill created: $SKILLS_HOME/<skill_name>/

📚 Book: <Full Title> — <Author>
📄 Pages: ~<N> | Chapters: <N>

Files generated:
  SKILL.md         — core frameworks + index   (~X tokens)
  chapters/        — <N> chapter summaries     (~X tokens each, ~X total)
  glossary.md      — key terms                 (~X tokens)
  patterns.md      — techniques & patterns     (~X tokens)
  cheatsheet.md    — quick reference           (~X tokens)
  ─────────────────────────────────────────────────────
  Total skill size: ~X tokens (按需載入，並非一次全部)

💡 Tip: check your agent's session cost/usage command to see actual token usage.

Usage:
  Ask for <skill_name>                  → 載入核心框架
  Ask <skill_name> about <topic>        → 尋找並解釋某個主題
  Ask <skill_name> for ch<N>            → 深入探索特定章節

Reload (如果您的 agent 沒有自動偵測新 skills):
  GitHub Copilot CLI:  /skills reload
  Claude Code:         重新啟動工作階段
  Amp:                 重新啟動工作階段

Share this skill (可選):
  GitHub repo, installable on any host (Step 11):  say "publish"
  Copilot ecosystem:  gh skill publish $SKILLS_HOME/<skill_name>
```

---

## 步驟 11 — 將生成的 skill 發佈到 GitHub (可選)

在步驟 10 報告之後，提供一次詢問 — 且僅限於步驟 9.5 掃描通過的情況：

> "Want me to publish this skill to GitHub so any Agent Skills host can install it with `npx skills add`? (yes / skip)"

如果使用者拒絕，請在此停止。必要條件：已驗證的 `gh` CLI（檢查 `gh auth status`）。如果缺少 `gh` 或未驗證，請提供設定協助（`brew install gh` 或 https://cli.github.com，接著執行 `gh auth login`） — 或是使用不需 `gh` 的路徑：使用者在 GitHub Web 介面上建立所選可見度的空白儲存庫，然後您執行下方的 `git init`/`add`/`commit` 指令，接著執行 `git remote add origin <repo-url> && git push -u origin main`。下方的可見度規則完全同樣適用於 Web 建立的儲存庫。

**可見度 (Visibility) 是一個獨立的封閉式問題 — 從不進行推斷，也不從先前的答案中讀取。** 一旦使用者接受發佈，請獨立詢問並要求提供一個單字的答覆：

> "Private or public repository? Reply with one word: `private` or `public`."

**答覆必須*正是* `public`，而不只是包含它 — 這是硬性規定，而非建議。** 除非在以下唯一情況下，否則在各種情況下均執行包含 `--private` 的 `gh repo create`：對可見度問題的答案是單字 `public`。嚴禁使用子字串比對，因為關於**來源授權的句子不是可見度的答案** — "it's public domain", "the book is public domain", "it's publicly available" 全部描述的是材料而非儲存庫，並且均會解析為 `--private`。釋義、完整句子、模稜兩可的答案、沉默或您自己的推斷都**不構成**同意：重新詢問一次，如果答覆仍然不是單字，請使用 `--private` 並在報告中說明。私人儲存庫稍後可以公開；衍生自書籍內容的公開推送無法取消發佈。

**版權閘門 — 在建立儲存庫前一律套用：** 章節檔案是綜合的摘要，而非原始文字，但它們仍然衍生自來源材料。根據 README 的版權與合理使用政策，從**第三方版權書籍**生成的 skills 必須保持私有 (private)；只有當來源為使用者自己的著作、採開放授權的內容，或使用者明確確認他們有權公開重新散布的材料時，才提供公開 (public) 選項 — 並聲明適用哪種情況。能夠取得內部公司材料並不表示有權揭露它：衍生自內部文件的 skills 必須維持**私有**，除非使用者聲明他們擁有出版權。

如果接受：

1. 在 `$SKILLS_HOME/<skill_name>/` 內部新增 `README.md` 儲存庫檔案（不要覆寫現有檔案） — 包含 skill 標題、單一段落的描述 ("Agent skill generated from *<Title>* by <Author> with [book-to-skill](https://github.com/virgiliojr94/book-to-skill)")、來自下方步驟 3 的安裝指令、檔案清單，以及一份宣告該內容為綜合摘要而非書籍文字的聲明。
2. 將 skill 資料夾初始化為 git 儲存庫並建立遠端（預設儲存庫名稱 `<skill_name>`；允許使用者覆寫 — 有些人偏好加上 `<skill_name>-skill` 後綴）。**巢狀儲存庫防護：**首先檢查 skill 資料夾是否已位於 git 儲存庫內 (`git -C "$SKILLS_HOME/<skill_name>" rev-parse --show-toplevel` — 像 `.claude/skills/` 這樣的專案本機根目錄始終如此)。如果是，請**不要**就地執行 `git init`：外部儲存庫會將該資料夾記錄為嵌入式儲存庫 (gitlink, mode 160000) 而不包含 `.gitmodules`，這會導致新 clone 外部專案時默默略過此 skill。相反地，請將 skill 資料夾複製到暫存目錄，從副本執行下方的指令，並告訴使用者發佈的儲存庫（而非專案本機資料夾）才是遠端的工作複本 (working copy)。

```bash
cd "$SKILLS_HOME/<skill_name>"
git init -b main
git add -A
git commit -m "Add <skill_name> skill"
gh repo create <repo_name> --private --source . --push
# 預設為 --private；只有在符合上述可見度規則時才替換為 --public
# (可見度答案是單字 "public" 並且版權閘門允許)
```

3. 報告儲存庫 URL 與跨主機安裝指令：

```
✅ Published: https://github.com/<owner>/<repo_name> (<private|public>)

Install on any Agent Skills host:
  npx skills add https://github.com/<owner>/<repo_name> --skill <skill_name>
```

   當觸發巢狀儲存庫防護並且儲存庫從暫存副本發佈時，新增一行說明 — 該本機資料夾永遠不會取得遠端，所以它的 Update/Fold-in 發佈優惠將永遠不會出現：

```
⚠️  Published from a copy: <skill folder> sits inside another git repository, so it has
    no remote of its own. To publish a later update, re-run Step 11, or clone
    https://github.com/<owner>/<repo_name> and fold new material into the clone.
```

根層級 `SKILL.md` 的佈局與 `skills` CLI 偵測到的完全相同，因此儲存庫可以直接安裝 — 不需要重組。在巢狀儲存庫以外的情況下，本機資料夾保留為此機器的實時安裝，並且是遠端的工作複本，因此後續的 Update/Fold-in 執行可以將它們的變更 commit 並 push 到同一個遠端。

---

## 更新 / 融入 (Update / Fold-in) Workflow

對位於 `$SKILLS_HOME/<skill_name>/` 的現有 skill 執行 更新/融入 (Update/Fold-in) 操作時：

### 1. 讀取現有 Skill 結構
讀取並解析現有 skill 的檔案：
- 讀取 `$SKILLS_HOME/<skill_name>/SKILL.md` 來解析現有的 **Chapter Index**, **Topic Index**, metadata（作者、總章節數）以及 **Core Frameworks**。
- 列出 `$SKILLS_HOME/<skill_name>/chapters/` 裡的所有檔案以找到最大的章節編號 (例如 `ch12`)。
- 讀取 `$SKILLS_HOME/<skill_name>/glossary.md`, `$SKILLS_HOME/<skill_name>/patterns.md`, 以及 `$SKILLS_HOME/<skill_name>/cheatsheet.md` 以查看已編入索引的術語和框架。

### 2. 匹配內容並識別修訂與新增
分析 `<tempdir>/book_skill_work/full_text.txt` 中新提取的文字，以識別新內容是否代表：
- **對現有章節的更新/修訂 (Updates/Revisions)**：如果新內容的某個區段直接更新或擴展了現有章節的主題，讀取現有的章節檔案，將新細節合併進去，並重寫該檔案。
- **新增內容 (New additions)**：如果內容引入了新的章節、論文或獨立的區段，在 `chapters/` 下建立**新的章節摘要檔案**。從現有最大章節編號之後開始編號 (例如，如果現有章節到 `ch12` 為止，則建立 `ch13-*.md`, `ch14-*.md` 等等)。

### 3. 生成或更新章節摘要檔案
針對每個新的或修訂的章節：
- 讀取新提取文字的對應區段。
- 遵循 **步驟 7** 的格式化指南來建構摘要。
- 在 `$SKILLS_HOME/<skill_name>/chapters/` 寫入/更新檔案。

### 4. 合併支援檔案
- **合併 glossary.md**:
  - 讀取現有的 `$SKILLS_HOME/<skill_name>/glossary.md`。
  - 從新內容中提取所有新術語和定義（步驟 8 glossary 指南）。
  - 將現有術語與新術語合併並按字母順序排列。
  - 如果術語已經存在，將新章節/來源參考附加在後面 (例如 `**Term** — 定義 (Ch 4, Ch 13)`)。
  - 將完全合併且按字母排序的列表重寫入 `$SKILLS_HOME/<skill_name>/glossary.md`。
- **合併 patterns.md**:
  - 讀取現有的 `$SKILLS_HOME/<skill_name>/patterns.md`。
  - 從新內容中提取任何新的技巧、演算法或模式。
  - 附加新模式，確保格式一致，並保持總長度精簡（2,500 tokens 以內）。
- **合併 cheatsheet.md**:
  - 讀取現有的 `$SKILLS_HOME/<skill_name>/cheatsheet.md`。
  - 提取新的比較規則、決策表或參數指南。
  - 將它們乾淨地整合進 cheatsheet 結構中。

### 5. 重新生成主 SKILL.md
更新主 skill 檔案 `$SKILLS_HOME/<skill_name>/SKILL.md`:
- **Metadata**: 遞增章節數，更新估算的頁數，並在適當時加入新來源名稱。將 `Generated` 日期更新為當前日期。
- **Core Frameworks**: 融入來自新內容的最具影響力的心智模型或原則（確保整體檔案維持在 4,000 tokens 以下）。
- **Chapter Index**: 將新章節附加到索引表中，連結到新建立的檔案。
- **Topic Index**: 按字母順序合併新主題。如果現有主題在新的章節中也被涵蓋，請將新章節連結附加至該行 (例如 `- **Topic** → ch05, ch13`)。

### 6. 掃描、清理與報告
一旦檔案成功寫入及合併，執行 **步驟 9.5**，接著進行 **步驟 10** 進行清理並列印一份自訂更新報告，總結新加入的章節、合併的術語表項目和更新的索引。如果 skill 資料夾是有 remote 的 git 儲存庫 (透過 **步驟 11** 發佈)，請提供 commit 更新並 push。

---

## 品質規則

1. **提取結構，而非摘要** — 捕捉具名的框架、確切的表述、反模式；而非章節重點回顧
2. **保留作者的精準度** — "The 5 Whys" ≠ "ask why multiple times"；保留確切的命名
3. **密度勝於完整度** — 一篇 1,000 tokens 的摘要勝過 10,000 tokens 的摘錄
4. **從業人員語氣 (Practitioner voice)** — 寫作格式為 "使用 X 當 Y 發生時"，而不是 "本書解釋了 X"
5. **前置載入 SKILL.md** — 壓實會保留前 5,000 tokens；最重要的內容放在最前面
6. **章節檔案為按需載入** — 在載入前它們不會計入 skill 預算
7. **絕不複製書籍原始文字** — 始終綜合、總結、提取重點訊號
8. **Topic index 相當重要** — agent 透過它來導航至正確的章節檔案
