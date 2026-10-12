# Evaluation Method & Evidence Limitations

> Local publication candidate · 2026-10-12 · 彙整封存結果，不是本機重跑報告。原始題目、企業 SQL、工具回傳、測試 JSON、部署設定與私人證據索引不在本作品內。

## What the Golden Set measures

| Gate | Method | What the result does not establish |
|---|---|---|
| Knowledge, 5 tasks | Required／forbidden regex checks; empty or truncated answers fail | General accuracy; regex may reject a semantically valid answer |
| SQL agent, 6 tasks | Compare expected numbers／names; check task completion and structured tool calls | Reliability across all enterprise queries or unseen schemas |
| Analysis, 8 tasks | Record completion, repetition, reasoning／report characters, tool calls and elapsed time | Correctness, faithful reasoning, or quality merely because an answer is longer |
| Speed, 8 prompts | Five fixed holdout prompts plus three longer-output prompts; streaming at temperature 0 | Concurrent serving capacity, cold-start latency or end-to-end application performance |

know5、agent6 與 depth8 是不同題組，不能合併成單一 Accuracy。原始工具評測使用原環境的 SQL 與 RAG；此版只公開去識別化方法與彙總，不提供公司服務或原始查詢。

## Decode throughput, TTFT and task latency

- **Decode tok/s**：對有效 speed samples 加總 `(completion_tokens − 1)`，除以第一至最後 content／reasoning SSE chunk 的時間總和。aggregate 僅納入 tokens >16、有有效時間區間的樣本；短題觸及 2,048-token 上限的輸出被排除，避免重複長輸出灌高速度。
- **TTFT**：模型 HTTP 請求送出至第一個含 content／reasoning 的 SSE chunk。RAG context 在此前取回，因此不是完整應用端到端 latency，也不是逐 token kernel profiling。
- **Task latency**：agent6 表中的秒數是整組工具題耗時；depth 表則是已答完深度題的平均完整耗時。兩者與 speed gate 的 decode 不能互換。
- **Characters ≠ tokens**：reasoning／report 長度為字元數，資料集 token 數為 input tokens；都不能直接當成品質分數或 supervised-token 數。

原始 per-question 時間欄位經四捨五入，重算 aggregate 可有微小差異；保留原始 aggregate，不補造精度。沒有 raw SSE、重複輪次、p95／p99、併發或冷暖快取資料。

## Archived result summaries

**Series A · 2026-10-08 — deployment configurations**

| Configuration | Decode tok/s | Knowledge | SQL agent | SQL suite time |
|---|---:|---:|---:|---:|
| v0.4 runtime LoRA | 27.01 | 5/5 | 4/6 | 130 s |
| v0.4 merged + old draft | 45.14 | 4/5 | 5/6 | 130 s |
| v0.4 merged + new draft | 49.63 | 4/5 | 6/6 | 109 s |

**Series B · 2026-10-09–10 — analysis evaluation**

| Configuration | Decode tok/s | Knowledge | SQL agent | Analysis completed | Mean analysis latency |
|---|---:|---:|---:|---:|---:|
| v0.4 reference | 36.19 | 5/5 | 5/6 | 8/8 | 30.6 s |
| v0.5 + old draft | 38.53 | 5/5 | 6/6 | 8/8 | 333.6 s |
| v0.5 + new draft, final | 42.30 | 5/5 | 6/6 | 8/8 | 425.6 s |

來源為兩系列封存 gate JSON 的 aggregate 欄位，本次唯讀核對。v0.5 final 的平均報告 3,432.2 字元／12.5 次工具呼叫；v0.4 reference 為 401.1 字元／2.0 次。較長分析伴隨更長等待時間，沒有據此宣稱較高準確率。

## Comparability and scoring changes

1. **A 與 B 不直接比較**：來源敘事分別稱短 Prompt 與 deployed-context，但封存 speed 函式使用知識指令＋RAG，缺逐 run 的腳本版本與實際 request。Prompt 差異保留「待核對」，不由 49.63 vs 42.30 推論模型退步。
2. **配置比較不是因果消融**：BF16 merge、重新量化、kernel 路徑、target 輸出與 draft 都可能影響 decode；QLoRA 微調不直接等同推理加速，也未證明各因素的獨立貢獻。
3. **保留原始自動分數**：v0.4 final 知識題為 4/5；敘事中的人工「實質 5/5」不覆蓋 JSON。
4. **判分規則曾修改**：v0.5 舊 draft 原 gate 記 loops=2，後續選版重新判定為 0，因思考內重複 SQL 草稿不一定是失敗迴圈。保留兩種紀錄，不默默統一。選版速度 42.14 與 final gate 42.30 也屬不同階段。
5. **測試集隔離有限**：固定題曾反覆用於修補與選版；builder 對舊資料的相似題只記錄、不剔除。早期 Final Holdout 與後期 speed 問題檔不是已確認的同一集合，不能宣稱完全獨立泛化測試。

## Evidence strength and reproducibility

封存 JSON 能支持「所記錄的欄位數值」；靜態原始碼能支持「實作與檢查存在」，都不等同本次完整執行成功或獨立認證。敘事根因缺單因素消融時仍是分析假設。

Training time 約 v0.4 7.8 h／v0.5 11.7 h、訓練 peak 與記憶體 probe 等屬**封存紀錄值**，缺完整 logs／metrics。v0.5 build summary 記錄 5,134 rows、5,043,275 input tokens、103 條教師分析及 100 條教師知識；rows 含重複舊資料。v0.4 所報 2,552 與組成合計 2,555 不一致，仍待最終 manifest 核對。

來源名稱 Qwen3.8 與 config 的 Qwen3_5 架構命名仍待模型卡／snapshot／hash 核對；缺最終量化 report、完整 artifact hashes、runtime manifests 與逐 run Prompt 條件。不主張量化無損、全鏈路公開可重現或通用 Benchmark 勝出。

公開重現方案為規劃中：另經授權建立合成資料、mock SQL、獨立相容環境與版本鎖定。本次不執行模型、SQL 或部署，也未加入訓練程式、資料集與權重。

Read next: [Project README](../README.md) · [Engineering Journey](ENGINEERING_JOURNEY.md)。
