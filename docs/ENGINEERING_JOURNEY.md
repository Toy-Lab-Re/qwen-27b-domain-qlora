# Engineering Journey：v0.1 → v0.5

> 2026-10-12 技術修訂。版本號指原始模型實驗。原開發 Agent 已對照原始資料、summary、metrics 與 log 做事後唯讀查核；本版依去識別化結論修正，沒有重跑實驗。原始證據與內部審查紀錄私下保存。評測限制見 [Evaluation Method](EVALUATION_METHOD.md)。

## 封存訓練記錄與模型來源

| Version | Dataset rows | Epochs | Optimizer steps | Recorded training time | Mean loss: first → last 10% steps | Recorded training memory peak |
|---|---:|---:|---:|---|---|---:|
| v0.1 | 1,850 | 1 | 232 | 5.54 h | 2.44 → 1.72 | 23.8 GiB |
| v0.2 | 1,875 | 1 | 235 | 5.14 h | 2.45 → 1.62 | 24.6 GiB |
| v0.3 | 1,913 | 2 | 479 | 11.45 h | 2.32 → 0.33 | 23.8 GiB |
| v0.4 | 2,552 | 2 | 638 | 7.76 h（27,931.6 s） | 2.44 → 0.34 | 38.6 GiB |
| v0.5 | 5,134 | 1 | 642 | 11.68 h（42,037.1 s） | 2.31 → 0.38 | 80.54 GiB |

這些是原始訓練 summary、逐步 metrics 與 log 的查核值，不是本機重現。v0.2 metrics 有 238 列，step 21–23 各重複一次，故 unique optimizer steps 是 235；續跑原因的完整原始紀錄仍未確認。v0.2／v0.3 舊 run_config 未更新 rows／epoch，採 summary 與 metrics 而非過期設定。v0.4／v0.5 log 均有完成標記；final loss 分別 0.40／0.64。記憶體峰值是訓練記錄指標，不是整台統一記憶體的總用量。

確認的 BF16 base 為 `Qwen/Qwen3.8-27B`，revision `1d4bf0f2ff6012fd82039f2fa52739d0dd7c60c0`，18 shards。架構類別為 `Qwen3_5ForConditionalGeneration`／`qwen3_5`；文字模型共 64 層（48 linear-attention＋16 full-attention）、hidden size 5,120、vocab 248,320。repo 模型識別與實作類別名稱不同，不能據類別改稱 Qwen3.5；官方命名原因未找到，不自行解釋。

量化比對對象是 `nvidia/Qwen3.8-27B-NVFP4`，revision `482ca0f3832238542f8f5295dde86b5f22711d80`。本案 serving 產物由自己的 adapter 合併與重新校準產生，並加回 base 原樣的 333 個 vision tensors；層級配置相符不表示權重與官方相同，也不證明量化無損。

## 1. v0.1：先確認訓練到 Serving 的機制

**狀態：已完成機制 POC／Candidate，非正式驗收。** Dataset manifest 記錄 1,850 rows、251,684 tokens、1 epoch；optimizer steps 已確認為 232。五版 v0.1～v0.5 都沿用同一個 Technical Smoke Backbone：BF16 base 以 ModelOpt `NVFP4_DEFAULT_CFG`、32 筆校準建立的 uniform NVFP4 訓練專用 backbone。它沒有採用服務端的 FP8 attention＋NVFP4 混合精度配置。

原治理禁止把此 backbone 作為 Production base、部署到正式推論服務，亦禁止由它訓練出的 adapter Promotion。後續採 BF16 merge 再量化，不是直接部署 backbone；但未找到正式解除決議，合併產物仍保守視為未獲 Promotion 核准。此限制不妨礙描述已完成的技術驗證與內部 Candidate 試用，但不能宣稱正式發布。

第一階段串接 freeze backbone、PEFT adapter、assistant mask、HF export、SGLang wrapper 與 adapter 選擇。曾遇到 shape mismatch、wrapper、融合 module 命名、selector 與 kernel fusion 相容性問題；解法是分開 canonical adapter 與 serving 格式，並以 logprobs 比對 adapter 是否生效。匯出報告中的「48 層」指 48 個 linear-attention 層，把其 LoRA `in_proj_qkv` 拆成 q/k/v，最大絕對誤差 0.0、無孤立 key；模型總共仍是 64 層。此格式匯出用於 v0.1～v0.4 runtime LoRA serving（驗證環境 SGLang 0.5.20），合併版不使用這條 adapter 匯出路徑。

**限制**：原件已由開發 Agent 查核，但沒有隨公開作品分發。原始五題的錯誤與改善描述只針對特定問法，不是泛化分數；單次 logprobs 或格式匯出驗證也不代表完整服務可靠性。

