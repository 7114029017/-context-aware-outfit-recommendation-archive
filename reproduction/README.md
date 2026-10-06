# TORS thesis reproduction artifact

本資料夾是《情境感知驅動的智慧穿搭推薦系統》之 2026 可重現性整理層。

目的不是改寫學姊原始研究資料，而是在保持原研究目錄不變的前提下，提供：

- 可執行的資料、環境與訓練檢查流程；
- 固定的 train / validation / test / CP / FITB / OR ID；
- 五個固定模型訓練 seeds；
- CP、FITB、OR 與 Judge 設定；
- 一鍵完整重現入口；
- seed-level raw results 與 summary；
- 已完成 reference run 的可核對證據；
- 對無法恢復的歷史來源限制做明確標示。

## 0. Clean-room full reproduction（接手者請先看）

這一節是**從全新 clone 開始的正式交接流程**。若目標是驗證「接手者只看本 README，
能否從乾淨環境完成一次新的 35-unit reproduction」，請不要複製舊的
`.venv`、`_external/`、`reproduction/runs/` 或
`_reproduction_runs/`，也不要使用已完成 reference run 的 reuse 模式。

目前 repository 的正式 reproduction branch 是：

`main`

正式 full-run launcher 開始訓練前會檢查兩件事：目前 branch 是 `main`，以及學姊原始
研究資料夾 01～03 的每個檔案都與 `reproduction/environment/archived_sources_manifest.json`
記錄的大小和 SHA-256 一致。任一項不符就拒絕開始昂貴訓練；可單獨執行
`python3 reproduction/scripts/verify_archived_sources.py` 檢查第二項。

### 0.1 Clone 正式 reproduction branch

    cd /path/to/your/workspace

    git clone \
      --branch main \
      https://github.com/7114029017/context-aware-outfit-recommendation.git

    cd context-aware-outfit-recommendation

    git branch --show-current
    git rev-parse HEAD
    git status --short

預期 branch 為 `main`。正式執行前，tracked working tree 不應有自行修改的學姊原始
研究程式。本 repository 的版本來源見根目錄 `README.md` 的「版本來源」。

### 0.2 取得 Git LFS feature artifacts

模型訓練除了 Polyvore annotation / metadata 外，還需要 repository 中保存的
8 份 precomputed feature artifacts。請先安裝 Git LFS，然後：

    git lfs install
    git lfs pull

後續 `check.sh` 會對這 8 份 feature 檔做 byte-size 與 SHA-256 驗證；
若仍是 LFS pointer、檔案缺失或 hash 不符，full run 不應開始。

### 0.3 建立新的 Python environment

已完成 reference run 的 recorded runtime 為：

- Python 3.12.3
- PyTorch 2.9.1+cu130
- torchvision 0.24.1
- CUDA build 13.0
- cuDNN 9.13.0
- GPU: NVIDIA GB10

建立新的 venv：

    python3 -m venv .venv-repro
    source .venv-repro/bin/activate
    python -m pip install --upgrade pip

若目標機器可使用 CUDA 13.0 wheel，可用下列版本組合盡量貼近 reference runtime：

    python -m pip install \
      torch==2.9.1 \
      torchvision==0.24.1 \
      --index-url https://download.pytorch.org/whl/cu130

若目標 GPU / driver 需要不同 CUDA-enabled PyTorch build，請安裝與該機器相容的
PyTorch / torchvision；不要用 CPU-only PyTorch 進行正式 full reproduction。
任何 runtime drift 都應保留在 environment evidence，不要為了對齊歷史版本而
偷偷替換或隱藏。

安裝其餘 reproduction dependencies：

    python -m pip install \
      -r reproduction/environment/requirements-reproduction-runtime.txt

快速確認 CUDA：

    python - <<'PY'
    import torch, torchvision
    print("torch:", torch.__version__)
    print("torchvision:", torchvision.__version__)
    print("cuda available:", torch.cuda.is_available())
    print("torch cuda:", torch.version.cuda)
    print("cudnn:", torch.backends.cudnn.version())
    print("gpu:", torch.cuda.get_device_name(0) if torch.cuda.is_available() else "NONE")
    PY

正式 full run 前，`cuda available` 必須為 `True`。

### 0.4 重新下載 Polyvore annotations / metadata

不要從舊工作目錄複製 dataset。由本 artifact 的 bootstrap 重新下載：

    bash reproduction/scripts/bootstrap_data.sh

