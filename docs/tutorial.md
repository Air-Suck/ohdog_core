# Sim-to-sim（昇腾 310P）

控制环在 CPU，策略前向在 310P。实现细节见 [gpu_surpport.md](./gpu_surpport.md)。

## 1. 环境

```bash
conda activate ohdog
source /usr/local/Ascend/cann-8.5.0/set_env.sh
npu-smi info    # 应看到 310P3
```

首次需要：

```bash
conda create -n ohdog python=3.10 -y
conda activate ohdog
pip install "numpy<2" mujoco sympy decorator attrs protobuf scipy
```

## 2. ONNX → OM（已有 `policy/*.om` 可跳过）

```bash
conda activate ohdog
export PATH="$CONDA_PREFIX/bin:$PATH"
source /usr/local/Ascend/cann-8.5.0/set_env.sh
bash scripts/compile_onnx_to_om.sh
```

看到 `ATC run success` 且 `policy/*.om` 存在即可。

## 3. 编译

```bash
conda activate ohdog
source /usr/local/Ascend/cann-8.5.0/set_env.sh
cmake -S . -B build_sim -DTARGET_PLATFORM=x86 -DCMAKE_BUILD_TYPE=Release
cmake --build build_sim --target test_ascend_infer rl_sim -j
```

## 4. 启动

仓库根目录：

```bash
# 可选：只测 NPU 前向
./build_sim/test_ascend_infer

# sim-to-sim（脚本自动 idle → standup → run，默认 12 秒）
./build_sim/rl_sim 12
```

## 5. 怎么看出 policy 在 310P 上

盯日志里的 **`[Ascend]`**，下面三处同时出现才算卸载成功：

**① 芯片**

```
[Ascend] soc_name         : Ascend310P3
[Ascend] expected_chip    : Ascend310P3
```

**② 加载的是 OM + ACL，不是 ONNX/CPU**

```
[PolicyPath] runpolicy -> ".../policy/runpolicy.om"
[Ascend] ---------- policy offloaded to NPU ----------
[Ascend] backend          : ACL aclmdlExecute
```

`rl_sim` 启动时会连续加载多个 `.om`（run / hs / backflip / mid / jump），每条都有 `model_id`。

**③ 推理在 NPU 上反复执行**

进入 `run_control` 后每隔约 50 次打一行：

```
[Ascend] infer#1 device=0 soc=Ascend310P3 backend=ACL model_id=1 cost=0.xx ms  (policy running on Ascend NPU, not CPU)
```

`backend=ACL`、`soc=Ascend310P3`、括号里的英文是判定依据。`cost` 一般远小于 20 ms。

**④ 仿真本身成功**

```
idle_state ------------> standup_state
standup_state ------------> run_control
[E2E] entered_run=1 ... final_height=0.33...
[E2E] SUCCESS: policy loaded on Ascend NPU and MuJoCo sim stayed up.
```

若出现 `[ORT-CPU] loaded ...onnx`，说明走了 CPU，检查是否编了 `USE_ASCEND`、以及 `policy/*.om` 是否存在。