## 2. v0.2：機制通過不代表修補成功

**狀態：已完成實驗，品質混合／不宜宣稱修復。** 在 1,850 筆上原地修改 11 筆、新增 25 筆，實際為 1,875 rows；「36」是變更總數，不是淨新增。訓練為 235 steps、1 epoch、5.14 h。診斷題自動分數 0/15，並出現矛盾、欄位幻覺與負遷移。

**失敗 → 分析 → 下一版**：少量定向資料沒有穩定抑制模型通用常識；新增描述還被過度套用到不相關概念。報告將訊號密度不足列為可能原因，不能改寫成已證明的唯一原因。下一版擴大問法與回歸題，並增加 epoch。

早期敘事記載把背景 stdout 停止轉發誤判為 hang，提前終止程序再 resume。metrics 的重複 step 與 summary 支持續跑及完成記錄，但這次查核仍未找到可完整確認續跑原因的原件；不能把敘事當成已證實的操作因果。教訓仍是對照 metrics、checkpoint 與程序狀態，而非只看 stdout。

## 3. v0.3：低 Loss 與回答可靠性分開看

**狀態：已完成候選實驗，仍有退化與矛盾。** 1,850 筆＋63 筆新增（另有 11 筆原地修改）＝1,913 rows；2 epochs 共 3,826 列、479 steps、11.45 h。原件確認 loss 前／後 10% steps 平均為 2.32／0.33；最後一步 loss 0.011，但該步只有 2 筆資料，不能單憑此值判定過擬合或背答案。較低訓練 loss 不等於品質或泛化改善，資料與監督範圍變更時也不宜直接跨版比較。早期敘事的 dev loss 0.515 不當成這次新查核的獨立泛化證據。

原始五題人工比較稱兩題首次修復，15 題診斷自動分為 4/15；報告亦指出換問法後自相矛盾、幻覺改換形式、重複退化。故不能以更低 loss 宣稱領域品質已通過正式驗收，更不能跨 mask／資料版本直接比較 loss。

後續工程分析發現：早期監督主要落在 `</think>` 後，推論卻由含 `<think>` 的 generation prefix 開始。終止 reasoning 的行為缺少對應監督是合理根因假設；v0.4 同時改資料、mask、Prompt 與工具情境，沒有單獨消融量測，不能把全部改善歸於一項修正。

## 4. 推理優化支線：不是 LoRA 訓練能力測試

來源記錄無 speculative decoding 約 11–12 tok/s、借用 draft 約 21、warm-start 約 30.12，另有從零 draft 訓練與 NEXTN/MTP 失敗。這些早期速度缺原始 sweep／SSE 結果，僅作部分支持的工程紀錄。

| 失敗／問題 | 來源分析 | 修改與可陳述結論 |
|---|---|---|
| 小量蒸餾資料、從零 draft 訓練變慢 | 原敘事推測過擬合；另一輪缺 `draft_checkpoint_path` | 明確 warm-start；過擬合與資料量不足仍需對照試驗 |
| teacher-forced accepted length 好、真實速度差 | 訓練代理指標與 decode workload 不一致 | 以掛上 target 的實際串流速度選 checkpoint，不只看訓練指標 |
| sweep 看似換了 draft 卻結果不變 | 原敘事指向 command／volume 不一致 | 檢查實際掛載與權重 hash；不將敘事當成獨立性能驗證 |
| SpecForge 與 serving 版本相衝突 | SpecForge 相依 SGLang 0.5.18，serving 使用 0.5.21 | capture／draft 訓練與 serving 分開環境 |

查核確認三代 warm-start 系譜：`incoai/Qwen3.8-27B-DFlash2` → v0.3 step450 → v0.4 merged step450 → v0.5 step300。v0.5 step150／300／450 選版測速為 38.3／42.14／42.14 tok/s；採用 step300，之後 final gate 為 42.30，兩種測量不混用。SpecForge 的封存 commit 前綴為 `53398a8f`，draft 訓練使用自建 image；這不是已提供的公開重現環境。

Serving image 已核對為 `lmsysorg/sglang:v0.5.21-cu130`，容器內 SGLang 0.5.21；RepoDigest 是 `sha256:b1259f3ea3275f66237c498ea388919729018bc9f01c3d638391e06e2cf3f469`，不能以 Image ID 當 digest。運行條件為 DFLASH、8 draft tokens、KV `fp8_e4m3`、mem-fraction 0.80、tool parser `qwen3_coder`；reasoning parser 的日期與評測差異見下文。