目前 bootstrap source 為 `Stylique/Polyvore`，預設下載到：

`_external/uiuc-polyvore-hf/`

並記錄 local path 至：

`reproduction/.local/polyvore_root.txt`

Bootstrap 下載的是 reproduction 所需 annotation / metadata，
**不重新散布原始 Polyvore 圖片**。模型使用的 precomputed image/text features
來自前一節的 Git LFS artifacts。

### 0.5 Clean preflight

先執行：

    bash reproduction/scripts/check.sh

此步驟不做正式論文訓練，會檢查 Python/CUDA、8 份 feature artifacts、
dataset scope，以及 6 份 main ordered split manifests 的 byte identity。

成功的 check 輸出預設位於：

`reproduction/runs/check_<UTC timestamp>/`

若 check 失敗，請先保留輸出與錯誤訊息；不要修改學姊 `01`～`04` 原始研究
目錄來讓 verifier 通過。

### 0.6 建議先跑 smoke test

在開始約兩天的正式訓練前，建議執行：

    bash reproduction/scripts/smoke_test.sh

Smoke test 只驗證小型 subset 的 CP -> CIR 執行鏈，不可將其數值與論文正式結果比較。

成功的 smoke 輸出預設位於：

`reproduction/runs/smoke_<UTC timestamp>/`

NVIDIA GB10 這類 CPU／GPU 共用記憶體的機器：CUDA 只把真正空閒的記憶體算作可用，
不含系統檔案快取。若沒有其他 GPU 程式在跑，smoke test 卻出現
`CUDA error: out of memory`，請先釋放檔案快取（只清快取，不影響資料），
再重跑 smoke test；正式 full run 開始前也建議先清一次：

    sudo sh -c 'sync; echo 3 > /proc/sys/vm/drop_caches'

### 0.7 從頭執行完整 35-unit reproduction

Clean-room 驗收請明確使用 `--fresh`：

    bash reproduction/scripts/reproduce_all.sh --fresh

`--fresh` 會真正重新執行：

- main Original × seeds 1,2,3,4,5；
- main Context × seeds 1,2,3,4,5；
- candidate ablation 的 Original / Context / No-Weather / No-Occasion / No-Style
  × seeds 1,2,3,4,5；
- 共 10 main + 25 candidate-ablation = **35 training units**；
- 後續 secondary analyses、Chapter 4 report、statistics、
  `THESIS_TABLES_SUMMARY.md` 與 final reproduction summary。

完整輸出預設位於：

`reproduction/runs/full_<UTC timestamp>/`

正式 run 的 wall time 為 55:07:40（同一台機器另有其他工作；先前兩次完整 run 為
49:42:27 與 49:09:31）；實際時間會依 GPU / runtime 而變化。
長時間遠端執行建議使用 `tmux` / `screen` 或等價的持續 session，但這不是
scientific pipeline 的必要條件。

### 0.8 成功驗收條件

完整流程結束後，不要只看 terminal 最後一行。先找到該次 full run：

    RUN="$(find reproduction/runs -maxdepth 1 -type d -name 'full_*' | sort | tail -n 1)"
    echo "$RUN"
    cat "$RUN/RUN_STATUS.txt"

正式成功必須看到：

`PASSED`

並確認關鍵 final outputs：

    test -f "$RUN/reproduction_summary.json" && echo "summary OK"
    test -f "$RUN/chapter4/chapter4_report.json" && echo "chapter4 OK"
    test -f "$RUN/chapter4/THESIS_TABLES_SUMMARY.md" && echo "table report OK"
    test -f "$RUN/logs/final_terminal_summary.txt" && echo "terminal summary OK"

最後可重新顯示老師驗收摘要：

    python3 reproduction/scripts/show_reproduction_summary.py \
      --run-root "$RUN"

預期報告會包含 10 個 main units、25 個 candidate-ablation units、五-seed summary、
表 4-11 / 4-12、消融結果、統計檢定、Chapter 4 逐表 evidence 與 provenance limitations。

若 `RUN_STATUS.txt` 為 `FAILED`，請保留整個 run directory 與
`logs/full_console.log`。不要臨時改學姊原始程式或人工搬結果來把該 run
改成成功；先將失敗視為 reproduction artifact / environment 的可追查問題。

