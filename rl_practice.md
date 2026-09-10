## 1. 工具与框架的关系

| 层次 | 工具 | 主要职责 |
| --- | --- | --- |
| 环境接口 | Gymnasium | 定义 `reset()`、`step()`、观察空间和动作空间等标准 |
| 测试环境 | Gymnasium environments | 提供 CartPole、Pendulum、LunarLander、MuJoCo 等任务 |
| RL 算法 | Stable-Baselines3 | 提供 PPO、DQN、SAC、TD3、A2C 等成熟实现 |
| 机器人仿真 | Isaac Sim | 提供物理、机器人、传感器、渲染和场景仿真 |
| 机器人学习 | Isaac Lab | 在 Isaac Sim 上构建并行环境、机器人任务和训练流程 |

整体关系如下：

```text
PPO / SAC / 其他算法
        ↓
SB3、RSL-RL、RL-Games
        ↓
Gymnasium 环境或 Isaac Lab 机器人任务
        ↓
Isaac Sim：物理、机器人、传感器、渲染
```

## 2. Gym

OpenAI Gym 是较早流行起来的强化学习环境接口。它通过统一 API 连接环境与算法，让同一个算法实现能够用于多个任务。

早期代码常见形式：

```python
obs = env.reset()
obs, reward, done, info = env.step(action)
```

