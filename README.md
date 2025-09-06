# RLTrainPPT: 基于强化学习的PPT Agent 训练模型

## 目标
- **训练PPT大纲Agent**: 根据输入的主题，通过多轮网络搜索，自动生成结构合理、内容丰富的PPT大纲。
- **训练PPT内容Agent**: 根据给定的大纲，通过网络搜索和图片搜索，自动填充每页PPT的具体内容，确保来源可靠、格式正确。

## 训练的GPU资源
采用的是智星云的平台,1-2RMB1小时，还不错。网址：https://gpu.ai-galaxy.cn/

## 项目组件

- **[LLMCache](backend/LLMCache)**: 一个为大语言模型（LLM）请求提供缓存和代理的服务。主要用于在国内环境中稳定访问OpenAI等外部API，并缓存请求结果以节约成本和时间。

- **[Clash代理](backend/clash)**: 一个Clash容器化配置，为`LLMCache`和其他需要访问外部网络的服务提供稳定的网络代理。

- **[训练PPT生成大纲 (ART_Langgraph_outline)](backend/ART_Langgraph_outline)**: 接收一个主题（Topic），通过强化学习训练一个能够进行多轮网络搜索、分析、总结，并最终生成一个结构化Markdown大纲的智能体。

- **[训练PPT内容Agent (ART_Langgraph_content)](backend/ART_Langgraph_content)**: 接收一个结构化的PPT大纲（JSON格式），通过强化学习训练一个能够自动研究和填充内容的智能体。它会为大纲的每个要点进行网络搜索，生成详细文本，并附上引用来源。

## 核心技术

- **Agentic RL Transformer (ART)**: Google DeepMind开源的强化学习框架，本项目使用其 `GRPO` (Generative Rollout Policy Optimization) 算法对模型进行优化。
- **LangGraph**: 一个用于构建可循环、有状态的、基于LLM的应用的库。本项目用它来构建 ReAct (Reasoning and Acting) 风格的智能体。
- **智谱AI SDK (`zhipuai`)**: 作为智能体获取外部信息的网页搜索工具。
- **Weights & Biases (`wandb`)**: 用于记录和可视化训练过程，方便监控和分析模型性能。

## 工作流程

### 1. 大纲生成 (Outline Generation)
1.  **输入**: 用户提供一个PPT主题，例如“AIGC在医疗影像的应用趋势”。
2.  **智能体处理 (`Rollout`)**:
    -   智能体接收主题，并生成多个搜索查询。
    -   调用`web_search_tool`工具执行搜索，并分析结果。
    -   基于搜索结果，生成一个结构化的Markdown大纲，包含标题、5个一级部分、每个一级部分下设3-4个二级小节，以及每个小节的要点。
    -   调用`return_final_outline_tool`工具返回最终的大纲和参考的URL列表。
3.  **奖励计算**: 系统根据输出大纲的结构完整性、内容相关性、来源多样性等维度计算奖励。
4.  **模型训练**: `art`框架根据奖励使用`GRPO`算法对模型进行微调。

### 2. 内容填充 (Content Filling)
1.  **输入**: 提供一个结构化的PPT大纲JSON，其中`text`字段为待填充的占位符。
2.  **智能体处理 (`Rollout`)**:
    -   智能体遍历大纲中的每一个待填充项。
    -   针对每一项生成精确的搜索查询，并调用`web_search_tool`。
    -   分析搜索结果，提炼关键信息，生成2-4句话的文本内容，并在末尾附上引用标记（如`[1]`, `[2]`）。
    -   将填充好的内容放回原始JSON结构中。
3.  **输出**: 智能体调用`return_filled_outline_tool`工具，返回包含所有填充内容和引用URL列表的完整JSON。
4.  **奖励计算**: 系统根据输出的格式一致性、内容填充完整度、引用标注准确性、来源多样性等计算奖励。
5.  **模型训练**: `art`框架根据奖励再次对模型进行微调。

## 项目结构

```
RLTrainPPT/
├── backend/
│   ├── LLMCache/              # LLM 缓存与代理服务
│   ├── clash/                 # 网络代理容器
│   ├── ART_Langgraph_outline/ # PPT 大纲生成 Agent
│   └── ART_Langgraph_content/ # PPT 内容填充 Agent
├── ART/                       # ART 框架 (子模块)
└── README.md                  # 本文件
```

