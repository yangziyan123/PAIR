# Windows 本地多模型实验指导

## 1. 适用范围

这份文档面向你的当前目标：

- 在 **Windows 原生环境** 下，先把这 3 个 8B 级别模型跑通：
  - `Llama 3 8B`
  - `Qwen3 8B`
  - `Ministral 3 8B`
- 完成最小可用验证：
  - 模型能下载
  - 模型能在本机正常回复
  - 你能通过命令行、PowerShell API、Python OpenAI 兼容接口访问它们
- 为后续接入当前 `PAIR` 仓库做准备

本文 **先不改 `PAIR` 代码**。目标是把 Windows 侧模型运行链路先稳定打通。

## 2. 为什么 Windows 下先选 Ollama

在你当前这个阶段，Windows 原生最稳、最省事的路线是：

- **Windows**
- **Ollama**
- **本地 API**
- **逐个模型做冒烟测试**

原因：

- Ollama 官方文档明确支持 **Windows 原生**，并说明安装后会在后台运行，本地 API 默认监听 `http://localhost:11434`。
- 这三个模型目前都能通过 Ollama 的模型库直接拉取。
- 这样可以避免你一开始就陷入 `vLLM + WSL2 + CUDA/驱动 + Linux` 的组合复杂度。

结论：

- **第一阶段**：先在 Windows 原生用 Ollama 跑通三种模型
- **第二阶段**：再考虑把它们接入当前 `PAIR` 仓库

## 3. 截至 2026-03-19 我查到的三个模型入口

### 3.1 模型对应关系

| 目标模型 | Windows 下推荐本地标签 | 官方/模型页信息 |
|---|---|---|
| Llama 3 8B | `llama3:8b` | Ollama 页面显示约 `4.7GB`，上下文 `8K` |
| Qwen3 8B | `qwen3:8b` | Ollama 页面显示约 `5.2GB`，上下文 `40K` |
| Ministral 3 8B | `ministral-3:8b` | Ollama 页面显示约 `6.0GB`，`Q4_K_M` 量化，页面注明需要 `Ollama 0.13.1` |

### 3.2 你要知道的现实差异

- `Llama 3 8B`
  在 Ollama 里最容易直接跑通，适合先做第一个样例。

- `Qwen3 8B`
  在官方 Hugging Face 模型页里明确说明：
  - 需要较新的 `transformers`
  - `transformers < 4.51.0` 可能会报 `KeyError: 'qwen3'`
  - 本地使用时也支持 Ollama

- `Ministral 3 8B`
  当前官方 Ollama 页面写得很清楚：
  - 标签是 `ministral-3:8b`
  - 当前页面写明它需要 `Ollama 0.13.1`
  - 所以如果你 Ollama 太旧，前两个模型能跑，第三个可能会失败

## 4. 先检查你的 Windows 条件

根据 Ollama 官方 Windows 文档，最低要确认这些条件：

- Windows 10 22H2 或更新版本
- 如果你是 NVIDIA 显卡，驱动版本至少要满足官方要求
- 如果你是 AMD 显卡，也要先装好对应驱动

你可以先在 PowerShell 里做两个最基本检查：

```powershell
winver
```

如果你是 NVIDIA：

```powershell
nvidia-smi
```

成功标准：

- `winver` 能确认系统版本不是太老
- `nvidia-smi` 能正常显示 GPU 和驱动信息

如果 `nvidia-smi` 都不通，先不要急着装模型。

## 5. 第一步：安装 Ollama for Windows

### 5.1 安装方式

去官方 Windows 下载页安装：

- https://ollama.com/download/windows

或者看官方 Windows 文档：

- https://docs.ollama.com/windows

官方文档说明：

- Ollama 可以作为 **Windows 原生应用**运行
- 安装后 `ollama` 命令可直接在 `cmd` / `powershell` 中使用
- API 默认在 `http://localhost:11434`

### 5.2 安装完成后验证

开一个新的 PowerShell：

```powershell
ollama --version
```

然后再执行：

```powershell
ollama list
```

如果能看到版本号和一个空模型列表，说明安装基本正常。

## 6. 第二步：把模型存储目录改到你能控制的位置

官方 Windows 文档说明可以通过环境变量 `OLLAMA_MODELS` 修改模型存放目录。

强烈建议你现在就改，不然后面 3 个模型会默认往用户目录下写。

### 6.1 推荐目录

例如你可以用：

```text
D:\OllamaModels
```

### 6.2 Windows 设置方法

1. 打开“编辑账户的环境变量”
2. 新建用户变量：
   - 变量名：`OLLAMA_MODELS`
   - 变量值：`D:\OllamaModels`
3. 保存后，退出 Ollama 托盘程序
4. 重新启动 Ollama
5. 重新开一个 PowerShell

### 6.3 重新验证

```powershell
echo $env:OLLAMA_MODELS
```

应该能看到你刚设置的目录。

## 7. 第三步：先拉最容易成功的模型

建议你按这个顺序拉：

1. `llama3:8b`
2. `qwen3:8b`
3. `ministral-3:8b`

原因：

