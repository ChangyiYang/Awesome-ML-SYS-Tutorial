# SGLang & veRL: Pioneering End-to-End Multi-Turn RLHF
by: The SGLang Team, May 03, 2025

We are thrilled to announce the release of the first fully functional, convergence-verified, end-to-end open source multi-turn Reinforcement Learning with Human Feedback (RLHF) framework, powered by SGLang and integrated with veRL.

After two months of intense development and a final five-day sprint, our team has delivered a robust solution that enables asynchronous multi-turn dialogues and tool-calling in Agentic RL. This release marks a significant step forward in scalable RLHF for large language models.

Pull Request: [volcengine/verl#1037](https://github.com/volcengine/verl/pull/1037)

Training performance: https://docs.net9.org/notes/editor/#_8

## What Problem We Solved?

Multi-turn RLHF is critical for training language models to handle complex and interactive dialogues, such as coding tasks, where the outputs of the generated scripts are needed as feedback for the next generation. However, existing frameworks lacked multi-turn rollout support, with a standardized tool calling method.

Our goals are：

- Support asynchronous multi-turn interactions with high efficiency.  
- Integrate tool-calling into RLHF workflows in an easily-extensible style.  
- Prove our training performance on complex, multi-turn suitable tasks.  
- Operate reliably in large-scale, distributed environments.  

## Our Solution

1. **From Batch-Level to Request-Level Rollout** : We extended veRL’s rollout interface to support asynchronous, per-request multi-turn interactions, where each dialogue can independently span multiple rounds and tool calls, decoupling from the batch structure.
2. **Generalized Tool Calling with Unified Schema**: We support `OpenAIFunctionToolSchema`, convertible with veRL’s Model Context Protocol / Agent2Agent Protocol. This allows seamless tool integration across both training and inference workflows.
3. **Parameter Injection Mechanism**: Users can dynamically select tools during sampling (e.g., via `need_tools_kwargs`) and inject call parameters (`tool_kwargs`), streamlining tool integration.
4. **GRPO Policy Gradient**: We adopt the GRPO strategy (`adv_estimator = grpo`) for stable reward propagation over multi-turn sequences.

## How to Use It

### Environment Setup

#### Create a New Docker Container

```bash
docker run \
    -it \
    --shm-size 32g \
    --gpus all \
    -v /models/shared/.cache:/root/.cache \
    --ipc=host \
    --network=host \
    --privileged \
    --name sglang_{your-name} \
    lmsysorg/sglang:dev \
    /bin/zsh
```

To restart the container after exiting from it:

```bash
docker start -i sglang_{your-name}
```

#### Update Python

```bash
apt update
apt install -y python3.10 python3.10-venv
python3 -m ensurepip --upgrade
```

#### Use Python Virtual Environment

```bash
# Create a virtual environment
python3 -m venv ~/.python/veRL-multiturn-rollout

# Activate the virtual environment
source ~/.python/veRL-multiturn-rollout/bin/activate

# Install uv
python3 -m pip install uv
```

#### Clone the veRL Main Repository

```bash
cd ~
git clone https://github.com/volcengine/verl.git
cd verl
```

#### Set Up the Python Environment

```bash
# Install SGLang
python3 -m uv pip install -e ".[sglang]"

# Manually install flash-attn
python3 -m uv pip install wheel
python3 -m uv pip install packaging
python3 -m uv pip install flash-attn --no-build-isolation --no-deps

# Install veRL
python3 -m uv pip install .
python3 -m uv pip install -r ./requirements.txt
```

### 8-GPU SGLang Test

#### Set Up Your `WANDB_API_KEY` Before Running

Refer to [this guide](https://community.wandb.ai/t/where-can-i-find-the-api-token-for-my-project/7914) if you're not sure where to find your API token.

```bash
export WANDB_API_KEY={YOUR_WANDB_API_KEY}

# Define a timestamp function
function now() {
    date '+%Y-%m-%d-%H-%M'
}
```

#### Download the Dataset

```bash
python3 ./examples/data_preprocess/gsm8k_multiturn_w_tool.py
```

#### Run

```bash
# First make sure the now() function is available in current shell
# Create logs directory if it doesn't exist
mkdir -p logs

# Set GPUs and run with better log organization
export CUDA_VISIBLE_DEVICES=0,1,2,3,4,5,6,7

nohup bash examples/sglang_multiturn/run_qwen2.5-3b_gsm8k_multiturn.sh trainer.experiment_name=qwen2.5-3b_rm-gsm8k-sgl-multiturn-$(now) > logs/gsm8k-$(now).log 2>&1 &
```

### Configurations

#### Basic Configuration

To enable multi-turn rollout, make sure to configure the following fields in your rollout configuration:

```yaml
actor_rollout_ref:
  rollout:
    name: "sglang_async"
    multi_turn:
      enable: True
```

This configuration activates the AsyncSGLangRollout engine for multi-turn interaction during rollout.

#### Custom Tool Configuration

Tools are a critical component of our framework, which enables environment interactions, such as executing scripts or calculating rewards. To integrate custom tools, you can define their behavior in a separate YAML file and reference it in the rollout configuration. Here’s how to set it up:

```yaml
actor_rollout_ref:
  rollout:
    multi_turn:
      tool_config_path: <path_to_tool_yaml_file>
      format: chatml
```

- `tool_config_path` specifies the YAML file containing tool definitions.
- `format`  indicates the format for tool interaction messages (currently supports `chatml` only).

#### Example: GSM8K Tool Configuration

To illustrate how to configure a tool, we provide an example based on the Gsm8kTool, designed for evaluating answers to GSM8K math problems to calculate rewards. The tool configuration is defined [here](https://github.com/volcengine/verl/blob/main/examples/sglang_multiturn/config/tool_config/gsm8k_tool_config.yaml).

Breakdown of the Configuration:

- **class_name**: Specifies the Python class implementing the tool’s logic. The class must be accessible in the codebase.
- **config**: An optional field for additional tool-specific configurations (e.g., API keys, model parameters).
- **tool_schema**: Defines the tool’s interface using a schema compatible with OpenAIFunctionToolSchema or veRL’s protocols.

- **type: "function"**: Indicates the tool is a function-based tool.
- **function.name**: The tool’s identifier (calc_gsm8k_reward), used during tool selection.
- **function.description**: A human-readable description of the tool’s purpose.
- **function.parameters**: Describes the input parameters the tool expects. The required field defines the mandatory parameters..

## Challenges, Methods, and Open Issues

- **Padding Strategy Mismatch**: veRL adopts a left-padding strategy for prompts and a right-padding strategy for responses, which introduces a padding inconsistency with our initial version of code. To mitigate this, our implementation explicitly tracks token positions across segments and applies tailored masks to maintain correctness.
- **Multi-Turn Loss Masking:** Most existing RLHF frameworks assume a single-turn generation pattern and lack support for granular, token-level loss masking across multiple dialogue turns. However, in multi-turn settings—especially with tool interactions—not every generated token should contribute to the learning signal. For example, some tool-generated responses should be excluded from the optimization process. We addressed this by designing a custom multi-turn loss masking mechanism, allowing fine-grained control over which tokens are included in policy gradient updates, thereby ensuring an accurate reward computation.
- **Generalized Tool API Design**: Environment in the RLHF training scenarios could be complicated, and customized tools are needed for agents to interact with the outside world. To support flexible and reusable tool integration, we designed a generalized tool interface. This design enables users to register tools with their customized functions into the rollout process. By unifying tool definitions in a schema-driven format, we make our framework highly extensible to easily add tools and reuse them across tasks and models without much modification.
- **SPMD Conflicts in Tool Invocation**: In tensor-parallel (TP) environments, invoking external tools—such as API calls, script evaluators, or reward calculators—must be controlled to avoid concurrency issues. A naive implementation may result in the same tool being called multiple times in parallel across TP ranks, causing inconsistencies or deadlocks. To avoid this, all tool invocations are restricted to TP rank 0, with results broadcast to other ranks. This avoids performance bottlenecks due to redundant calls.
- **Asynchronous Rollout for Multi-Turn Interactions**: Synchronous rollouts often suffer from the long-tail problem, where the slowest sample in a batch delays the entire pipeline. This issue is especially prominent in multi-turn tasks involving variable-length dialogues and tool calls. To address this, we implemented asynchronous rollout at the request level, allowing each dialogue to progress independently. 
- **NaN Losses**: During training, NaN losses were observed. We analyzed that it may be because of the extreme discrepancies between the actor and reference model log probabilities, which lead to unstable importance sampling ratios. This is often due to rare tokens or divergent generations. We addressed this by monitoring log probability statistics via wandb, tuning the KL penalty, and applying clip thresholds to prevent such instabilities from propagating.**Event Loop Conflicts:** SGLang already embeds an internal asyncio loop. Creating additional loops externally causes rollouts to hang indefinitely.
- **Event Loop Conflicts in Asynchronous Execution**: During testing, we encountered a problem: with enable_memory_saver on, async_generate got hung. After extensive investigation, we found that the root cause was the existence of multiple concurrent event loops, violating Python’s asyncio design. SGLang internally manages its own asynchronous event loop to coordinate token streaming, multi-turn interaction, and memory-efficient generation. We mistakenly added a second event loop, thus making the program stuck forever. Our fix ensures that all async execution happens within SGLang’s own loop by running the existing loop instead of invoking asyncio.run() inside async_generate.

## Acknowledgments

------

- 面壁
- 智谱
- SGLang RLHF Team

## Our Work Plan

------

- Integration of multi-turn RLHF-suitable tasks (e.g., R1-Searcher, sandbox fusion)
- Support more policy gradient estimators beyond GRPO
- Improve step-level reward integration
- Simulate human-in-the-loop for multi-turn RLHF
- Introduce micro-batching and agentic rollout loop

We welcome the community to collaborate with us to push forward the frontier of RLHF research and applications.