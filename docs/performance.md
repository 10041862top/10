---
description: "實測 book-to-skill 的 token 成本：比直接將整本書丟入 context 節省了 24x-51x 的 tokens，解析「探索迴圈稅 (Discovery Loop Tax)」，並提供可重現的各書籍實測數據表。"
seo_title: "效能與 Token 成本 - book-to-skill 基準測試"
---

# 效能與成本 (Performance & Cost)

以下所有的數據都是**實際測量**，而非預估值。我們使用 `tiktoken` (cl100k_base) 來計算 token 數量，並使用 `tools/discovery_tax.py` 作為探索模型的基準。您可以使用文中顯示的指令來重現任何一項數據。

## 擷取 (實際轉換)

使用 `pdftotext` (PDF) 與 `ebooklib` (EPUB) 測得：

| 書籍 | 格式 | 頁數 | Tokens | 自動偵測到的章節數 |
|------|--------|------:|-------:|-----------------------:|
| Think Python 2 | PDF | 244 | 119K | 19 |
| Working Backwards | PDF | 371 | 175K | 10 |
| Pro Git | PDF | 501 | 229K | — † |
| Moby-Dick | EPUB | — | 301K | 133 |

† 《Pro Git》的章節使用段落標題 (沒有 `Chapter N` 標示)，因此無法自動分段。《白鯨記 (Moby-Dick)》的內文使用純標題，但其使用羅馬數字的目錄有被成功偵測到 (133 個章節) — 詳見 README 中的 *已知限制 (Known limitations)*。

**技術書籍的擷取方法很重要。** 以一本 103 頁的技術 PDF 為例：

| 擷取方法 | 耗時 | 表格數量 | 程式碼區塊數量 |
|--------|-----:|-------:|------------:|
| pdftotext | 0.1s | 0 | 0 |
| Docling (technical 模式) | 164s | 48 | 36 |

pdftotext 是瞬間完成的，但會攤平結構；Docling 大約每頁需要 1.5 秒，但能將表格與程式碼保留為 markdown 格式。一般文章請選擇 `text` 模式，若有程式碼或表格請選擇 `technical` 模式。

## 探索迴圈稅 (The Discovery Loop Tax)

為了解答**一個**針對性問題而進入 context 的 token 數量。book-to-skill 會載入一個常駐的核心檔案 (約 4K) 加上一個編譯好的章節 (約 1K) ≈ **5,000 tokens**。

| 書籍 (章節大小) | Context-dump (直接丟入) | Discovery loop (探索迴圈) | book-to-skill | 與丟入/迴圈的對比優勢 |
|---------------------|-------------:|---------------:|--------------:|:--------------:|
| Think Python 2 (小) | 119,264 | 12,152 | ~5,000 | 24× / 2.4× |
| Working Backwards (中) | 175,253 | 33,444 | ~5,000 | 35× / 6.7× |
| AI Engineering (大) | 256,287 | 77,866 | ~5,000 | 51× / 15.6× |

```bash
python3 tools/discovery_tax.py --full-text /tmp/book_skill_work/full_text.txt --target-chapter 5
```

- **對比 Context-dump 的優勢** (24–51×) 是最強而有力的主張：因為這種成本在*每一次對話的每一回合*都會重複發生。
- **對比 Discovery-loop 的優勢** (2.4–15.6×) 是在使用書籍真實目錄/章節大小模型時的單次成本優勢；這個優勢會隨著章節大小成正比放大。

## 生成成本

以下是從實測 tokens 估算出的單次完整轉換成本 (以 Claude Sonnet 4.5 費率為準：輸入/輸出每百萬 Token 為 \$3 / \$15)：

| 書籍 | 輸入 | 輸出 | 預估成本 |
|------|------:|-------:|------:|
| Think Python 2 | 155K | 28K | \$0.88 |
| Working Backwards | 228K | 19K | \$0.96 |
| Pro Git | 298K | 23K | \$1.23 |
| Moby-Dick | 391K | 17K | \$1.42 |

產出一個完整的 skill 大約只需要 **每本書 \$1** — 且只需支付一次。如果每次會話都要將同一份 PDF 重新讀入 context，隨著時間累積，其花費將遠遠超過這個數字 (請參考上方的「探索迴圈稅」)。

## 生成 Skill 的輸出品質

這是關於自適應深度 (adaptive-depth) 變更 (`v1.0.0`, #20) 在單一章節上的前後對比：

| 產出物件 | 舊規格 | 新規格 |
|----------|---------:|---------:|
| 章節檔案 (tokens) | 473 | 1,219 |
| 包含實際操作範例 | 否 | 是 |
| 備忘錄 (Cheatsheet) 決策規則 | 0 | 32 |
| 備忘錄 關鍵字/定義 行數 | 9 | 0 |

新規格將備忘錄從單純的術語表轉變為決策層，並為「學習深度 (study-depth)」的章節提供了重製的實際操作範例。
