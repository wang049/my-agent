# 规格书抄表

第一个功能：**工位电脑上丢进一份 PDF / Word / Excel，大约 5 分钟出一张可核对的表。**

规格书只在本机识别，**不出公网**。不做开放式聊天 Agent。流程是：

1. 本机 `document_reader` 读文件（带页码和表格）
2. 本机 Ollama（`127.0.0.1`）一次结构化抽取
3. 写成 Excel：值 + 页码 + 原文摘录 + 置信度；黄底 = 请人工看

扫描件 8 分钟是下一阶段（本机 OCR），这一版先打数字稿。5 分钟是否达标，要用**目标工位电脑 + 本机模型**重测，不能拿以前的 Krow Cloud 耗时当验收。

## 工位电脑准备

1. 安装 [Ollama](https://ollama.com)（Windows / macOS 均可）
2. 拉取本机模型（只需一次；离线工位用 U 盘拷模型，不要把规格书发上网）：

```bash
ollama pull qwen2.5:7b
```

3. 保持 Ollama 在后台运行（Windows 装好一般会自动起来）

不需要 `KROW_API_KEY`。程序默认只打 `http://127.0.0.1:11434`；若指向 `api.krow.cn` 或其他非本机地址，会直接失败。

可选环境变量：

```bash
export SPEC_SHEET_LLM_URL=http://127.0.0.1:11434
export SPEC_SHEET_LLM_MODEL=qwen2.5:7b
```

内存紧张用 `qwen2.5:7b`；工位有 16GB 以上内存可试 `qwen2.5:14b`（质量更好，更慢）。

## 本地「点一下出表」演示

模拟以后在 Workbench 里点一次出表。只监听本机 `127.0.0.1`，仍走同一条流水线，文件不出公网。Ollama 先开着。

```bash
cd ~/spec-sheet
python -m spec_sheet.demo
```

浏览器会打开 http://127.0.0.1:8765/ 。选 PDF / Word / Excel，点「出表」，完成后下载核对表。这不是 Krow Workbench 本身。

## 环境

Python 3.12。在项目目录：

```bash
cd ~/spec-sheet
export PATH="$HOME/.venv/bin:$HOME/.local/bin:$PATH"
```

## 先用示例文件测

项目里有一份虚构的「交流伺服电机」规格书，三种格式都有：

```bash
cd ~/spec-sheet
python -m spec_sheet examples/sample_servo_spec.pdf
python -m spec_sheet examples/sample_servo_spec.docx
python -m spec_sheet examples/sample_servo_spec.xlsx
```

抽完打开 `output/sample_servo_spec_核对表.xlsx`。可对答案：型号 `SV-200A-400W`，额定功率 `400 W`，防护等级 `IP65`。

对照黄金答案打分（不调模型）：

```bash
python -m spec_sheet.eval output/sample_servo_spec_核对表.xlsx examples/golden/sample_servo_spec.json
```

困难示例（历史型号 / 峰值电流 / 包装重量干扰）等客户文件前用来找抽错：

```bash
python -m spec_sheet examples/sample_servo_spec_hard.pdf
python -m spec_sheet.eval output/sample_servo_spec_hard_核对表.xlsx examples/golden/sample_servo_spec_hard.json
```

每本规格书只是中文、英文或德文其中一种（不是同一页三种语言）。核对表表头仍用中文，英文 `Type` / 德文 `Typ` 映射到「型号」。分别测：

```bash
python -m spec_sheet examples/sample_servo_spec.pdf
python -m spec_sheet examples/sample_servo_spec_en.pdf
python -m spec_sheet examples/sample_servo_spec_de.pdf
```

德文示例用小数逗号（`2,8 A`、`1,35 kg`），打分时与 `2.8` / `1.35` 视为同一数值：

```bash
python -m spec_sheet.eval output/sample_servo_spec_en_核对表.xlsx examples/golden/sample_servo_spec_en.json
python -m spec_sheet.eval output/sample_servo_spec_de_核对表.xlsx examples/golden/sample_servo_spec_de.json
```

## 跑自己的规格书

```bash
cd ~/spec-sheet
python -m spec_sheet /路径/你的规格书.pdf
```

Word / Excel 同样：

```bash
python -m spec_sheet /路径/规格书.docx
python -m spec_sheet /路径/参数表.xlsx
```

指定输出位置：

```bash
python -m spec_sheet /路径/规格书.pdf -o ~/Desktop/核对表.xlsx
```

成功时终端类似：

```
源文件 : 某某规格书.pdf
扫描件 : 否（按数字稿处理）
输出   : /Users/你/spec-sheet/output/某某规格书_核对表.xlsx
字段   : 20 行，其中 8 行建议人工核对
耗时   : 47.2 秒
```

打开 xlsx：

- 工作表「核对表」：逐字段核对
- 黄底行：空值，或置信度 &lt; 0.8
- 「核对状态」下拉：待核对 / 通过 / 需修改

## 现在不要做的事

- 不要把规格书发到 `api.krow.cn` 或任何公网模型
- 不要 `git clone aullik5/krow`（私有仓，外部开发者不需要）
- 不要对整本规格书跑 `agent.run("帮我抄表")`（会超时、漏字段）

## 客户文件还没到时

先把困难示例和黄金答案在**本机模型、断公网**下跑通；同时向客户要 5 份脱敏样张（最好抄的、最厚的、表格最乱的、一本扫描件、一份他们现在的人工抄表）。不要先做开放式 Agent 或精细 OCR。

## 下一步

本机数字稿跑通、人工核对可接受、工位电脑耗时可接受之后，再加：Windows 拖放壳、本机扫描件 OCR、你们自己的字段模板。