**與正式 run 比對。** terminal 摘要拿論文終稿的數字對照；驗收要確認的是本次 run
與正式 run `full_20261002T161428Z` 的結論是否一致（老師指引第二、十一節）。
在 0.3 建立的 environment 中執行：

    source .venv-repro/bin/activate
    python3 reproduction/scripts/compare_with_official_run.py --run-root "$RUN"

工具會並排列出主實驗與公平子集消融的五 seed 平均，並逐條判定指引第九節的七條
結論（判定方式與對帳表 R01～R07 相同）。最後一行是判定：

| 判定 | 意思 |
|---|---|
| `IDENTICAL` | 95 個逐 seed 結果檔與正式 run 逐位元相同。同一台 NVIDIA GB10、同一 runtime 應得到此結果。 |
| `CONSISTENT` | 數值不完全相同，但七條結論都一致。不同 GPU、CUDA 或 PyTorch build 時的預期結果。 |
| `NOT CONSISTENT` | 至少一條結論不同，或公平子集、OR 題目與正式 run 不同。請保留整個 run 目錄並回報。 |

加 `--checkpoints` 會再核對 70 個 checkpoint 的 SHA-256。報告同時存成
`$RUN/official_comparison.txt` 與 `$RUN/official_comparison.json`。

**完整結果在哪裡。** terminal 摘要只列平均值、R@1 與 R@3 的顯著性，以及老師的
10 項清單。完整統計在 run 目錄的檔案裡（路徑相對於 `$RUN`）：

| 內容 | 檔案 |
|---|---|
| 表 4-11、4-12（mean ± SD、Δ、95% CI、BH 校正後 p、Cohen's dz） | `statistics/table_4_11_cp_main.csv`、`statistics/table_4_12_or_main.csv` |
| 8 項主要指標的 paired t-test 與 BH 明細 | `statistics/main_paired_bh_8metrics.csv`、`statistics/statistical_evidence.md` |
| 表 4-13、4-14（公平子集 Original vs Context） | `ablation/summary/T03_stage1_cp_original_vs_full.csv`、`ablation/summary/T04_stage1_cir_original_vs_full.csv` |
| 表 4-15、4-16（消融） | `ablation/summary/T05_stage2_cp_ablation.csv`、`ablation/summary/T06_stage2_cir_ablation.csv` |
| 消融的 95% CI、p 值、Cohen's dz | `ablation/summary/T10_stage2_cp_seed_detail.csv`、`ablation/summary/T11_stage2_cir_seed_detail.csv` |
| 每個 seed 的結果 | `main/<variant>_seed<N>/evaluation/results_cp.csv`、`results_cir.csv`；`ablation/summary/fresh_25unit_metrics.csv` |
| 第四章 20 張表的來源與限制 | `chapter4/THESIS_TABLES_SUMMARY.md` |
| 所有輸出的索引 | `INDEX.md` |

正式 run 的同一批檔案收錄於 `results/summary/full_20261002T161428Z/`
（`ablation/summary/` 收錄後為 `ablation/`），數值整理在 `docs/expected_results.md`。

### 0.9 目前 clean-room 驗證狀態

**正式 run（修訂稿唯一的數值來源）**：照本節從 GitHub 全新 clone、建立新
environment 後執行的 clean-room run：

- run ID：`full_20261002T161428Z`
- commit：`b9bf5aac7d07cb39cacc7f73eb28166848c19fe4`
- 2026-10-02T16:14:28Z → 2026-10-04T23:22:08Z（55:07:40）
- `RUN_STATUS.txt = PASSED`
- 與下列兩次 run 比對：70 / 70 checkpoint SHA-256、70 / 70 seed-level result
  CSV、25 / 25 fair-subset detail CSV 完全相同；所有 summary 數值一致

驗收紀錄見 `reproduction/docs/clean_room_acceptance.md`；結果收錄於
`results/raw/full_20261002T161428Z/`、`results/summary/full_20261002T161428Z/`
與 `results/final_reference_manifest.json`。

先前兩次完整 run 作為佐證：2026 reference full run 已完成 35 units 並為
`PASSED`，該次執行完成於 repository restructuring 之前。

重構後已依本節以 `bash reproduction/scripts/reproduce_all.sh --fresh`
完成第二次全新的 35-unit end-to-end run：