## 5. v0.4：格式、Loss、產物與評測一起修正

**狀態：候選工程與 gate 結果有證據，非正式 Production Release，未獲 Promotion 核准。**

1. **資料與推論對齊**：修正舊定義並加入改寫、短 reasoning、工具軌跡及不需查詢的題目。名目 A 1,913＋B 168×3＋C 28＋D 80＋E 30＝2,555；一題改寫回傳零個變體令 B 少 2 成為 502，C 相似度過濾少 1 成為 27，實際 2,552。A 另有 17 筆原地修正。summary、檔案行數與 build report 一致。新資料有 bigram Jaccard ≥0.6 過濾，舊 A 的 92 筆相似題只記錄、不剔除，不能宣稱無洩漏。
2. **Loss 邊界**：v0.4 資料編碼器 分段 tokenize，generation prefix 之前遮罩，prefix 後到 `<|im_end|>` 監督；`_no_loss` 保留失敗回合但不學錯誤回應。程式支持實作存在，未在本機執行 encoder。
3. **訓練成本**：同硬體、同資料順序、每 optimizer step 8 rows，舊 batch=1／無 fla／不合批只跑 4 步：254.6／282.0／101.0／114.6 s；fla 0.5.2＋token 合批 4096 的前 4 步為 94.0／97.0／20.2／16.5 s。逐步比值約 2.7–6.9×，兩項修改同時進行，不能拆出各自貢獻。新設定全程平均 43.8 s/step（訓練迴圈記錄 27,923 s ÷638）；這與 summary 的 27,931.6 s 是不同計時統計。不得把舊第 1 步與新全程平均當同口徑加速比較。
4. **部署產物**：400 個 LoRA modules 以 scale 2.0 合併回 BF16，平均相對 delta norm 為 v0.4 0.0071／v0.5 0.0069。量化是 Local-Hessian，報告確認沒有 max fallback；401 個量化層為 208 FP8＋193 NVFP4，與官方參考逐類精度配置相符，不是權重相同。193＝64×3 MLP＋lm_head，208＝48×3 linear-attention＋16×4 full-attention。v0.4 合併至匯出約18分鐘、calibration tokens 81,667；其實際校準筆數未確認，不由 v0.5 推測。
5. **Draft 與測量**：對合併後 target 蒸餾並 warm-start draft。原始 gate 記錄 runtime LoRA 27.01、merged old draft 45.14、final 49.63 decode tok/s；同一 v0.4 合併模型隔天為 36.19。兩次第8題各輸出800／90 tokens，差異與輸出長度同時出現，沒有單變因控制、不能證明速度差原因。比較是整個配置，不將提升全部歸於 merge。

v0.5 量化的請求值512不是實際樣本數：實際只取領域訓練資料256筆、85,579 tokens，預期另一半蒸餾資料未加入；Local-Hessian量化耗時555.8 s。這不是 NVIDIA 官方校準集，不宣稱官方權重相同或品質無損。

第一次 pipeline 出錯的紀錄包括：以 proxy 合成 `/v1/models` 作就緒判斷，以及用 tokens／總時間把短答的 TTFT 與長迴圈混入速度。來源碼改成真實 completion 就緒檢查與串流 decode 指標。這是驗測設計修正，不是「新版模型一定較快」的證明。

final JSON know5 是4/5，其中一題必要關鍵詞規則未涵蓋回答的同義說法，人工判讀正確；不覆寫自動分數。agent6 6/6是固定六題通過，不能宣稱通用查詢準確率100%。

## 6. v0.5：恢復較長分析，也接受 Latency 代價

**狀態：Adopted Candidate，僅內部試用；未經 Production 驗收或 Promotion 核准。** 自動流程依使用者事先授權的採用規則於2026-10-10採用，不等於正式核准，也未對134題 Final Holdout評分。

使用者回饋 v0.4 快而缺分析深度。對封存 v0.4 最終2,552 rows事後唯讀重算：所有assistant回合2,652個，`reasoning_content`缺少或去空白後為空的回合1,913個，比例72.1%；非空回合中位數29字元。空回合對應原本沒有reasoning的1,913筆舊格式資料。當時統計未存檔，本值不是同期原始統計。v0.5對舊資料遮罩reasoning、對教師資料保留完整監督；「空reasoning令分析變短」與「mask修好重複」仍是根因假設，沒有消融實驗。逐token邊界驗證報告也未找到；合批同筆loss檢查rel diff 0.018 <0.10不代替token級驗證。

### 教師資料的過濾與配比