- 先用最稳的 `llama3:8b` 验证流程
- 再试 `qwen3:8b`
- 最后处理对 Ollama 版本要求更敏感的 `ministral-3:8b`

### 7.1 拉取 Llama 3 8B

```powershell
ollama pull llama3:8b
```

### 7.2 拉取 Qwen3 8B

```powershell
ollama pull qwen3:8b
```

### 7.3 拉取 Ministral 3 8B

```powershell
ollama pull ministral-3:8b
```

### 7.4 拉完后查看本地模型

```powershell
ollama list
```

你应该能看到至少这三个标签中的已成功项：

- `llama3:8b`
- `qwen3:8b`
- `ministral-3:8b`

## 8. 如果 `ministral-3:8b` 拉取失败，先检查 Ollama 版本

这是最容易踩的坑。

官方 Ollama 的 `ministral-3:8b` 页面目前写明：

- 该模型需要 `Ollama 0.13.1`

所以如果你前两个模型都正常，但 `ministral-3:8b` 失败，优先做这三步：

1. 看当前版本：

```powershell
ollama --version
```

2. 去官方 Windows 下载页更新 Ollama：

- https://ollama.com/download/windows

3. 升级后重新执行：

```powershell
ollama pull ministral-3:8b
```

## 9. 第四步：逐个做交互式冒烟测试

这里的目标非常简单：

- 确认模型真的能响应
- 确认不是“下载完成但运行失败”

### 9.1 测试 Llama 3 8B

```powershell
ollama run llama3:8b
```

输入一条无害请求，例如：

```text
请用一句中文介绍你自己。
```

### 9.2 测试 Qwen3 8B

```powershell
ollama run qwen3:8b
```

输入：

```text
请用一句中文介绍你自己。
```

### 9.3 测试 Ministral 3 8B

```powershell
ollama run ministral-3:8b
```

输入：

```text
请用一句中文介绍你自己。
```

### 9.4 成功标准

每个模型都至少满足：

- 命令可以进入交互
- 能返回完整文本
- 没有立即退出或报显存错误

## 10. 第五步：用 PowerShell 直接打 API

这一步比交互命令更重要，因为后续接仓库时你真正要用的是 **本地 API**。

Ollama 官方 Windows 文档说明：

- 安装后 API 默认在 `http://localhost:11434`

### 10.1 测试 Llama 3 8B

```powershell
(Invoke-WebRequest -Method POST `
  -Uri http://localhost:11434/api/chat `
  -ContentType "application/json" `
  -Body '{"model":"llama3:8b","messages":[{"role":"user","content":"请用一句中文介绍你自己。"}],"stream":false}').Content | ConvertFrom-Json
```

### 10.2 测试 Qwen3 8B

```powershell
(Invoke-WebRequest -Method POST `
  -Uri http://localhost:11434/api/chat `
  -ContentType "application/json" `
  -Body '{"model":"qwen3:8b","messages":[{"role":"user","content":"请用一句中文介绍你自己。"}],"stream":false}').Content | ConvertFrom-Json
```

### 10.3 测试 Ministral 3 8B

```powershell
(Invoke-WebRequest -Method POST `
  -Uri http://localhost:11434/api/chat `
  -ContentType "application/json" `
  -Body '{"model":"ministral-3:8b","messages":[{"role":"user","content":"请用一句中文介绍你自己。"}],"stream":false}').Content | ConvertFrom-Json
```

### 10.4 你真正要看什么

返回 JSON 里重点看：

- 是否有 `message`
- `message.content` 是否为正常文本
- 是否报内存不足、模型不存在、版本不兼容

## 11. 第六步：用 Python 的 OpenAI 兼容方式访问

Ollama 官方文档明确提供了 OpenAI 兼容接口，示例写法是：

- `base_url='http://localhost:11434/v1/'`
- `api_key='ollama'`，这个值必填但会被忽略

这一步非常重要，因为你后续如果接实验代码，大概率会走这种兼容接口思路。

### 11.1 创建最小 Python 环境

在仓库目录里开 PowerShell：

```powershell
cd D:\大创\PAIR
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip setuptools wheel
pip install openai requests
```

### 11.2 写一个最小测试脚本

```powershell
@'
from openai import OpenAI

MODELS = [
    "llama3:8b",
    "qwen3:8b",
    "ministral-3:8b",
]

client = OpenAI(
    base_url="http://localhost:11434/v1/",
    api_key="ollama",
)

for model in MODELS:
    print(f"=== {model} ===")
    resp = client.chat.completions.create(
        model=model,
        messages=[
            {"role": "system", "content": "You are a concise assistant."},
            {"role": "user", "content": "Reply in one short Chinese sentence introducing yourself."},
        ],
        temperature=0.2,
        max_tokens=80,
    )
    print(resp.choices[0].message.content)
    print()
'@ | python -
```

### 11.3 成功标准

你应该能依次看到三段模型回复。

如果只有前两个成功、第三个失败，优先回去排查：

- `ollama --version`
- `ollama pull ministral-3:8b`
- Ollama 是否需要更新

## 12. 第七步：记录三种模型的基础信息