- run ID：`full_20260924T102513Z`
- commit：`2cd8d04694ef314955be48ba47689e98a4b13d35`
- 2026-09-24T10:25:13Z → 2026-09-26T11:34:44Z（49:09:31）
- `RUN_STATUS.txt = PASSED`
- 與 reference run 比對：70 / 70 checkpoint SHA-256、95 / 95 seed-level
  result CSV SHA-256 完全相同；main / ablation / statistics / Chapter 4
  數值輸出一致（同一台 NVIDIA GB10、同一 runtime）

逐項證據見 `reproduction/docs/post_restructure_fresh_run_verification.md`。
不同 GPU / CUDA / PyTorch build 不預期 bit-identical；此時應以 §0.8 的
`PASSED` 與 retraining tolerance 判讀。

## 1. Provenance boundary

學姊原始研究內容位於 repository 根目錄的：

- `01_資料建構_data_construction/`
- `02_模型訓練和驗證_model_training_validation/`
- `03_實驗與結果_experiments_results/`
- `04_文件資料_documents/`

2026 reproduction artifact 不修改這四個資料夾。

Frozen senior baseline：

`7a5cc9cd8f884865e1e27a234c18dd132f949598`

此 commit 屬於原始開發 repository，不在本 repository 的歷史中。01～03 在該 commit
的內容記錄於 `reproduction/environment/archived_sources_manifest.json`，完整重現開始前
由 `reproduction/scripts/verify_archived_sources.py` 逐檔核對。

## 2. Official run and earlier reference runs

修訂稿的正式結果來自 clean-room run `full_20261002T161428Z`（commit `b9bf5aa`，
55:07:40，`PASSED`），細節見 §0.9 與 `docs/clean_room_acceptance.md`：

- raw results：`results/raw/full_20261002T161428Z/`
- summaries 與論文表格：`results/summary/full_20261002T161428Z/`
- seed index（每個 unit 的 checkpoint 雜湊、最佳 epoch、runtime）：
  `results/summary/full_20261002T161428Z/seed_index.csv`
- final manifest：`results/final_reference_manifest.json`
- 雜湊清單：`results/SHA256SUMS_full_20261002T161428Z.txt`

以下是先前的 reference run；它與正式 run 逐位元相同，`reproduce_all.sh`
預設的 reuse 模式仍以它做嚴格驗證。

已完成一次完整 fresh reproduction：

- reference ID: `reference_20260921T175217Z`
- training commit: `eb081e7a426cf5bd91acd8bb0f92a8398f08f45e`
- start: 2026-09-21T17:52:17Z
- finish: 2026-09-23T19:34:44Z
- wall time: 49:42:27
- main units: 10
- candidate-ablation units: 25
- total training units: 35
- status: `PASSED`

Repository restructuring 發生在完整 run 完成之後。

目前 `reproduction/results/` 中的 reference results 保存的是該次執行的結果。
重構後另外完成的 `--fresh` run（`full_20260924T102513Z`）與其 bit-identical，
見 `reproduction/docs/post_restructure_fresh_run_verification.md`。

若目的只是檢查已完成的重現結果，不需要重新訓練。

Reference raw results：

`reproduction/results/raw/reference_20260921T175217Z/`

Reference summaries：

`reproduction/results/summary/reference_20260921T175217Z/`

Seed-level provenance：

`reproduction/results/summary/reference_20260921T175217Z/reference_seed_index.csv`

## 3. Repository contents

| Path | Purpose |
|---|---|
| `configs/` | CP、FITB、OR、Judge 與 shared model 設定 |
| `data/` | 資料來源、取得方式與 provenance |
| `environment/` | runtime requirements、reference environment 與 workstation snapshot |
| `splits/` | 固定 main IDs 與 reconstructed candidate ablation IDs |
| `prompts/` | 可保存的 semantic prompt 與 Judge C/C* checklists |
| `scripts/` | check、smoke、full reproduction 與統計程式 |
| `results/raw/` | seed-level reference raw evidence |
| `results/summary/` | reference summary、Chapter 4 比較與 statistics |
| `results/tuning.csv` | 可恢復的 final-setting tuning provenance |
| `docs/` | provenance、expected results、論文對帳表、TORS compliance 與限制 |
| `runs/` | 新執行產物；Git ignored。預設以 `check_` / `smoke_` / `main_` / `full_` 區分執行模式 |

## 4. Data access

詳細資料 provenance：

`reproduction/data/README.md`

