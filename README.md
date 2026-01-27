````markdown
# IoT Timeseries Validation Tools（IoT 数据校验工具集）

概览：针对IoT平台功能短板，服务各基地数据运维人员，使用一份 Excel 作为输入（采集点编码 + 时间戳），按基地配置批量拉取 IoT 时序数据，输出可对账的明细表（Excel）与异常曲线图（PNG），用于快速核验、定位与留痕。

## 说明

这是一组面向“某制造企业 / 某 IoT 平台 / 某 EMS”日常数据核验场景的轻量工具，核心诉求是把**人工在平台/接口里反复查询、复制粘贴、截图留证**的过程，工程化成：

- **可复用的输入规范**：Excel 模板（采集点编码 / 时间戳）
- **可切换的环境**：通过配置文件注入各基地接口地址（不提交凭据/内网信息）
- **可审计的输出**：带运行时刻戳命名的 Excel 结果文件 + 可归档的曲线图
---

## 适用场景 / Pain points

典型场景：

- **抄表值/现场记录 vs 平台返回值**需要按某个“目标时间戳”核对，但平台查询通常只能按区间翻页，人工找“最接近的一条”费时且易错。
- **异常点排查**：需要把异常发生前后若干分钟/小时的曲线拉出来，看是否存在阶跃、毛刺、长时间恒定、断点等问题，人工截图难留痕、难批量。
- 多基地环境下，接口地址/路由不一致，临时切换环境成本高。

---

## 产物（输出物）与用途

### 1) 时间戳核验输出（Nearest Value）
- **输出文件**：`多个时间的iot数据_YYYYMMDDhhmmss.xlsx`
- **字段**：`采集点编码 / 返回值 / 时间戳(实际返回点)`  
- **用途**：按“目标时间戳”找到**指定前后偏移窗口内**最接近的一条返回值，用于对账、核验、争议澄清。

### 2) 区间全量拉取输出（Full Timeseries）
- **输出目录**：`anomaly_analysis/outputs/`
- **输出文件**：`outputs/多个时间的iot数据_YYYYMMDDhhmmss.xlsx`
- **字段**：`采集点编码 / 返回值 / 时间戳 / 请求开始时间 / 请求结束时间`
- **用途**：把“异常点前后窗口”内所有点拉齐，作为后续分析、绘图、留档的原始数据集。

### 3) 异常曲线图（PNG）
- **输出目录**：`anomaly_analysis/outputs/`
- **输出文件**：`{采集点编码}_{异常点时间戳}.png`
- **用途**：把每个采集点、每个请求窗口画成一张曲线图，便于快速定位异常形态与沟通（可直接贴工单/周报/复盘）。

---

## 工作流 / Pipeline（模块分层）

输入到输出的完整链条：

1. **输入**：Excel（必须包含两列：`采集点编码`、`时间戳`）
2. **分组与批处理**：按 `时间戳` 分组，将同一时间戳下的多个 `tagCodes` 合并成一次请求（降低请求次数）
3. **时间窗口生成**：对每个时间戳生成 `startTime/endTime = timestamp ± offset_minutes`
4. **接口拉取**：POST 请求
   - Request body：`{"tagCodes": [...], "startTime": "...", "endTime": "..."}`
   - Response 预期：`code == 0`，并包含 `data[].tagCode` + `data[].timeSeries[]`
5. **两种加工策略**：
   - **Nearest Value**：在 `timeSeries` 中找离目标时间戳最近的数据点
   - **Full Timeseries**：保留窗口内所有点，并记录请求窗口（开始/结束）
6. **输出**：按运行时刻戳生成 Excel；（可选）读取 outputs 下所有结果文件，按点位/窗口分组绘制并保存 PNG

---

## 目录导览（核心脚本/模块）

