# Linux 无 Ollama 实验指导

## 1. 目标

这份文档只做一件事：

在你租用的 **Linux GPU 环境** 中，**不用 Ollama**，从环境准备、模型服务、仓库依赖、代码跑通，到正式实验执行，全部在 Linux 上完成。

本文采用的唯一技术路线是：

- `Ubuntu 22.04`
- `Python 3.11`
- `Hugging Face + vLLM`
- `vLLM OpenAI-compatible server`
- `PAIR` 仓库

我不再给你多个方案。为了省时间，路线固定。

## 2. 先给结论

如果你要在 Linux 租卡环境里尽快跑完实验，又不想碰 Ollama，**最稳的路线就是 vLLM**。

原因如下：

- vLLM 官方 Quickstart 明确支持：
  - `Linux`
  - `Python 3.10 -- 3.13`
  - `vllm serve` 直接起 OpenAI-compatible server  
  来源：  
  https://docs.vllm.ai/en/v0.11.2/getting_started/quickstart/  
  https://docs.vllm.ai/en/v0.8.0/serving/openai_compatible_server.html

- Qwen 官方 `Qwen3-8B` 模型页明确写了：
  - 推荐最新 `transformers`
  - 部署时可以直接用 `vllm>=0.8.5`
  来源：  
  https://huggingface.co/Qwen/Qwen3-8B

- Mistral 官方 `Ministral-8B-Instruct-2410` 模型页明确写了：
  - 推荐使用 `vLLM`
  - 单卡运行需要 **24GB GPU RAM**
  来源：  
  https://huggingface.co/mistralai/Ministral-8B-Instruct-2410

所以这份文档的判断很直接：

- **Llama 3 8B**：走 vLLM
- **Qwen3 8B**：走 vLLM
- **Ministral 8B**：走 vLLM

## 3. 这条路线对硬件的含义

### 3.1 GPU 选择

如果你在 `V100` 和 `A800` 之间选，优先级还是：

- **A800 优先**
- **V100 其次**

尤其是你要跑 `Ministral 8B` 的时候更要注意。

Mistral 官方模型页明确写了：

- `Ministral-8B-Instruct-2410` 单卡需要 **24GB GPU RAM**

这意味着：

- `V100 16GB`：不建议拿来跑 `Ministral 8B`
- `V100 32GB`：可以尝试
- `A800 40GB/80GB`：更稳

### 3.2 一次只跑一个目标模型

预算有限时，不要同时把三个模型都作为服务挂起来。

正确做法是：

1. 启一个模型的 vLLM server
2. 跑这一组实验
3. 关掉
4. 再启动下一个模型

这能显著降低显存压力，也更容易排错。

## 4. 当前仓库的现实情况

在你正式开跑之前，要先认清一个事实：

**当前仓库不是拿来就能直接跑 `Llama 3 / Qwen3 / Ministral` 的。**

我根据仓库代码做的判断是：

- [main.py](/d:/大创/PAIR/main.py) 当前 `--attack-model` / `--target-model` 的 `choices` 没有这三个模型。
- [config.py](/d:/大创/PAIR/config.py) 当前模型枚举和模板映射里也没有这三个模型。
- [language_models.py](/d:/大创/PAIR/language_models.py) 当前 `APILiteLLM` 没有现成的“本地 vLLM `api_base`”接口配置。
- [conversers.py](/d:/大创/PAIR/conversers.py) 当前 target 侧默认还带 `JailbreakBench` 路径。

所以你要把这篇文档理解成：

1. 先在 Linux 上把 **本地模型服务 + Python 环境 + 依赖** 全部跑通
2. 再在同一台 Linux 机器上，把 `PAIR` 仓库的本地模型接入改造做完
3. 再执行实验

这份文档会把这个顺序写完整。

## 5. 推荐的 Linux 基础镜像

如果你还能选镜像，推荐：

- `Ubuntu 22.04`
- NVIDIA 驱动已安装
- 能正常访问外网
- 最好有 `git`、`python3`、`python3-venv`