未來從乾淨環境下載所需 annotation / metadata：

    bash reproduction/scripts/bootstrap_data.sh

Bootstrap 使用設定的 Polyvore annotation/metadata mirror。

原始 Polyvore 圖片不由此腳本重新散布。

模型使用的 preserved feature artifacts 位於學姊原始模型資料夾，
並由 SHA-256 / byte-size verifier 在執行前檢查。

若只查看已保存的 reference results，不需要重新下載或重新訓練。

## 5. Environment setup

詳細說明：

`reproduction/environment/README.md`

建立 Python environment 的基本流程：

    python3 -m venv .venv-repro
    source .venv-repro/bin/activate
    python -m pip install --upgrade pip

先安裝符合實際 GPU / driver 的 CUDA-enabled PyTorch 與 torchvision。
已完成 reference runtime 使用 PyTorch 2.9.1+cu130 / torchvision 0.24.1；
若機器可使用 CUDA 13.0 wheel，可參考第 0 節的版本組合。再安裝 reproduction requirements：

    python -m pip install -r reproduction/environment/requirements-reproduction-runtime.txt

第一次交接／clean-room full reproduction 請優先依第 0 節的完整順序執行，
不要只從本節中途開始。

已完成 reference run 的環境證據保存在：

`reproduction/environment/reference_20260921T175217Z/`

目前已知的歷史限制是：2026 runtime 並不能證明與 2025 原始環境完全相同。

## 6. Fixed IDs

Main manifests：

- `splits/train_ids.csv`
- `splits/validation_ids.csv`
- `splits/test_ids.csv`
- `splits/cp_ids.csv`
- `splits/fitb_ids.csv`
- `splits/or_ids.csv`

Main scope：

| Scope | Count |
|---|---:|
| train | 16,995 |
| validation | 3,000 |
| test | 15,145 |
| CP test pairs | 30,290 |
| FITB questions | 15,145 |
| OR evaluable pairs | 9,311 |

Main manifests 有固定 SHA-256，clean preflight 會重新建立並做 byte-for-byte
identity check。

公平比較子集（表 4-13～4-16 使用）IDs 位於：

`splits/fair_subset/`

| Scope | Count | 檔案 |
|---|---:|---|
| total | 21,903 | `fair_subset_ids.txt` |
| train | 10,225 | `train_ids.txt` |
| validation | 1,748 | `valid_ids.txt` |
| test | 9,930 | `test_ids.txt` |
| OR（CIR）evaluable queries | 3,432 | `or_query_ids.csv` |

這是修訂稿正式使用的 reproducibly reconstructed fair subset：由天氣、場合、
風格三個因子皆具備的樣本組成，每次完整重現都會重新建立並驗證。建構規則、
排除條件、產生程式與 checksum 見 `docs/fair_subset.md`。

歷史原始 memberwise fair-subset ID file 未被恢復，因此不宣稱與歷史 subset
逐筆完全相同。

## 7. Experiment configuration

主要設定：

- `configs/cp.yaml`
- `configs/fitb.yaml`
- `configs/or.yaml`
- `configs/main_hybrid_attention.yaml`
- `configs/llm_judge.yaml`

Main fresh reproduction seeds：

`1, 2, 3, 4, 5`

人工稽核 seed `42` 與模型訓練 seeds 不同。

Evidence-backed main settings 包括：

- Hybrid Attention + FashionCLIP
- 100 epochs
- batch size 50
- Adam
- learning rate 5e-5
- weight decay 0
- gradient clipping 0.5
- dropout 0.1
- 16 attention heads
- 3 transformer layers
- float16 mixed precision
- hard negative from epoch 40
- validation FITB accuracy checkpoint selection

OR / CIR 使用 matching same-seed CP best checkpoint 初始化，
並使用 MarginRankingLoss，margin = 0.3。

## 8. Quick preflight

新的 clean environment 可先執行：

    bash reproduction/scripts/check.sh

此步驟會檢查：

- reproduction shell pipeline 是否可解析；
- Python packages 與 CUDA；
- 八份 preserved feature artifacts；
- dataset scope；
- 六份 main ordered split manifests 的 byte identity。

不會執行正式論文訓練。

未指定 `--run-root` 時，輸出預設位於：

`reproduction/runs/check_<UTC timestamp>/`

## 9. Optional smoke test

