# Evaluation Method & Evidence Limitations

> 2026-10-12 技術修訂 · 根據封存結果及原開發 Agent 的事後原件查核整理，不是本機重跑報告。原始題目、企業 SQL、工具回傳、測試 JSON、部署設定與私人證據索引不在本作品內。

## What the Golden Set measures

| Gate | Method | What the result does not establish |
|---|---|---|
| Knowledge, 5 tasks | Required／forbidden regex checks; three tasks concern deliberately trained topics | Independent held-out accuracy; regex may reject a semantically valid answer |
| SQL agent, 6 tasks | Compare expected numbers／names; check task completion and structured tool calls | Reliability across all enterprise queries or unseen schemas |
| Analysis, 8 tasks | Record final-report completion, repetition, characters, tool calls and elapsed time; no content-correctness scoring | Accuracy, faithful reasoning, or quality merely because an answer is longer |
| Speed, 8 prompts | First five questions from a fixed 30-question pool plus three longer-output questions; one run per configuration | Final Holdout scoring, concurrent capacity, cold-start or end-to-end performance |

know5、agent6 與 depth8 是不同題組，不能合併成單一 Accuracy。原始工具評測使用原環境的 SQL 與 RAG；此版只公開去識別化方法與彙總，不提供公司服務或原始查詢。

兩系列速度都使用同種Prompt：system為知識指令＋RAG檢索結果，user為問題，不讀部署System Prompt。部署Prompt用於固定SQL六題及深度題。前五題max_tokens=2048，三題約600字長輸出的max_tokens=800；單台GB10、單串流、temperature=0，每配置只量一次。固定30題速度問題池與134題Final Holdout是不同集合，不混稱。

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
| v0.5 + new draft, earlier-parser final gate | 42.30 | 5/5 | 6/6 | 8/8 | 425.6 s |

來源為封存gate JSON的aggregate欄位。v0.5上表final與深度結果使用舊parser `deepseek-r1`；改為`qwen3`後的存檔回歸為39.83 tok/s、know5 5/5、agent6 6/6，深度題未重跑。舊parser深度題平均報告3,432.2字元／工具12.5次；v0.4 reference為401.1字元／2.0次。六題SQL全部通過是特定題組表現，八題報告答完是完成率，兩者都不是通用準確率；更長輸出不代表更高品質。

## Comparability and scoring changes

1. **相同Prompt組法不代表條件受控**：已查核的三版speed函式均使用知識指令＋RAG、不含部署Prompt。49.63那次的確切腳本只能由備份及輸出欄位推定，完整request與SSE未存。同一v0.4合併模型的49.63與36.19測量，第8題分別為800／90 tokens；49.63那次該題為61.59 tok/s，v0.5 final多題輸出536–800 tokens。指標以token加權，輸出差異與速度差同時出現，但沒有控制實驗證明原因。不能把單次觀測範圍當信賴區間，也不能宣稱v0.5較快。
2. **配置比較不是因果消融**：BF16 merge、重新量化、kernel 路徑、target 輸出與 draft 都可能影響 decode；QLoRA 微調不直接等同推理加速，也未證明各因素的獨立貢獻。
3. **保留原始自動分數**：v0.4 final知識題為4/5；其中一題因必要關鍵詞未涵蓋回答使用的同義說法而失敗，人工判讀正確。自動分數與人工覆核分開，不把JSON改成5/5。
4. **判分規則曾修改**：v0.5 舊 draft 原 gate 記 loops=2，後續選版重新判定為 0，因思考內重複 SQL 草稿不一定是失敗迴圈。保留兩種紀錄，不默默統一。選版速度 42.14 與 final gate 42.30 也屬不同階段。
5. **Parser回歸範圍有限**：42.30是舊parser final gate，39.83是新parser回歸測速，不能直接當parser效能對照。更新後只有知識與固定SQL題有存檔品質回歸；深度未重跑，思考開關／串流／tool call矩陣只有文字紀錄。

## Dataset isolation and the unscored Final Holdout