如果镜像自带 Python 3.12，也能用，但对当前仓库来说，我还是建议你单独创建 **Python 3.11** 环境。

原因不是 vLLM 不支持 3.12，而是你这个仓库依赖比较杂，3.11 通常更省事。

## 6. 第一步：登录后立刻做基础检查

```bash
nvidia-smi
uname -a
python3 --version
df -h
free -h
```

你要确认四件事：

1. GPU 型号是不是你租的那张
2. 驱动正常
3. 系统盘 / 数据盘空间够用
4. 内存不是小得离谱

如果这一步都不对，不要继续装东西。

## 7. 第二步：准备 Python 3.11 环境

如果系统里已经有 `python3.11`：

```bash
python3.11 -m venv ~/pair-venv
source ~/pair-venv/bin/activate
python -m pip install --upgrade pip setuptools wheel
```

如果没有 `python3.11`，你有两个选择：

1. 用系统自带 Python 3.12 先创建环境
2. 或者安装 `micromamba/conda` 再建 3.11 环境

为了少折腾，这里先给最直接的建议：

- **有 3.11 就用 3.11**
- **没有就先用 3.12 跑 vLLM 和模型服务，但仓库依赖部分要额外小心**

## 8. 第三步：安装基础依赖

先装系统层最小工具：

```bash
sudo apt-get update
sudo apt-get install -y git curl wget build-essential
```

再装 Python 层基础包：

```bash
pip install --upgrade uv
```

## 9. 第四步：安装 vLLM

vLLM 官方 Quickstart 直接推荐：

```bash
uv venv --python 3.11 --seed
source .venv/bin/activate
uv pip install vllm --torch-backend=auto
```

如果你已经在 `~/pair-venv` 里了，也可以直接：

```bash
uv pip install vllm --torch-backend=auto
```

安装完先验证：

```bash
python -c "import vllm; print(vllm.__version__)"
vllm --help
```

## 10. 第五步：准备 Hugging Face 访问权限

你至少要确认下面三个模型的访问情况：

- `meta-llama/Meta-Llama-3-8B-Instruct`
- `Qwen/Qwen3-8B`
- `mistralai/Ministral-8B-Instruct-2410`

### 10.1 登录 Hugging Face

```bash
pip install huggingface_hub
huggingface-cli login
```

### 10.2 你要知道的权限区别

- `Llama 3 8B` 通常需要你在 Hugging Face 上接受 Meta 许可
- `Qwen3-8B` 是 `Apache-2.0`
- `Ministral-8B-Instruct-2410` 在模型页上标的是 `Mistral Research License`

所以在租卡前你最好就已经确认过：

- 这三个模型你都能访问

## 11. 第六步：先单独把三个模型服务跑通

### 11.1 Llama 3 8B

开一个终端，起服务：

```bash
vllm serve meta-llama/Meta-Llama-3-8B-Instruct \
  --dtype auto \
  --api-key local-vllm \
  --host 127.0.0.1 \
  --port 8000 \
  --max-model-len 4096
```

另开一个终端测试：

```bash
python - <<'PY'
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8000/v1",
    api_key="local-vllm",
)

resp = client.chat.completions.create(
    model="meta-llama/Meta-Llama-3-8B-Instruct",
    messages=[{"role": "user", "content": "Reply with one short Chinese sentence introducing yourself."}],
    max_tokens=64,
    temperature=0.2,
)

print(resp.choices[0].message.content)
PY
```

### 11.2 Qwen3 8B

Qwen 官方模型页明确写了：

- 推荐 `transformers` 最新版
- `vllm>=0.8.5`
- 可以用 `vllm serve Qwen/Qwen3-8B --enable-reasoning --reasoning-parser deepseek_r1`

先用官方建议的最小服务启动：

```bash
vllm serve Qwen/Qwen3-8B \
  --reasoning-parser qwen3 \
  --dtype auto \
  --api-key local-vllm \
  --host 127.0.0.1 \
  --port 8000 \
  --max-model-len 4096
```

再测：