功能 smoke test：

    bash reproduction/scripts/smoke_test.sh

Smoke test 僅使用小型 subset 與 1 epoch 驗證 CP -> CIR pipeline 是否能運作。

Smoke test 的 numerical metrics 不可拿來與論文正式結果比較。

2026-09-24 已曾成功執行 relocated entrypoint smoke test，
輸出 `status: passed`。

未指定 `--run-root` 時，輸出預設位於：

`reproduction/runs/smoke_<UTC timestamp>/`

## 10. Full reproduction

完整重現入口：

    bash reproduction/scripts/reproduce_all.sh

這個入口現在具有「已完成 reference run 安全重用」機制。

若本機仍保留：

`_reproduction_runs/full_20260921T175217Z/`

則預設會先做嚴格驗證：

1. local `RUN_STATUS.txt = PASSED`；
2. committed reference seed index 必須恰好 35 rows；
3. 10 個 main + 25 個 candidate-ablation identity 必須完整；
4. 70 個 CP/CIR checkpoint SHA-256 必須逐一對上 seed index；
5. `reproduction/results/SHA256SUMS.txt` 的 201 個 committed reference results 必須全部 byte-identical。

全部通過後才會顯示：

`[SKIP VERIFIED FULL TRAINING]`

並直接使用已保存的 reference raw / summary / Chapter 4 / statistics。
這個模式**不會啟動 CP/CIR 訓練，也不宣稱第二次完成 35-unit training**。

驗證成功後，入口會自動執行 read-only：

    python3 reproduction/scripts/show_reproduction_summary.py --reference

並在 terminal 直接顯示：

- 表 4-11 CP：論文值 vs 五-seed 重現 mean ± SD；
- 表 4-12 OR/CIR：論文值 vs 五-seed 重現 mean ± SD；
- R@1、R@3 的 paired test 與 8-metric BH 校正結果；
- candidate fair-subset 消融五-seed摘要；
- Weather / Occasion OR 消融 fresh vs archived sign；
- Chapter 4 evidence coverage 與主要結果路徑。

這個顯示步驟只讀既有 CSV / JSON，不載入 checkpoint、不使用 CUDA、不重新訓練。

同一份 terminal summary 最後還會列出老師 2026-09-22 指定的 **10 項
reproducibility checklist**，逐項標示 `PASS` / `PASS*` / `PARTIAL` /
`READY`，並直接附上 raw、summary、environment、tuning、README、
branch / commit 等證據位置，這 10 項不需要再人工逐資料夾尋找。完整統計與表
4-13～4-16 不在 terminal 顯示，檔案位置見 §0.8。

如果本機沒有完整 reference run（例如新的 clean clone），則：

    bash reproduction/scripts/reproduce_all.sh

仍會自動進入全新的完整重現：

1. environment / feature / split identity gates；
2. Original 與 Context main experiments，5 seeds each；
3. candidate fair-subset five variants x 5 seeds；
4. preserved-output secondary analysis；
5. Chapter 4 comparison report；
6. five-seed summary 與 post-hoc statistics。

若即使 reference run 存在，仍要**明確重新訓練全部 35 units**：

    bash reproduction/scripts/reproduce_all.sh --fresh

也可以指定另一份已完成的 local full run 做 strict reuse verification：

    bash reproduction/scripts/reproduce_all.sh --reuse-reference /absolute/path/to/completed/full_run

Fresh run 預設新輸出位於：

`reproduction/runs/full_<UTC timestamp>/`

這仍是昂貴的訓練入口；`--fresh` 會真正重新執行訓練。

修訂稿的正式結果是 clean-room run `full_20261002T161428Z`（見 §0.9、§2）。它與
restructuring 前完成的 reference run（`eb081e7a...`）以及重構後的 `--fresh` run
`full_20260924T102513Z` 逐位元相同。

新的 `--fresh` full run 若完成為 `PASSED`，同樣會在最後從該次 run 的
`main/summary/`、`ablation/summary/`、`statistics/`、`chapter4/`
自動顯示同一份教授檢查摘要。terminal summary 失敗不會竄改已完成訓練的
`RUN_STATUS.txt`；原始 CSV / JSON 仍是權威結果。跑完後依 §0.8 與正式 run
比對。

另外，完整流程現在會自動產生一份 **表 4-1 ～ 表 4-20 的逐表來源報告**：

    chapter4/THESIS_TABLES_SUMMARY.md