Gym 已停止后续开发，GitHub 仓库于 2026 年 4 月归档。新项目直接使用 Gymnasium。详见 [OpenAI Gym 仓库](https://github.com/openai/gym)。

## 3. Gymnasium

Gymnasium 是 Gym 的持续维护版本，由 Farama Foundation 维护。标准交互方式如下：

```python
import gymnasium as gym

env = gym.make("CartPole-v1")
obs, info = env.reset(seed=42)

obs, reward, terminated, truncated, info = env.step(action)

if terminated or truncated:
    obs, info = env.reset()
```

结束标志的含义：

- `terminated=True`：任务自身达到终止状态，例如机器人摔倒或抵达目标。
- `truncated=True`：时间限制等外部条件导致 episode 停止。
- 一个 episode 结束时，可使用 `done = terminated or truncated` 判断是否需要重置环境。

两种结束情况会影响价值函数 bootstrap，应在手写 PPO、DQN 时认真处理。详见 [Gymnasium 迁移指南](https://gymnasium.farama.org/main/introduction/migration_guide/)。

常用环境：

- `CartPole-v1`：离散动作，适合 PPO、DQN 入门。
- `Pendulum-v1`：连续动作，适合 PPO、SAC 入门。
- `LunarLander-v3`：稍复杂的控制任务。
- FrozenLake、Blackjack：适合表格型 Q-learning。
- MuJoCo：适合连续控制和机器人动力学。
- Atari：适合图像观察和离散动作。

## 4. Stable-Baselines3

Stable-Baselines3，简称 SB3，是基于 PyTorch 的强化学习算法库，提供：

- PPO、DQN、SAC、TD3、A2C 等算法。
- 轨迹采样、GAE、PPO clipping 和网络优化。
- `check_env()` 自定义环境检查。
- `Monitor` 训练数据记录。
- `VecEnv` 多环境并行。
- `VecNormalize` 观察和奖励标准化。
- TensorBoard、评估、回调和模型保存。

简单示例：

```python
import gymnasium as gym
from stable_baselines3 import PPO

env = gym.make("CartPole-v1")

model = PPO(
    "MlpPolicy",
    env,
    verbose=1,
    tensorboard_log="./logs/",
)

model.learn(total_timesteps=100_000)
model.save("ppo_cartpole")
```

SB3 适合用来建立可靠基线。可以先验证任务是否能够被 PPO 解决，再编写自己的 PPO，并对比两者的训练曲线和最终回报。

官方资料：

- [SB3 安装文档](https://stable-baselines3.readthedocs.io/en/master/guide/install.html)
- [SB3 示例](https://stable-baselines3.readthedocs.io/en/master/guide/examples.html)
- [SB3 PPO 文档](https://stable-baselines3.readthedocs.io/en/master/modules/ppo.html)

## 5. Isaac Sim

Isaac Sim 是 NVIDIA 的机器人仿真平台，主要提供：

- PhysX/Newton 物理仿真。
- 刚体、关节、碰撞和接触。
- 摄像头、深度、激光雷达、IMU 等传感器。
- URDF、MJCF、USD 等机器人和场景格式。
- RTX 渲染与合成数据生成。
- ROS 2 集成。
- 机械臂、移动机器人、人形机器人等仿真能力。

Isaac Sim 负责构建机器人所处的虚拟世界。它对 NVIDIA GPU、驱动、内存和磁盘空间有较高要求，安装前应检查目标版本的兼容性。

官方资料：

- [Isaac Sim 官方介绍](https://docs.isaacsim.omniverse.nvidia.com/latest/index.html)
- [Isaac Sim Python 安装文档](https://docs.isaacsim.omniverse.nvidia.com/latest/installation/install_python.html)

## 6. Isaac Lab

Isaac Lab 建立在 Isaac Sim 之上，专门服务于机器人学习，主要提供：

- 大规模并行机器人环境。
- 机器人、执行器和传感器抽象。
- 观察、动作、奖励和终止条件配置。
- Domain Randomization。
- Curriculum Learning。
- 机械臂、四足和人形机器人等预制任务。
- SB3、RSL-RL、RL-Games、SKRL 等训练框架适配器。

Isaac Lab 环境通常使用批量 PyTorch tensor。接入 SB3 时需要 `Sb3VecEnvWrapper` 等适配器。

官方资料：

- [Isaac Lab Quickstart](https://isaac-sim.github.io/IsaacLab/main/source/setup/quickstart.html)
- [Isaac Lab 安装文档](https://isaac-sim.github.io/IsaacLab/main/source/setup/installation/pip_installation.html)
- [Isaac Lab 使用 SB3 训练](https://isaac-sim.github.io/IsaacLab/main/source/tutorials/03_envs/run_rl_training.html)

## 7. 当前阶段的安装建议

先创建一个轻量 Python 环境：

```bash
python -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
pip install "stable-baselines3[extra]" "gymnasium[classic-control]"
```

该组合包含或带入以下常用依赖：

- Gymnasium
- Stable-Baselines3
- PyTorch
- NumPy
- TensorBoard
- Matplotlib
- OpenCV 等可选工具

验证安装：

```bash
python -c "import gymnasium, stable_baselines3, torch; print(gymnasium.__version__, stable_baselines3.__version__, torch.__version__)"
```

当前阶段推荐组合：

```text
Python + PyTorch + Gymnasium + Stable-Baselines3 + TensorBoard
```

开始机器人强化学习时，再为 Isaac Sim 和 Isaac Lab 建立独立环境，并严格参考对应版本的兼容矩阵。

## 8. 其他按需了解的框架

- **CleanRL**：算法集中在单个脚本中，适合阅读 PPO 源码。
- **RL Baselines3 Zoo**：提供 SB3 训练、调参和复现实验工具。
- **RSL-RL**：常用于 GPU 并行机器人运动训练和 Isaac Lab。
- **TorchRL**：PyTorch 生态中的强化学习组件库。
- **Ray RLlib**：适合分布式训练和大规模实验。
- **Tianshou**：模块化的 PyTorch 强化学习框架。
- **PettingZoo**：多智能体环境标准。
- **MuJoCo**：高速机器人动力学仿真。
- **Brax**：基于 JAX 的大规模并行物理环境。
- **Weights & Biases**：远程记录和比较实验。

这些工具都可以在产生明确需求时再安装。

## 9. 推荐的 Demo 学习路线

| 顺序 | Demo | 主要收获 | 难度 |
| --- | --- | --- | --- |
| 1 | FrozenLake + Q-learning | 贝尔曼更新、Q 表、探索策略 | ★ |
| 2 | CartPole + SB3 PPO | 完整 PPO 训练流程 | ★ |
| 3 | CartPole + 手写 PPO | GAE、clip loss、minibatch update | ★★ |
| 4 | Pendulum + PPO/SAC | 连续动作和高斯策略 | ★★ |
| 5 | 自定义 GridWorld | 状态、动作、奖励和终止条件设计 | ★★ |
| 6 | Isaac Lab CartPole | GPU 并行环境和机器人仿真 | ★★★★ |

建议先完成前三项，形成“理论—成熟实现—自主实现”的闭环。

## 10. Demo 1：熟悉 Gymnasium 接口

创建 `random_cartpole.py`：

```python
import gymnasium as gym

env = gym.make("CartPole-v1", render_mode="human")

obs, info = env.reset(seed=42)
episode_reward = 0.0

while True:
    action = env.action_space.sample()

    next_obs, reward, terminated, truncated, info = env.step(action)

    episode_reward += reward
    obs = next_obs

    if terminated or truncated:
        print(f"episode reward: {episode_reward}")
        obs, info = env.reset()
        episode_reward = 0.0
```

运行：

```bash
python random_cartpole.py
```

重点观察：

- `obs` 的形状和每个元素的含义。
- `action_space` 有哪些动作。
- reward 在什么时候产生。
- `terminated` 和 `truncated` 何时出现。
- episode 怎样开始、结束和重置。

参考 [Gymnasium Basic Usage](https://gymnasium.farama.org/main/introduction/basic_usage/)。

## 11. Demo 2：使用 SB3 PPO 训练 CartPole

创建 `train_ppo.py`：

```python
from stable_baselines3 import PPO
from stable_baselines3.common.env_util import make_vec_env
from stable_baselines3.common.evaluation import evaluate_policy

train_env = make_vec_env(
    "CartPole-v1",
    n_envs=4,
    seed=42,
    monitor_dir="./logs/monitor/",
)

model = PPO(
    policy="MlpPolicy",
    env=train_env,
    learning_rate=3e-4,
    n_steps=1024,
    batch_size=64,
    gamma=0.99,
    gae_lambda=0.95,
    clip_range=0.2,
    ent_coef=0.0,
    verbose=1,
    tensorboard_log="./logs/tensorboard/",
    seed=42,
)

model.learn(
    total_timesteps=100_000,
    progress_bar=True,
)

model.save("./models/ppo_cartpole")

mean_reward, std_reward = evaluate_policy(
    model,
    model.get_env(),
    n_eval_episodes=10,
    deterministic=True,
)

print(f"mean reward: {mean_reward:.2f} +/- {std_reward:.2f}")

train_env.close()
```

运行：

```bash
mkdir -p models
python train_ppo.py
```

查看 TensorBoard：

```bash
tensorboard --logdir ./logs/tensorboard
```

浏览器通常访问：

```text
http://localhost:6006
```

重点指标：

- `rollout/ep_rew_mean`：平均 episode reward。
- `rollout/ep_len_mean`：平均 episode 长度。
- `train/approx_kl`：新旧策略的差异。
- `train/clip_fraction`：触发 PPO clipping 的样本比例。
- `train/entropy_loss`：策略的随机程度。
- `train/explained_variance`：价值网络解释回报变化的能力。
- `train/value_loss`：价值函数误差。

## 12. Demo 3：播放并录制模型

创建 `play_ppo.py`：

```python
from pathlib import Path

import gymnasium as gym
from stable_baselines3 import PPO

video_dir = Path("./videos")
video_dir.mkdir(exist_ok=True)

env = gym.make(
    "CartPole-v1",
    render_mode="rgb_array",
)

env = gym.wrappers.RecordVideo(
    env,
    video_folder=str(video_dir),
    episode_trigger=lambda episode_id: True,
    name_prefix="ppo-cartpole",
)

model = PPO.load("./models/ppo_cartpole")

obs, info = env.reset(seed=42)
episode_reward = 0.0

while True:
    action, _ = model.predict(obs, deterministic=True)

    obs, reward, terminated, truncated, info = env.step(action)
    episode_reward += reward

    if terminated or truncated:
        print(f"episode reward: {episode_reward}")
        break

env.close()
```

运行：

```bash
python play_ppo.py
```

视频保存在：

```text
videos/
```

至此便形成一个完整实验流程：

```text
创建环境 → 采样 → PPO 训练 → TensorBoard → 保存模型 → 评估 → 录制视频
```

## 13. 手写 PPO 的实现顺序

推荐目录：

```text
ppo_from_scratch/
├── agent.py          # Actor 和 Critic
├── rollout_buffer.py # 保存轨迹
├── train.py          # 采样和更新
├── evaluate.py       # 评估和录制
└── utils.py          # seed、日志等
```

推荐实现顺序：

1. Actor 输出动作分布。
2. Critic 输出状态价值。
3. 收集 `obs/action/reward/value/log_prob/done`。
4. 计算 TD residual。
5. 反向计算 GAE。
6. 计算 return。
7. 标准化 advantage。
8. 执行多轮 minibatch 更新。
9. 实现 clipped surrogate loss。
10. 加入 value loss、entropy bonus 和 gradient clipping。
11. 与 SB3 的 reward 曲线进行对比。

参考 [CleanRL 单文件 PPO](https://github.com/vwxyzjn/cleanrl/blob/master/cleanrl/ppo.py)。

## 14. 怎样寻找合适的 Demo

### 14.1 从官方教程开始

- [Gymnasium Tutorials](https://gymnasium.farama.org/tutorials/)
- [Gymnasium FrozenLake Q-learning](https://gymnasium.farama.org/tutorials/training_agents/frozenlake_q_learning/)
- [SB3 Examples](https://stable-baselines3.readthedocs.io/en/master/guide/examples.html)
- [CleanRL PPO](https://docs.cleanrl.dev/rl-algorithms/ppo/)
- [Isaac Lab Quickstart](https://isaac-sim.github.io/IsaacLab/main/source/setup/quickstart.html)

### 14.2 列出 Gymnasium 环境 ID

```python
import gymnasium as gym

for env_id in sorted(gym.envs.registry.keys()):
    print(env_id)
```

按名称过滤：

```python
import gymnasium as gym

for env_id in sorted(gym.envs.registry.keys()):
    if "CartPole" in env_id or "Pendulum" in env_id:
        print(env_id)
```

### 14.3 使用针对性的搜索词

```text
Gymnasium CartPole PPO example
Stable-Baselines3 Pendulum PPO
Gymnasium custom environment tutorial
CleanRL PPO CartPole
Isaac Lab Cartpole PPO tutorial
```

限定官方站点：

```text
site:gymnasium.farama.org Q-learning tutorial
site:stable-baselines3.readthedocs.io PPO example
site:isaac-sim.github.io/IsaacLab cartpole training
```

### 14.4 判断 Demo 是否适合入门

优先选择以下特征：

- 状态为小型向量。
- 动作空间简单。
- reward 比较密集。
- CPU 几分钟内可以看到提升。
- 成功指标明确。
- 支持渲染或录制。
- 使用 `import gymnasium as gym`。
- `step()` 返回五个值。
- 依赖数量较少。
- 提供 TensorBoard 或 episode reward 日志。

Atari 图像输入、多智能体、大型机器人 locomotion 和稀疏奖励机械臂抓取涉及更多工程问题，可以安排在基础流程熟练以后。

## 15. 最终建议

当前最值得完成的路线：

```text
FrozenLake Q-learning
        ↓
SB3 PPO CartPole
        ↓
手写 PPO CartPole
        ↓
Pendulum 连续控制
        ↓
自定义 Gymnasium 环境
        ↓
Isaac Lab 机器人强化学习
```

每个实验至少保存以下结果：

- 超参数配置。
- 随机种子。
- episode reward 曲线。
- episode length 曲线。
- 训练用时。
- 最终评估回报。
- 一段模型运行视频。

强化学习训练具有较强随机性。调试时应从简单环境开始，持续监控训练指标，并使用 SB3 等成熟实现作为基线。