教師分析217條候選→合格154→隨機抽120→剔除超過48k tokens的17→103；教師知識207→合格151→抽100。分析初篩拒絕63條，原因包括違反領域規則27、標準數字不符21、報告長度14、未答完7、數字缺少工具結果支持3、簡體字2、與保護題相似1；原因可重疊，不加總為唯一拒絕數。

| Part | Final rows |
|---|---:|
| Legacy A | 1,913×2＝3,826 |
| B | 502×2＝1,004 |
| C | 27×2＝54 |
| D | 32（80中48被教師資料取代） |
| E | 15（30中15被取代） |
| Teacher knowledge | 100 |
| Teacher analysis | 103 |
| Total | 5,134 |

總input tokens為5,043,275；其中計loss的supervised tokens為1,429,800。103條分析占2,904,583 input tokens（57.6%），不是supervised-token占比；rows含重複，不能視為5,134個獨立新案例。最長row為46,410 tokens。

這個比例說明長軌跡對 token budget 的影響，並不能證明教師 trace 的每段推理正確或 faithful。teacher judge、數字 matching 與 SQL grounding 仍需人工抽驗及完整紀錄。

### 長 Row 與單機記憶體

builder cap為48,000，trainer guard為49,152，pipeline override為1 epoch。長supervised positions分段lm_head＋CE並checkpoint重算，避免保留整段vocabulary logits。原始log確認chunked/full CE均為0.345422、rel=0.00e+00，但只比對一個樣本且沒有記錄該樣本長度，不是多長度等價性驗證。46,410-token最長row forward/backward probe為78.04 GiB；全程訓練峰值80.54 GiB，兩者不同。642 steps、11.68 h有完成原件支持，仍非本機重現。

### 品質、深度與速度分開驗收

舊parser `deepseek-r1`下的final JSON為know5 5/5、固定agent6 6/6、depth answered 8/8、loops 0，speed 42.30 decode tok/s。depth平均reasoning 21,866.6字元、報告3,432.2字元、工具12.5次、425.6 s；v0.4 reference平均30.6 s。8/8只評產出最終報告，沒有內容正確性評分；較長分析有延遲成本，不包裝成全面品質或效率勝出。

兩系列speed函式的Prompt均是知識指令＋RAG，不含部署System Prompt；部署Prompt用於SQL與深度題。每配置只測一次，輸出長度不同；v0.4測得36.19／49.63，v0.5舊parser final 42.30、新parser回歸39.83 tok/s。這是單次觀測範圍，不是可靠性能區間，不能判定v0.5較快。早期49.63那次的確切執行腳本未完整保存，仍缺完整request與raw SSE；不再把限制說成已證實的Prompt差異。

封存選版紀錄 用於選版的速度為 42.14，final gate 為 42.30，是不同階段結果。old-draft gate `loops=2`，decision 源碼重新依最終報告／未答完狀態判斷，summary 改記 0；應保存兩版判分依據，不覆寫原始結果。深度採用規則還容許 agent6 比 reference 少一題（但至少 4）與 decode ≥0.80× reference，不是所有版本都採用 ≥1.10× 加速門檻。

### Serving 收尾修正

- proxy timeout 300→1,800 s：程式支持修正存在；長非串流單輪可能沒有位元組輸出，應與整題 wall time 分開看。
- buffering 使用 `read1()`＋flush，結束連線避免前端一直等待；這是 SSE 傳輸處理，不是 decode kernel 加速。
- `deepseek-r1`→`qwen3`，tool parser為`qwen3_coder`。2026-10-10的存檔回歸支持know5 5/5、固定agent6 6/6、39.83 decode tok/s；深度題未重跑。xhigh／medium／low／false／none思考開關、串流與tool call矩陣只有文字檢查紀錄，原件未找到，不宣稱完整驗證。

這些都僅作原始服務的歷史紀錄，本作品沒有啟動或更改服務。

## 7. 面試討論與 AI 輔助工程

可使用本歷程討論：為何區分 frozen quantized backbone 與 serving checkpoint、mask 監督哪些 token、長軌跡如何影響記憶體、如何處理 regex 判分偏差、如何選 draft，以及深度與 latency 如何驗收。

使用者已確認角色為 Project Owner，負責需求、技術方向、方案選擇、使用回饋及報告結果審閱。Claude Code 負責大部分程式、訓練腳本、環境設定、執行與除錯；使用者沒有親自撰寫底層訓練程式。Codex 隨後整理封存證據與作品文件。完整已確認聲明見 [README](../README.md#6-my-contributions--ai-assisted-development)。

公開重現、額外 Benchmark 與泛化能力消融均為規劃中。方法及缺件見 [Evaluation Method](EVALUATION_METHOD.md)；本作品的展示範圍不包含企業原始資料或部署設定。