此報告會對每一張表明確列出：

- 是否由 2026 fresh 重新訓練得到；
- 是否由學姊保存的 row-level / score / audit / case evidence 重新統計或查證；
- 保存資訊的來源性質；
- 本流程實際做了什麼、沒有做什麼；
- 對應的 machine-readable evidence 路徑；
- source-exact / historical provenance 的限制。

Reference reuse 模式也會在 `reproduction/runs/reference_20260921T175217Z/`
產生同名 `THESIS_TABLES_SUMMARY.md`，方便老師或接手者直接閱讀，而不必自己逐一尋找 20 張表的來源。

## 11. Expected results

正式 run 的預期結果（表 4-11～4-16 的數值，論文值列為歷史參照）：

`docs/expected_results.md`

正式 run 的主實驗結果與統計：

`results/summary/full_20261002T161428Z/main/`、`results/summary/full_20261002T161428Z/statistics/`

完整重現會自動產生論文表 4-11～4-16，欄位與論文相同（mean ± SD、Δ、95% CI、
BH 校正後 p、顯著性、Cohen's dz）。路徑相對於 run 目錄：

| 論文表 | 檔案 |
|---|---|
| 4-11 CP 主實驗 | `statistics/table_4_11_cp_main.csv` |
| 4-12 OR 主實驗 | `statistics/table_4_12_or_main.csv` |
| 4-13 公平子集 CP | `ablation/summary/T03_stage1_cp_original_vs_full.csv` |
| 4-14 公平子集 OR | `ablation/summary/T04_stage1_cir_original_vs_full.csv` |
| 4-15 CP 消融 | `ablation/summary/T05_stage2_cp_ablation.csv` |
| 4-16 OR 消融 | `ablation/summary/T06_stage2_cir_ablation.csv` |

主實驗的 BH 校正以 8 項主要指標為同一族群；公平子集以表列 5 項指標為同一族群，
沿用學姊原表（`03_…/T03`、`T04`）的規則。收錄到 `results/summary/<run>/` 後，
`ablation/summary/` 會變成 `ablation/`；正式 run 的六張表都在
`results/final_reference_manifest.json` 的 `thesis_tables`。

Main reproduction 的 16 個 paper-comparison means 均落在預先使用的
absolute tolerance 0.005 內。

八個 Context-minus-Original main aggregate metric directions 均為正。

五-seed paired analysis 經八個 main metrics 的 Benjamini-Hochberg correction
後，7 / 8 metrics 達 p < .05；Recall@1 未達顯著。

這些統計是描述已完成 reproduction evidence，不應解讀為所有歷史來源已被
精確恢復。

論文終稿與 TORS 投稿稿的逐項對帳表（每項列出稿件值、最新值、來源 CSV、
run ID 與程式位置，並標示需保留、更新數字或改寫的敘述）：

`docs/manuscript_reconciliation/reconciliation.md`

稿件數值轉錄於 `docs/manuscript_reconciliation/manuscript_values.csv`；
對帳表以正式 run 為來源，由下列指令產生：

    python3 reproduction/scripts/build_manuscript_reconciliation.py --official

要以另一個已完成的 run 產生對帳表時，用 `--run-root <run 目錄>` 並加上
`--out-dir <其他目錄>`，以免覆寫正式對帳表。

## 12. Hyperparameter tuning provenance

文件：

`docs/hyperparameter_tuning.md`

Machine-readable record：

`results/tuning.csv`

主模型（Hybrid Attention CP／CIR）的全部設定都是原論文（Wang & Zhong, IEEE Access 2024）
與原作者公開程式的設定：交接版本的 `config/base_config.py` 與原作者 repository 完全相同，
訓練程式中所有超參數相關程式行也相同；十份歷史 CP 訓練紀錄都是同一組設定。
validation 只用來選每次訓練的最佳 epoch，沒有證據顯示做過超參數搜尋。

每個設定依老師的四類（依文獻固定、validation 選擇、歷史紀錄遺失、補充分析）
逐項分類並附來源。第二模型（Two-Tower）的設定值有保存，但選擇理由未記錄；
也尚未做任何補充敏感度分析。因此 strict TORS A7 status 維持 `PARTIAL`。

本 artifact 不把 final settings 偽裝成歷史 tuning trials。

## 13. LLM-as-a-Judge

Judge provenance：

