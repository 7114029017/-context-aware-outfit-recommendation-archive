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
`7a5cc9cd8f884865e1e27a234c18dd132f949598` 不變。完整重現開始前，
`reproduction/scripts/pipeline/launch_full.sh` 會依
`reproduction/environment/archived_sources_manifest.json` 逐檔核對 01～03 的內容。

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

## 版本來源

本 repository 是重現成果的公開版本，只有 `main` 一個分支。重現期間的開發在另一個
原始開發 repository 進行，正式 run 的紀錄和文件中出現的 commit 編號（例如正式 run 的
`b9bf5aa`、frozen baseline `7a5cc9c`）與 repository 網址，指的都是那個 repository。

本 repository 的第一個 commit（`458b723`）與原始開發 repository 的 commit `6693528`
內容完全相同（git tree `00a91b400f1df790464d21fae41f72c1b27da4a1`）。正式 run 之後，
原始開發 repository 只更新了文件、整理後的結果與報表工具；設定、切分、訓練與統計
程式都沒有改動。之後在本 repository 的修改見 `git log`。

修訂稿與 README 引用的數值只來自 `reproduction/results/summary/full_20261002T161428Z/`。