## 快速开始

### 1. 环境准备
-   Python 3.10 或更高版本。
-   Docker (用于运行 `clash` 代理)。
-   智谱AI API密钥。
-   (可选) OpenAI API密钥。
详细参考文档：
[prepare.md](doc/prepare.md)

### 2. 启动依赖服务 (代理和缓存)

#### 1. 启动Clash代理 (如果需要)
如果你的环境无法直接访问OpenAI等服务，请先配置和启动Clash代理。
```bash
cd backend/clash

# 替换 config.yaml 为你自己的Clash配置文件
# cp /path/to/your/config.yaml ./config.yaml

# 启动clash服务
./clash-linux-amd64-v1.10.0 -d .

# 或者使用Docker
# docker build -t clash .
# docker run -p 7890:7890 -p 9090:9090 clash
```

#### 2. 启动LLMCache服务
该服务为后续的Agent训练提供统一的LLM接口和缓存。
```bash
cd backend/LLMCache

# (推荐) 创建并激活虚拟环境
python -m venv venv
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量 (拷贝并修改.env文件)
cp env_template .env
# 在.env文件中填入你的API密钥和代理地址
# OPENAI_API_KEY=sk-proj-xxx
# ZHIPU_API_KEY=xxxxx
# HTTP_PROXY=http://127.0.0.1:7890
# HTTPS_PROXY=http://127.0.0.1:7890

# 启动缓存服务
python LLM_cache.py
```

### 3. 训练PPT大纲Agent
```bash
cd backend/ART_Langgraph_outline

# (推荐) 创建并激活虚拟环境
python -m venv venv
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp env_template .env
# 在.env中填入Wandb和API密钥信息

# 运行训练
python train.py

# 运行测试
python model_test.py
```

### 4. 训练PPT内容Agent
```bash
cd backend/ART_Langgraph_content

# (推荐) 创建并激活虚拟环境
python -m venv venv
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 配置环境变量
cp env_template .env
# 在.env中填入Wandb和API密钥信息

# 运行训练
python train.py

# 运行测试
python model_test.py
```

### 5. 在PPT中使用训练完成的模型
- 合并训练后生成的LoRA模型与基础模型, 使用程序：[merge_lora.py](doc/merge_lora.py)
- 进入容器，使用Ollama或VLLM等工具加载合并后的模型，并以兼容OpenAI的模式提供API服务。 
```bash
# 进入合并后的模型目录的上一级，然后使用vllm运行该模型
python -m vllm.entrypoints.openai.api_server --host 0.0.0.0 --model qwen2.5-7b-contentmodel
# 测试模型
curl http://localhost:8000/v1/chat/completions \
  -H "Authorization: Bearer token-abc123" -H "Content-Type: application/json" \
  -d '{
    "model":"qwen2.5-7b-contentmodel",
    "messages":[{"role":"user","content":"你好！"}]
  }'

curl http://localhost:8000/v1/completions     -H "Content-Type: application/json"     -d '{"model": "qwen2.5-7b-contentmodel","prompt": "你好", "max_tokens": 100,"temperature": 0}'
输出:
{"id":"chatcmpl-b8b7c3c2d82c4241bebe1c7bec94c9b2","object":"chat.completion","created":1756992891,"model":"qwen2.5-7b-contentmodel","choices":[{"index":0,"message":{"role":"assistant","reasoning_content":null,"content":"你好！很高兴为你服务。有什么我可以帮助你的吗？","tool_calls":[]},"logprobs":null,"finish_reason":"stop","stop_reason":null}],"usage":{"prompt_tokens":31,"total_tokens":44,"completion_tokens":13,"prompt_tokens_details":null},"prompt_logprobs":null}

```
- 在TrainPPTAgent程序中，通过LiteLLM客户端库调用该模型服务接口。
```bash
编辑backend/simpleOutline/.env
MODEL_PROVIDER=vllm
LLM_MODEL=qwen2.5-7b-contentmodel
VLLM_API_KEY=EMPTY
VLLM_API_URL=http://127.0.0.1:8000/v1
```

## 有任何问题联系我
![weichat.png](doc/weichat.png)