```bash
python - <<'PY'
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8000/v1",
    api_key="local-vllm",
)

resp = client.chat.completions.create(
    model="Qwen/Qwen3-8B",
    messages=[{"role": "user", "content": "Reply with one short Chinese sentence introducing yourself."}],
    max_tokens=64,
    temperature=0.2,
)

print(resp.choices[0].message.content)
PY
```

说明：

- Qwen3 默认可能带 reasoning 行为
- 对后续实验来说，这会影响输出长度和速度
- 如果你后面发现它显著拖慢实验，应该在接仓库时统一处理“thinking / non-thinking”策略

### 11.3 Ministral 8B

Mistral 官方模型页直接推荐 vLLM，并给了服务命令。

启动：

```bash
pip install --upgrade mistral_common

vllm serve mistralai/Ministral-8B-Instruct-2410 \
  --tokenizer_mode mistral \
  --config_format mistral \
  --load_format mistral \
  --dtype auto \
  --api-key local-vllm \
  --host 127.0.0.1 \
  --port 8000 \
  --max-model-len 4096
```

测试：

```bash
python - <<'PY'
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8000/v1",
    api_key="local-vllm",
)

resp = client.chat.completions.create(
    model="mistralai/Ministral-8B-Instruct-2410",
    messages=[{"role": "user", "content": "Reply with one short Chinese sentence introducing yourself."}],
    max_tokens=64,
    temperature=0.2,
)

print(resp.choices[0].message.content)
PY
```

## 12. 第七步：准备当前 PAIR 仓库依赖

当前仓库没有完整 `requirements.txt`，所以你要手动补。

进入仓库：

```bash
cd ~/PAIR
```

建议先单独建一个项目虚拟环境：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip setuptools<81 wheel
```

再装当前仓库用得到的基础依赖：

```bash
pip install fschat litellm wandb pandas psutil openai anthropic google-generativeai huggingface_hub vllm jailbreakbench mistral_common
```

### 12.1 先做 import 自检

```bash
python - <<'PY'
import main
import conversers
import judges
import language_models
print("PAIR imports ok")
PY
```

如果这里报错，不要继续。

## 13. 第八步：把当前仓库改到能调用本地 vLLM

这一步是必须的。

原因很简单：

- 当前 [main.py](/d:/大创/PAIR/main.py) 还没有这三个模型的 CLI 选项
- 当前 [config.py](/d:/大创/PAIR/config.py) 没有这三个模型的映射
- 当前 [language_models.py](/d:/大创/PAIR/language_models.py) 也没有直接面向本地 `base_url` 的配置

### 13.1 最少要改哪些文件

你至少要处理：

- [config.py](/d:/大创/PAIR/config.py)
  新增 `Llama 3 / Qwen3 / Ministral` 的模型枚举、模板名、API key 名或本地服务占位映射。

- [main.py](/d:/大创/PAIR/main.py)
  把这三个模型加到 `--attack-model` / `--target-model` 可选项中。

- [language_models.py](/d:/大创/PAIR/language_models.py)
  给 `APILiteLLM` 增加面向本地 vLLM 的 `base_url` / `api_key` / 模型名路由能力。

- [conversers.py](/d:/大创/PAIR/conversers.py)
  让 target 侧可以走本地 vLLM，而不是默认偏向 JailbreakBench 路线。

### 13.2 你现在不应该做什么

不要在实验当天一边跑一边改这些文件。

正确顺序是：

1. 先把 vLLM 服务跑通
2. 先把仓库依赖跑通
3. 再做模型接入改造
4. 用无害样例做 dry run
5. 最后才跑正式实验

## 14. 第九步：先做一次无害 dry run

在你真正跑正式实验前，先用一个完全无害的目标把整条链路打通。

例如你可以先验证类似这种 benign 目标：

```bash
python main.py \
  --attack-model gpt-3.5-turbo-1106 \
  --target-model <你的本地模型选项> \
  --judge-model no-judge \
  --goal "Write a short poem about the ocean" \
  --target-str "Sure, here is a short poem about the ocean" \
  --n-streams 1 \
  --n-iterations 1 \
  -v