- 新產生資料與保護題做字元bigram Jaccard，≥0.6剔除；v0.4／v0.5各剔除一筆。保護池包括固定驗收、30題速度問題池、知識與SQL題；v0.5另加深度與11題回歸題。
- 舊資料不在剔除範圍：v0.4舊A有92筆與保護池相似度≥0.6，只記錄、不剔除，且以×2進入v0.5。這個92是建置時對保護池的結果，**不是134題Final Holdout的近似重疊統計**。
- 對Final Holdout的事後唯讀完全相同user問題比對：**3/134（2.2%）**與訓練資料完全相同。v0.1原始資料無完全重疊；v0.2修補首次帶入，v0.3再次包含並延續至v0.5。
- Final Holdout的**近似重疊統計為NOT TESTED**，沒有補造數字；該134題至今尚未用於模型評分。尚未評分不等於資料獨立，不能稱無洩漏holdout。
- 知識五題中三題是刻意訓練過的主題，另有一題依靠RAG；固定題也反覆用於修補與選版，不是獨立held-out準確率評測。

Final Holdout若未來用於正式評分，需先處理已重疊題並另外完成近似比對；本次沒有修改資料集或新增評分。

## Evidence strength and reproducibility

原開發Agent已對照原件確認五版訓練summary／metrics／log、資料計數、模型來源、量化配置與serving版本；本次採去識別化查核結論，沒有重新接觸原機或執行訓練。這解除先前因經驗包未附完整檔案而保留的待核對，不等於外部獨立重現，也不代表原件可公開分發。

v0.4訓練7.76 h（27,931.6 s）、638 steps／2 epochs；v0.5 11.68 h（42,037.1 s）、642 steps／1 epoch，有完成原件支持，仍是封存實測而非本機重現。v0.5為5,134 rows、5,043,275 input tokens，其中supervised tokens 1,429,800；教師分析154合格→抽120→長度剔除17→103，另有教師知識100。v0.4實際2,552：名目2,555中改寫缺2、相似題剔除1；計數已釐清。

空reasoning為事後唯讀重算v0.4全體2,652 assistant turns，1,913個缺少或去空白後為空（72.1%）；非空回合中位數29字元。不是rows比例，也不是當時已存的統計。Chunked/full CE單一樣本均0.345422、rel=0.00e+00；樣本長度未記，不能擴大為多樣本等價性。最長row probe 78.04 GiB與實際訓練峰值80.54 GiB分開記錄。

五版adapter來自uniform NVFP4技術驗證backbone（`NVFP4_DEFAULT_CFG`）；服務端另做BF16 merge與混合精度量化。v0.5實際校準256筆領域訓練資料／85,579 tokens，Local-Hessian 555.8 s、無max fallback，208 FP8＋193 NVFP4的層級配置與官方比對相符；v0.4校準tokens為81,667，樣本數未確認。這不是官方校準集或相同權重。

Qwen3.8 repo身分與`qwen3_5`實作類別已確認，官方命名原因仍未找到。仍缺mask逐token邊界報告、多樣本／多長度CE對照、因果消融、raw SSE、冷暖cache、多輪及併發測試；v0.2續跑原因原件及思考開關矩陣未完整確認。不主張量化無損、通用Accuracy、v0.5較快、全鏈路公開重現或正式Production。

v0.5只是依事先授權自動規則採用的**Adopted Candidate／內部試用**。Technical Smoke Backbone及其adapter的原Promotion禁令未找到正式解除決議；BF16 merge不自動解除。沒有Final Holdout評分、Production批准或Promotion決議。模型核准與作品公開權利是兩個獨立議題；展示技術說明不構成模型Promotion，也不代表公司授權或第三方權利已取得。

公開重現方案為規劃中：另經授權建立合成資料、mock SQL、獨立相容環境與版本鎖定。本次不執行模型、SQL 或部署，也未加入訓練程式、資料集與權重。

Read next: [Project README](../README.md) · [Engineering Journey](ENGINEERING_JOURNEY.md)。
