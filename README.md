# ResNet-18 Bare-Metal RV64IMC 重現指南

本專案以 LiteRT Micro（TFLM）執行量化 ResNet-18，提供 bare-metal C/C++ 程式、custom Spike 原始碼、profiling 腳本。

重現版本包含 `B0`、`B1_block8_ptr`、`X1`（QMAC8）與 `X2`（QMAC8 加上 requantization custom instruction）。模型與輸入資料均已提供，不需要重新訓練、量化或安裝 TensorFlow。

## 1. 環境需求

- Linux x86_64，建議 Ubuntu 24.04；Windows 可使用 WSL2 Ubuntu。
- Python 3.10 以上，以及支援 C++20 的主機編譯器。
- 初次安裝需要網路，並需預留數 GB 磁碟空間。
- 安裝主機套件需要 root／sudo；後續建置與量測不需要 root。
- 專案路徑不可含空白。

以下指令都在**專案根目錄**執行，也就是包含 `scripts/`、`workloads/`、`spike/` 的目錄。

使用壓縮檔時，先解壓並進入目錄：

```bash
tar -xzf rv64-resnet18-reproducible.tar.gz
cd rv64-resnet18-reproducible
```

若從 GitHub clone，則進入 clone 後的專案根目錄。

## 2. 安裝依賴與準備工具

```bash
bash scripts/install_host_deps.sh
python3 scripts/bootstrap.py --jobs 4
```

第一行安裝主機依賴，需要 root／sudo。若主機已安裝所需依賴，可跳過第一行。

第二行會下載固定版本的 RISC-V 工具鏈、編譯專案附上的 custom Spike，並在 `tools/` 準備執行環境。不需要自行預先安裝 RISC-V 工具鏈或 Spike。

工具鏈解壓與首次 Spike 編譯需要時間，在共享磁碟上可能較慢。

## 3. 一次完成全部重現

```bash
python3 scripts/reproduce.py all --jobs 4
```

此指令會重新建置四個版本，產生完整 profiling 與逐節點量測結果，不需要再分別執行下方的 `profile` 與 `nodes`。

| 量測 | 工作內容 |
|---|---|
| `profile` | 四個版本的完整 histogram、instruction mix、Invoke 指令數與 code size |
| `nodes` | 四個版本各節點的指令數與 operator 貢獻 |

完整 histogram 模擬可能耗時數十分鐘或更久。`--jobs 4` 控制平行編譯工作數，不會讓單次 Spike 模擬使用四個核心。

## 4. 報告主要看哪些檔案

**整體量測主要看 `profile/metrics.txt`；每個節點與 operator 貢獻主要看 `nodes/summary.md`。**

例如 X2 完成量測後，直接執行：

```bash
# 整體指標
cat results/X2/profile/metrics.txt

# 各節點與 operator 貢獻表格
cat results/X2/nodes/summary.md
```

將 `X2` 換成 `B0`、`B1_block8_ptr` 或 `X1`，即可查看其他版本。

### 輸出目錄

| 目錄 | 說明 |
|---|---|
| `results/<版本>/profile/` | 完整 profiling report |
| `results/<版本>/nodes/` | 逐節點與 operator 貢獻報告 |
| `workloads/<版本>/report/` | Profiling 執行時產生的工作報告；完成後複製至上方的 `profile/` |

例如工作報告的整體指標也可從下列路徑查看：

```bash
cat workloads/X2/report/metrics.txt
```

建議以 `results/` 下的報告作為檢視入口。

### 整體報告：`results/<版本>/profile/`

| 檔案 | 用途 |
|---|---|
| **`metrics.txt`** | **主要閱讀檔案：整理整體量測指標** |
| `metrics.md` | Markdown 格式的整體指標報告 |
| `metrics.json` | JSON 格式的整體量測指標 |
| `summary.txt` | Histogram 與指令計數摘要 |
| `instruction-counts.csv` | 各指令 mnemonic 的動態執行次數 |
| `instruction-mix.csv` | 各指令類別的數量與比例 |
| `measurements.json` | Invoke 計數器起訖值與 ELF 身分資訊 |
| `pc-histogram.txt` | Spike 原始輸出，包含各 PC 執行次數與量測讀值 |
| `disassembly.txt` | 反組譯內容，用於將 PC 對應至指令 |
| `program.elf` | 與此份報告對應的實際執行檔 |
| `run.log` | 程式執行輸出 |
| `cache.txt` | 啟用 cache 模擬時的統計；未啟用時可能為空 |

完整報告中的 `operator-contribution.csv` 可能只有通用 metrics 腳本能取得的資訊。**實測的 operator 貢獻請以以下 `nodes/summary.md` 與 `operators.csv` 為準。**

### 節點報告：`results/<版本>/nodes/`

| 檔案 | 用途 |
|---|---|
| **`summary.md`** | **主要閱讀檔案：各節點與 operator 貢獻的表格摘要** |
| `nodes.csv` | 每個節點的 retired instructions 與 Invoke 占比 |
| `operators.csv` | 依 operator 類型彙總的指令數與占比，例如所有 convolution 的合計 |
| `metadata.json` | 本次節點量測的資訊 |
| `run.log` | 加入節點量測後的推論執行輸出 |

### 量測範圍

- **Invoke retired instructions**：推論區間內的指令數。
- **Whole-program histogram**：包含推論以外的執行，不可直接當成 Invoke 指令數。
- **Node profiling**：讀取計數器會增加少量指令，因此其 Invoke 總數與正式 profile 略有差異。
- **Operator 貢獻**：以 retired instructions 占比表示，不是 cycle 或執行時間占比。

## 5. 單獨執行各階段（可選）

若已執行 `all` 並成功，不需要再執行本節。以下指令適合分開量測或只執行單一版本。

### 四個版本的節點與 operator 貢獻

```bash
python3 scripts/reproduce.py nodes --versions B0 B1_block8_ptr X1 X2 --jobs 4
```

### 四個版本的完整 profiling

```bash
python3 scripts/reproduce.py profile --versions B0 B1_block8_ptr X1 X2 --jobs 4
```

### 只執行 X2

```bash
python3 scripts/reproduce.py nodes --versions X2 --jobs 4
python3 scripts/reproduce.py profile --versions X2 --jobs 4
```