```text
.
├─ iot_validation_by_timestamp/
│  ├─ fetch_iot_timeseries_gui.py      # GUI：按时间戳取“最近值”（对账核验）
│  ├─ 各基地url.txt  # 形如：基地名=http://.../timeseries
│  └─ 待拉取的编码和时间戳.xlsx（输入模板）  # 两列：采集点编码 / 时间戳
│
├─ anomaly_analysis/
│  ├─ fetch_iot_timeseries_multiwindow_gui.py  # GUI：按窗口拉“全量时序”（异常分析）
│  ├─ plot_anomaly_curves.py                   # 批量绘图：输出 PNG 到 outputs/
│  ├─ 各基地url.txt
│  └─ 待拉取的编码和时间戳.xlsx（输入模板）
│
└─ requirements.txt                      # 依赖锁定（pandas/requests/matplotlib/openpyxl/tkinter）
````

---

## 快速使用（配置注入 / 不提交凭据）

### 0) 环境准备

```bash
python -m venv .venv
# Windows
.\.venv\Scripts\activate
# macOS/Linux
source .venv/bin/activate

pip install -r requirements.txt
```

### 1) 配置各基地 URL（不提交内网信息）

在运行目录放置 `各基地url.txt`（已脱敏，本地真实地址不提交）：

```ini
某基地A=http://基地IP:端口号/japrojecttag/timeseries
某基地B=http://基地IP:端口号/japrojecttag/timeseries
```

### 2) 准备输入 Excel（两列必填）

文件名：`待拉取的编码和时间戳.xlsx`
列名必须为：

| 采集点编码    | 时间戳                 |
| -------- | ------------------- |
| TAG_0001 | 2025-07-06 19:17:00 |
| TAG_0002 | 20250706191700      |

时间戳支持多格式（代码内做了兼容解析），例如：

* `yyyy/m/d h:mm:ss`
* `yyyy-m-d h:mm:ss`
* `yyyymmddhhmmss`
* 带毫秒：`yyyy-m-d h:mm:ss.fff`
* 以及日期级别输入等

---

## 运行方式

### A) 按时间戳核验（Nearest Value）

```bash
cd iot_validation_by_timestamp
python fetch_iot_timeseries_gui.py
```

在 GUI 中选择：

* 基地
* 时间偏移量（10–120 分钟）

输出：

* `多个时间的iot数据_*.xlsx`（当前目录）

### B) 按窗口拉全量时序（Full Timeseries）

```bash
cd anomaly_analysis
# 确保 outputs/ 存在（脚本当前不会自动创建）
mkdir outputs
python fetch_iot_timeseries_multiwindow_gui.py
```

在 GUI 中选择：

* 基地
* 时间偏移量（10–360 分钟）

输出：

* `outputs/多个时间的iot数据_*.xlsx`

### C) 绘制异常曲线图（PNG）

```bash
cd anomaly_analysis
python plot_anomaly_curves.py
```

输出：

* `outputs/*.png`

---

## 输出文件说明

### Excel：最近值（对账核验）

* `采集点编码`：tagCode
* `返回值`：tagValue（最近一条）
* `时间戳`：最近一条数据的实际时间（非输入时间戳）

### Excel：窗口全量（异常分析）

* `采集点编码` / `返回值` / `时间戳`
* `请求开始时间` / `请求结束时间`：该批请求覆盖的窗口，用于复现实验条件与分组绘图

### PNG：异常曲线

* 横轴：时间戳
* 纵轴：返回值
* 标注：采集点编码 + 异常点时间戳（窗口中点，用于定位该窗口对应的“目标点”）

---

## 配置与脱敏说明（合规边界）

* 公司/行业/系统名：README 及示例统一使用“某制造企业 / 某 IoT 平台 / 某 EMS”等匿名表述
* 接口地址：仓库仅保留**占位符示例**；真实地址放本地配置文件注入
* 数据示例：仅展示字段名与伪造/脱敏的 tagCode；如需截图，需去标识化处理（点位命名、基地名、时间段等）
* 输出文件：已在 `.gitignore` 中默认忽略 `outputs/`、`*.xlsx`（避免误提交运行产物）
