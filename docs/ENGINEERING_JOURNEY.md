# Engineering Journey：v0.1 → v0.5

> Portfolio v0.1，2026-10-12。版本號指原始模型實驗；本文件版本不等於模型 v0.1。所有實驗為封存紀錄，本次未執行。評測方法與證據限制見 [Evaluation Method](EVALUATION_METHOD.md)。原始證據與內部審查紀錄私下保存，不隨此作品發布。

## 1. v0.1：先確認訓練到 Serving 的機制

**狀態：已完成機制 POC／Candidate，非正式驗收。** Dataset manifest 記錄 1,850 rows、251,684 tokens、1 epoch，未納入 dev／validation；STATUS 明確標示非 Production。early backbone 為訓練專用 NVFP4，而參考部署 config 是混合精度；這兩種產物不可混用。

第一階段的價值是把 freeze backbone、PEFT adapter、assistant mask、HF export、SGLang wrapper 及 adapter 選擇串接起來。原始敘事記載 shape mismatch、ConditionalGeneration wrapper、融合 target module 命名、LoRA selector 與 kernel fusion 相容性問題；解法是分開 canonical PEFT adapter 與 serving 格式，並用 logprobs 確認 adapter 確實生效。對文字相同的回答，不能僅由肉眼推斷 adapter 未載入。

**限制**：訓練 preflight report、canonical adapter 和 lossless export report 沒有附入包內；部分相容性修正只有報告與程式片段支持。原始 5 題出現三項錯誤的描述也只是特定問法，不是泛化能力分數。

## 2. v0.2：機制通過不代表修補成功

**狀態：已完成實驗，品質混合／不宜宣稱修復。** 報告記錄 1,875 rows、235 steps、1 epoch，診斷題自動分數 0/15，新增矛盾、欄位幻覺與負遷移。playbook 把 36 筆稱為新增修補，但 1,850＋36 不等於 1,875；build 程式包含既有回答修改，需以最終 manifest 區分「改動」與「淨新增」，本版不宣稱新增 36 筆。

**失敗 → 分析 → 下一版**：少量定向資料沒有穩定抑制模型通用常識；新增描述還被過度套用到不相關概念。報告將訊號密度不足列為可能原因，不能改寫成已證明的唯一原因。下一版擴大問法與回歸題，並增加 epoch。

另一次操作失敗是把背景 stdout 停止轉發誤判為訓練 hang，提前終止程序，再由 checkpoint resume。附帶 resumed log 只到 step 26，支持續跑片段，不能單獨證明全程完成。操作教訓是對照 metrics、checkpoint 與程序狀態，而非只看轉發的 stdout。

## 3. v0.3：低 Loss 與回答可靠性分開看

**狀態：已完成候選實驗，仍有退化與矛盾。** launch log 記錄 1,913 rows、2 epochs、479 optimizer steps，但只保留到 step 6。報告稱約 11.4 h、train loss 前後區間 2.32→0.33、final 0.011、dev loss 0.515；完整 training summary 與 dev report 未附，這些屬部分支持。

原始五題人工比較稱兩題首次修復，15 題診斷自動分為 4/15；報告亦指出換問法後自相矛盾、幻覺改換形式、重複退化。故不能以更低 loss 宣稱領域品質已通過正式驗收，更不能跨 mask／資料版本直接比較 loss。

後續工程分析發現：早期監督主要落在 `</think>` 後，推論卻由含 `<think>` 的 generation prefix 開始。終止 reasoning 的行為缺少對應監督是合理根因假設；v0.4 同時改資料、mask、Prompt 與工具情境，沒有單獨消融量測，不能把全部改善歸於一項修正。

## 4. 推理優化支線：不是 LoRA 訓練能力測試

來源記錄無 speculative decoding 約 11–12 tok/s、借用 draft 約 21、warm-start 約 30.12，另有從零 draft 訓練與 NEXTN/MTP 失敗。這些早期速度缺原始 sweep／SSE 結果，僅作部分支持的工程紀錄。

| 失敗／問題 | 來源分析 | 修改與可陳述結論 |
|---|---|---|
| 小量蒸餾資料、從零 draft 訓練變慢 | 過擬合；另一輪缺 `draft_checkpoint_path` | draft config 顯示明確 warm-start；對「資料量不足」的泛化結論仍需對照試驗 |
| teacher-forced accepted length 好、真實速度差 | 訓練代理指標與 decode workload 不一致 | 以掛上 target 的實際串流速度選 checkpoint，不只看訓練指標 |
| sweep 看似換了 draft 卻結果不變 | command／volume 指向不一致 | 比對實際掛載與完整 hash；本包未提供完整 hash 清單 |
| SpecForge 與 serving 版本相衝突 | 環境綁定不同 SGLang 版本 | capture／draft 訓練與 serving 分開環境，完整 image digest 待補 |

## 5. v0.4：格式、Loss、產物與評測一起修正

**狀態：候選工程與 gate 結果有證據，正式 Release 權利／驗收待確認。**