你后面要做实验，最好现在就记录基线。

建议你做一个表，至少记这几项：

| 模型 | 标签 | 是否成功加载 | 首次下载体积 | 首次响应时间 | 中文回复是否正常 |
|---|---|---|---|---|---|
| Llama 3 8B | `llama3:8b` |  |  |  |  |
| Qwen3 8B | `qwen3:8b` |  |  |  |  |
| Ministral 3 8B | `ministral-3:8b` |  |  |  |  |

建议你额外记录：

- GPU 型号
- 显存大小
- Ollama 版本
- Windows 版本

这会影响你后面实验结果的可复现性。

## 13. 第八步：为当前 PAIR 仓库补最基本的 Python 依赖

即使你现在还不改代码，也建议先把仓库依赖补齐，避免后面才发现环境不完整。

在同一个虚拟环境里执行：

```powershell
pip install fschat litellm wandb pandas psutil openai anthropic google-generativeai
```

如果你后面还要用 `JailbreakBench`：

```powershell
pip install jailbreakbench
```

### 13.1 做一次 import 自检

```powershell
@'
import main
import conversers
import judges
import language_models
print("PAIR imports ok")
'@ | python -
```

如果这里报错，就先修依赖，不要进入下一阶段。

## 14. 第九步：什么时候算“Windows 下已经跑通”

满足下面这些条件，就算你已经把 Windows 本地模型流程跑通了：

1. `ollama --version` 正常
2. `ollama list` 能看到三个模型
3. `ollama run llama3:8b` 正常回复
4. `ollama run qwen3:8b` 正常回复
5. `ollama run ministral-3:8b` 正常回复
6. `http://localhost:11434/api/chat` 能返回三种模型的结果
7. Python `OpenAI(base_url="http://localhost:11434/v1/")` 能访问三种模型
8. 当前 `PAIR` 仓库主要 Python 模块能 import 成功

做到这里，就可以认为：

- **Windows 原生多模型本地服务已经准备好**
- 下一步才是 **接入 PAIR 仓库**

## 15. 常见问题

### 15.1 `ollama` 命令找不到

通常是这几种情况：

- 装完后没重新开终端
- Ollama 没有正确加入 PATH
- 安装没完成

先关掉 PowerShell，重新打开再试：

```powershell
ollama --version
```

### 15.2 模型下载很慢

这是正常现象，尤其第一次拉模型。

先做两件事：

- 确保模型目录不在系统盘拥挤区域
- 先只拉一个模型验证流程

### 15.3 显存不够

Windows 原生最常见的问题之一就是显存或系统内存吃紧。

排查顺序建议：

1. 先只开一个模型，不要并发
2. 关闭其他吃显存的软件
3. 先让 `llama3:8b` 单独跑通
4. 再去试 `qwen3:8b`
5. 最后处理 `ministral-3:8b`

### 15.4 `ministral-3:8b` 报版本问题

优先怀疑 Ollama 版本。

因为官方模型页当前明确写了它需要 `Ollama 0.13.1`。

### 15.5 Qwen3 用 Transformers 本地跑时报 `KeyError: 'qwen3'`

这是官方 Qwen3 模型页明确提示的问题。

如果你不是走 Ollama，而是想直接走 Transformers，本地版本需要足够新；官方页明确写到：

- `transformers < 4.51.0` 可能报这个错误

## 16. 当前阶段不建议你马上做的事

在 Windows 原生本地模型流程还没跑稳之前，不建议你马上做下面这些事：

- 不要一开始就尝试 3 个模型同时并发服务
- 不要一开始就接复杂评测框架
- 不要一开始就去碰 WSL2 + vLLM + 多模型混合部署
- 不要一开始就把“环境问题”和“PAIR 代码问题”混在一起查

最稳的顺序就是：

1. Windows 原生把三种模型都跑通
2. PowerShell API 跑通
3. Python OpenAI 兼容访问跑通
4. 仓库依赖补齐
5. 再进入 PAIR 接入阶段

## 17. 下一步建议

当你完成本文所有步骤后，下一份文档建议单独处理：

`Windows 本地三模型接入 PAIR 仓库改造说明`

那份文档再专门讲：

- `config.py` 要怎么扩模型名
- `main.py` 参数怎么加
- `language_models.py` 怎么接本地 Ollama OpenAI 兼容接口
- 先让哪一个模型作为 `target` 跑最稳

把“环境跑通”和“代码接入”分开，问题会少很多。

## 18. 官方参考链接

以下是本文用到的主要官方来源：

- Ollama Windows 文档
  https://docs.ollama.com/windows

- Ollama OpenAI compatibility
  https://docs.ollama.com/api/openai-compatibility

- Llama 3 on Ollama
  https://ollama.com/library/llama3

- Qwen3 on Ollama
  https://ollama.com/library/qwen3

- Qwen/Qwen3-8B 模型页
  https://huggingface.co/Qwen/Qwen3-8B

- Ministral 3 on Ollama
  https://ollama.com/library/ministral-3:8b

- Mistral 官方介绍页
  https://mistral.ai/news/mistral-3
