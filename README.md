# Context-Aware Outfit Recommendation

主要流程：

1. 資料建構：整理 PO-D 資料範圍、CLO / MET / temperature proxy、生成描述、LLM 評分清單與 W/O/S 三因子分割。
2. 模型訓練和驗證：保留主模型 Hybrid Attention 的 CP / CIR 程式與最後採用 checkpoint；CIR 由對應 CP checkpoint 初始化後訓練。另保留 Two-Tower 第二模型。
3. 實驗與結果：依期刊採納的實驗脈絡整理文字長度、target clue leakage、主推薦任務、W/O/S 消融、第二模型、反事實與人工稽核。各實驗資料夾內的 `圖表_figures_tables/` 保存該實驗對應的正式圖表資料。
4. 文件資料：保存期刊版本與碩論終稿。

大型原始資料如 Polyvore 圖片、完整 FashionCLIP 特徵與每個 epoch checkpoint 不納入；本資料夾只保留最後採用的模型、程式、正式圖表與論文/期刊採納的重點結果。

## 2026 reproduction artifact

學姊原始研究內容保留在：

- `01_資料建構_data_construction/`
- `02_模型訓練和驗證_model_training_validation/`
- `03_實驗與結果_experiments_results/`
- `04_文件資料_documents/`

上述四個資料夾在 2026 重現整理中維持相對於 frozen baseline
`7a5cc9cd8f884865e1e27a234c18dd132f949598` 不變。

新增的重現程式、設定、固定 ID、環境資訊、reference results 與 provenance
文件統一放在：

`reproduction/`

重現與交接請從：

`reproduction/README.md`

開始閱讀。

修訂稿的正式結果來自 clean-room run `full_20261002T161428Z`：從 GitHub 全新
clone、依 `reproduction/README.md` 建立環境後以 `reproduce_all.sh --fresh`
完成 35 組訓練（10 組主實驗、25 組公平子集消融），狀態 `PASSED`。它與先前的
reference run（`reference_20260921T175217Z`）及重構後的 `full_20260924T102513Z`
逐位元相同。驗收紀錄見 `reproduction/docs/clean_room_acceptance.md`。

## 分支與標籤

| 分支／標籤 | 內容 | 用途 |
|---|---|---|
| `main` | 預設分支，內容與 `thesis-full-reproduction` 相同 | 對外版本；日後的 release 標籤打在這裡 |
| `thesis-full-reproduction` | 2026-09-22 起的重現分支，從學姊原始交接版本分出 | 正式來源：reference run、2026-09-24 run 與正式 run `full_20261002T161428Z` 都由這個分支的 commit 執行；`reproduction/README.md` 的 clone 指令使用這個分支 |
| `reproduction-prep` | 最初的重現開發分支（2026-09-18～09-22），未合併 | 保存早期紀錄；其中的數值不是正式結果 |
| `archive/reproduction-prep-20260922` | 標籤，固定指向 `reproduction-prep` 的最後一個 commit | 封存 `reproduction-prep` 的最終狀態 |

三個分支的 01～04 資料夾完全相同，都是學姊的原始交接版本。修訂稿與 README
引用的數值只來自 `main`／`thesis-full-reproduction` 的
`reproduction/results/summary/full_20261002T161428Z/`。

`reproduction-prep` 的程式在 2026-09-22 整理後移入 `thesis-full-reproduction`。
該分支另保存下列 `thesis-full-reproduction` 沒有收錄的早期紀錄：

- 2026-09-18 重現開始時的環境、資料與 checkpoint 檢查：`results/raw/reproduction_start_20260918T070524Z/`
- 學姊保存 checkpoint 的評估：`results/raw/main_canonical_checkpoint/`、`results/summary/main_checkpoint_evidence/`、`results/summary/a14_checkpoint_comparison/`
- 標準化 decoder 的 CP 五 seed 候選測試：`results/raw/cp_standard_decoder_candidate_5seed/`
- 早期的主實驗五 seed 重跑摘要：`results/summary/fresh_5seed_main/`
- 第一版公平子集結果：`results/raw/fair_subset_notebook_v1/`
- 2026-09-20 以 notebook 進行重現的進度與稽核紀錄：`results/notebook_runs/tors_full_20260920/`