```

这里的重点不是实验内容，而是确认：

1. 你的本地 vLLM 服务可达
2. 仓库能发请求
3. 响应能正确回流
4. 日志和输出不报错

## 15. 第十步：正式实验怎么排顺序

预算有限时，不要一开始把三个模型全部铺开。

推荐顺序：

1. 先用最稳的一个模型完成全链路
2. 再复制相同流程到第二个模型
3. 最后处理最容易吃显存或最慢的那个模型

建议顺序：

1. `Meta-Llama-3-8B-Instruct`
2. `Qwen/Qwen3-8B`
3. `mistralai/Ministral-8B-Instruct-2410`

原因：

- Llama 3 8B 的路径最标准
- Qwen3 8B 要额外考虑 reasoning 行为
- Ministral 8B 对显存要求最敏感

## 16. 第十一步：正式实验执行原则

### 16.1 一次只挂一个模型服务

不要三开。

### 16.2 先用小设置验证

第一次正式跑，建议：

- `n_streams=1`
- `n_iterations=1`
- 最小必要日志

确认无误后，再逐步增加。

### 16.3 结果实时落盘

不要等实验结束才保存。

无论你最后用 `wandb` 还是本地文件，至少要保证：

- 每一轮实验都有落盘
- 中途掉线不会全丢

### 16.4 先跑最关键组合

不要按“模型名字母顺序”跑，要按“最值钱的结果优先”跑。

## 17. 第十二步：什么时候算全部跑通

当你满足下面这些条件，就算 Linux 全链路已经打通：

1. `vllm serve` 能成功起 `Llama 3 8B`
2. `vllm serve` 能成功起 `Qwen3 8B`
3. `vllm serve` 能成功起 `Ministral 8B`
4. 你能用 OpenAI Python 客户端打通这三个服务
5. 当前 `PAIR` 仓库主要模块 import 正常
6. 仓库已经改到能识别并调用本地 vLLM 模型
7. benign dry run 已完成
8. 至少一组正式实验已完成并成功保存结果

## 18. 这条路线最常见的坑

### 18.1 Qwen3 报 `KeyError: 'qwen3'`

这是官方模型页点名提醒的问题。优先升级 `transformers`。

### 18.2 Ministral 8B 显存不够

官方模型页已经明说单卡需要 24GB。  
如果你是 `V100 16GB`，先别怀疑命令。

### 18.3 vLLM 服务能起，但 chat 请求报模板错

vLLM 官方 OpenAI-compatible server 文档明确说明：

- 模型要有 chat template
- 没有就要手工传 `--chat-template`

如果你碰到这个问题，就先查模型 tokenizer 配置里有没有模板。

### 18.4 代码能 import，但实验跑不起来

通常不是 Linux 问题，而是：

- 你的本地模型还没接进当前仓库
- 或者 target 还在走 JailbreakBench 分支

## 19. 官方参考链接

- vLLM Quickstart  
  https://docs.vllm.ai/en/v0.11.2/getting_started/quickstart/

- vLLM OpenAI-compatible Server  
  https://docs.vllm.ai/en/v0.8.0/serving/openai_compatible_server.html

- Meta-Llama-3-8B-Instruct 模型页  
  https://huggingface.co/meta-llama/Meta-Llama-3-8B-Instruct

- Qwen3-8B 模型页  
  https://huggingface.co/Qwen/Qwen3-8B

- Ministral-8B-Instruct-2410 模型页  
  https://huggingface.co/mistralai/Ministral-8B-Instruct-2410

- Mistral 3 官方介绍  
  https://mistral.ai/news/mistral-3

## 20. 最后一条建议

如果你现在就是要“今天把实验跑起来”，那就不要再分神想别的栈。

就按这条固定路线走：

1. Linux
2. Python 3.11
3. vLLM
4. 先起模型服务
5. 再接仓库
6. 先 benign dry run
7. 再正式实验

别再回头碰 Ollama。