1. **資料與推論對齊**：修正舊定義、加入改寫題、短 reasoning、完整 system／工具規格與工具結果，並加入不需查詢的題目。builder 有 bigram Jaccard 0.6 相似度檢查，但舊 A 類資料只記錄相似題、不剔除。不能說完全排除所有 train/test leakage。
2. **Loss 邊界**：v0.4 資料編碼器 分段 tokenize，generation prefix 之前遮罩，prefix 後到 `<|im_end|>` 監督；`_no_loss` 保留失敗回合但不學錯誤回應。程式支持實作存在，未在本機執行 encoder。
3. **訓練成本**：trainer 支持 token-budget micro-batch、8 rows／optimizer step、shuffle／resume。playbook 稱 fla＋合批由 254 s／step 降到約 44 s，但缺同條件 profile 與完整兩組 logs；不能當已驗證 5.8× 加速。
4. **部署產物**：手動把 LoRA delta 加回 BF16，再以 FP8 attention、NVFP4 MLP／lm_head 配方量化。參考 config 的 208 FP8／193 NVFP4 層可重算；最終合併量化報告未附，不能說最終產物已核對完全相同。校準有 Local-Hessian 與 max fallback，也需要 report 確認實際走哪條。
5. **Draft 與測量**：合併後 target 產生蒸餾輸出、warm-start draft，再測 gate。JSON 記錄 runtime LoRA 27.01、merged old draft 45.14、final 49.63 decode tok/s；比較是整個配置，不能把提升完全歸於 adapter merge。

第一次 pipeline 出錯的紀錄包括：以 proxy 合成 `/v1/models` 作就緒判斷，以及用 tokens／總時間把短答的 TTFT 與長迴圈混入速度。來源碼改成真實 completion 就緒檢查與串流 decode 指標。這是驗測設計修正，不是「新版模型一定較快」的證明。

final JSON know5 是 4/5；文件的「實質 5/5」為人工語意覆核說法，兩者必須並列。agent6 final 6/6，但系列中同一題會波動；不能宣稱業務查詢正確率 100%。

## 6. v0.5：恢復較長分析，也接受 Latency 代價

**狀態：原始 decision 已採用 Candidate；不是 Production。** 使用者回饋 v0.4 快而缺分析深度；來源稱空 reasoning 占 72%，原始統計缺件。v0.5 的改法是對舊資料遮罩 reasoning，對教師深度軌跡保留完整監督。這不是要求所有問題都長思考，而是改變訓練訊號。

### 教師資料的過濾與配比

封存的教師資料建置摘要 記錄分析 accept 154／reject 63，後續 cap 與最長 row 過濾留下 103；知識 accept 151，最後收 100。拒絕項包括違反領域規則、數字不符、長度與完成狀態，拒絕原因有重疊。最終各分部 rows 加總 5,134、tokens 加總 5,043,275，103 條分析占 2,904,583 input tokens（約 57.6%，由原始計數計算）。這是 input tokens，不是所有 supervised tokens。

這個比例說明長軌跡對 token budget 的影響，並不能證明教師 trace 的每段推理正確或 faithful。teacher judge、數字 matching 與 SQL grounding 仍需人工抽驗及完整紀錄。

### 長 Row 與單機記憶體

builder 依 pipeline 使用 48,000 row cap；trainer guard 為 49,152，pipeline override 1 epoch，預設 2 epochs 不能當實際設定。程式在長 supervised positions 分段 lm_head＋CE，使用 checkpoint 重算避免保留整段 vocabulary logits；包含與 full CE 比對及最長 row backward probe。原始 probe 78 GiB、rel=0、642 steps／11.7 h 由部署紀錄與 playbook 支持，但缺完整訓練 log，仍不能列為本機驗證。

### 品質、深度與速度分開驗收

final JSON 為 know5 5/5、agent6 6/6、depth answered 8/8、loops 0，speed 42.30 decode tok/s。depth 平均 reasoning 21,866.6 字元、報告 3,432.2 字元、工具 12.5 次、425.6 s；對照 v0.4 深度題只需 30.6 s。較長分析有明顯延遲成本，不該包裝成全面品質或效率勝出。

來源將此系列標為完整 deployed-context 量法，但封存的speed函式仍組合知識指令與RAG，未直接載入deployed Prompt。缺逐run版本與request紀錄，無法確認速度部分的Prompt差異；兩系列分列，不跨系列推論因果。

封存選版紀錄 用於選版的速度為 42.14，final gate 為 42.30，是不同階段結果。old-draft gate `loops=2`，decision 源碼重新依最終報告／未答完狀態判斷，summary 改記 0；應保存兩版判分依據，不覆寫原始結果。深度採用規則還容許 agent6 比 reference 少一題（但至少 4）與 decode ≥0.80× reference，不是所有版本都採用 ≥1.10× 加速門檻。

### Serving 收尾修正

- proxy timeout 300→1,800 s：程式支持修正存在；長非串流單輪可能沒有位元組輸出，應與整題 wall time 分開看。
- buffering 使用 `read1()`＋flush，結束連線避免前端一直等待；這是 SSE 傳輸處理，不是 decode kernel 加速。
- `deepseek-r1`→`qwen3`，另設 `qwen3_coder` tool parser。部署檔支持新設定；關思考／開思考再測只有敘事，獨立結果待補。

這些都僅作原始服務的歷史紀錄，本作品沒有啟動或更改服務。

## 7. 面試討論與 AI 輔助工程

可使用本歷程討論：為何區分 frozen quantized backbone 與 serving checkpoint、mask 監督哪些 token、長軌跡如何影響記憶體、如何處理 regex 判分偏差、如何選 draft，以及深度與 latency 如何驗收。

使用者已確認角色為 Project Owner，負責需求、技術方向、方案選擇、使用回饋及報告結果審閱。Claude Code 負責大部分程式、訓練腳本、環境設定、執行與除錯；使用者沒有親自撰寫底層訓練程式。Codex 隨後整理封存證據與作品文件。完整已確認聲明見 [README](../README.md#6-my-contributions--ai-assisted-development)。

公開重現、額外 Benchmark 與泛化能力消融均為規劃中。方法及缺件見 [Evaluation Method](EVALUATION_METHOD.md)；公開權利仍需另行確認。