`docs/judge_provenance.md`

Archived machine-readable C / C* checklists：

`prompts/checklists/`

已保存：

- Qwen3-VL-32B-Instruct C / C*
- Gemma-3-27B-it C / C*
- weights / categories
- full preserved formal Judge outputs in the senior data tree
- robustness outputs used by preserved-output analysis

未恢復：

- exact historical Judge model revision；
- complete exact P0 / P1 / P2 executable prompt text；
- all historical generation parameters。

因此 preserved-output analysis 可重算，但 exact historical LLM generation
rerun provenance 為 partial。

## 14. Standardized decoder implementation

學姊 archived source 的 CP decoder 引用未保存的：

`DecoderLayerWithCrossAttn`

其 import 已被註解、定義不在交接資料中，因此 archived 程式無法直接執行，
早期自訂 decoder 的 exact forward behavior 也無法恢復。

修訂稿與本 artifact 的全部實驗（主實驗訓練、評估、公平子集消融與 smoke test）
統一採用 **standardized decoder implementation**：`torch.nn.TransformerDecoderLayer`
（3 層、16 heads、dropout 0.1，d_model 128）。這正是原論文作者公開程式
（WangXin93/text-conditioned-outfit-recommendation）的寫法；交接版本把它停用，
改用定義已遺失的自訂 class，因此標準化實作等同還原原作者的公開模型。

實作集中在 `scripts/standard_decoder.py`，只修改 working copy；學姊 repository
source 本身不修改。方法與輸入輸出說明見 `docs/standardized_decoder.md`。

本 artifact 不宣稱恢復歷史 `DecoderLayerWithCrossAttn` 的行為。

## 15. Result integrity

Reference result SHA-256 manifest：

`results/SHA256SUMS.txt`

Completed run identity：

`results/reference_20260921T175217Z_manifest.json`

Reference seed index：

`results/summary/reference_20260921T175217Z/reference_seed_index.csv`

大型 checkpoint 沒有重複 commit 到 reference result tree；checkpoint identity
由 preserved manifests 中的 SHA-256 記錄。

正式 run 完成後，用下列指令把它收進 `results/`（加上 `--verify-checkpoints`
會再重新計算 70 個 checkpoint 的 SHA-256）：

    python3 reproduction/scripts/package_official_run.py --run-root <run 目錄>

會產生 `results/raw/<run>/`、`results/summary/<run>/`（含 `seed_index.csv`，
每個 training unit 的 checkpoint 雜湊、最佳 epoch 與 runtime）、
`environment/<run>/`、`results/final_reference_manifest.json`（run ID、commit、
seeds、環境、切分 checksum、70 個 checkpoint 雜湊、結果位置與程式雜湊）以及
`results/SHA256SUMS_<run>.txt`。

## 16. TORS compliance status

目前狀態：

`docs/tors_compliance.md`

老師 2026-09-30 後續工作清單的逐項完成狀態與證據：

`docs/tors_followup_status.md`

Repository artifact 已完成核心 reproduction evidence 整理，但下列項目仍需在
正式對外交付階段完成或確認：

- repository public accessibility；
- fixed GitHub Release；
- Zenodo DOI，如投稿要求；
- final Artifact Availability statement；
- third-party dataset / artifact redistribution terms；
- strict A7 historical tuning provenance remains partial。

## 17. Known limitations

本 artifact 不宣稱以下項目已 source-exact recovery：

- historical fair-subset memberwise identity；
- historical DecoderLayerWithCrossAttn implementation；
- historical hyperparameter-search trial records；
- exact historical 2025 runtime；
- complete historical Judge generation prompt/parameter provenance；
- every qualitative/image-dependent thesis output。

因此整體研究來源層級 provenance 應描述為 `partial`。

逐項清單（缺什麼、本 artifact 怎麼處理、證據位置）見 `docs/known_limitations.md`。

另一方面，2026 main fresh numerical reproduction、seed-level evidence、
candidate-ablation reproduction artifact 與 preserved-output analysis 都有明確
保存與來源界線。

## 18. Handoff rule

不要修改學姊 `01` 至 `04` 原始研究目錄來讓數字符合論文。

新的實驗輸出一律放在：

`reproduction/runs/`

新的可重現性 artifact 一律放在：

`reproduction/`

不要使用 `git add .`；應精確 stage 要提交的 reproduction artifact。
