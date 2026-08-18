# 昇腾 310P 策略推理卸载（方案 B）

本文记录将 OHDog 策略前向从 ONNX Runtime CPU 卸载到 **昇腾 310P** 的修改过程、目的与验证结果。对应任务见 `task/gpu.md`。

方案：**ATC 离线把 `.onnx` 编成 `.om`，运行时用 AscendCL（`aclmdlExecute`）在 NPU 上执行**。控制环、状态机、观测拼装与动作后处理仍在 CPU。

---

## 1. 硬件与软件环境检查

本机 `npu-smi info` 结果：

| 项 | 值 |
|----|----|
| 芯片 | **310P3**（NPU 8，device 0/1） |
| 驱动 | npu-smi 25.5.1 |
| CANN | `/usr/local/Ascend/cann-8.5.0` |
| ATC | `/usr/local/Ascend/cann-8.5.0/bin/atc` |
| 设备节点 | `/dev/davinci0`、`/dev/davinci1` |

结论：当前设备具备 310P，可以走官方离线 OM + ACL 路径。310P 不是 CUDA GPU，不能把整份 C++ 控制程序放到 NPU 上跑。

额外 Python 依赖安装在 conda 环境 **`ohdog`**（按任务要求，不污染 base）：

```bash
conda create -n ohdog python=3.10 -y
conda activate ohdog
pip install "numpy<2" mujoco sympy decorator attrs protobuf scipy
```

- `mujoco`：C/Python 仿真与 `rl_sim` 链接 `libmujoco.so`
- `sympy` 等：ATC 图编译依赖（缺 `sympy` 时 ATC 报 `EC0010`）

---

## 2. 总体设计

```
CPU：5 ms 状态机 + Eigen 拼 obs + 动作缩放 + MuJoCo/DDS
NPU：仅 PolicyInfer::Run() → H2D → aclmdlExecute → D2H
```

运行时若路径解析到 `.om` 且编译定义了 `USE_ASCEND`，则走 ACL；否则仍走原来的 ORT CPU（便于无 NPU 的交叉编译）。

---

## 3. 修改清单与目的

### 3.1 新增推理后端

| 文件 | 目的 |
|------|------|
| `run_policy/policy_path.hpp` | 按 stem（如 `runpolicy`）查找模型。搜索 `$POLICY_DIR`、`/data/lite3_dev/policy`、仓库 `policy/`、`policy_bk/`。`USE_ASCEND` 时优先 `.om`。 |
| `run_policy/policy_infer.hpp` | 进程级 `aclInit`/`aclrtSetDevice` 单例；每个策略一个 `aclmdlLoadFromFile`。`Run()` 在策略线程里 `BindThread` 后执行。加载与周期性推理打印 `[Ascend]` 日志，证明图跑在 310P 上。无 `USE_ASCEND` 时回退 ORT。 |

### 3.2 策略 Runner：只替换前向，不改观测/动作语义

下列文件把 `Ort::Session` 换成 `PolicyInfer`，`Onnx_infer` / `session.Run` 改为 `infer_.Run()`。观测堆叠、相位、clip、关节映射保持原逻辑。

- `run_policy/lite3_policy_runner.hpp`（行走，输入 470）
- `run_policy/lite3_policy_jump.hpp`（跳跃，470）
- `run_policy/lite3_policy_handstand.hpp`（倒立，45）
- `run_policy/lite3_policy_backflip.hpp`（后空翻，360）
- `run_policy/lite3_policy_midmum.hpp`（中速，282）

未改：`lite3_policy_wave.hpp`（关键帧，无神经网络）、`lite3_policy_backflip_copy.hpp` / `_0521.hpp`（未被状态机引用）。

`run_policy/policy_runner_base.hpp`：去掉未使用的 `onnxruntime_cxx_api.h`，避免仿真目标无故依赖 ORT。

### 3.3 状态机模型路径

`run_control_state.hpp`、`jump_control_state.hpp`、`hs_control_state.hpp`、`mid_control_state.hpp`、`bf_control_state.hpp` 不再写死 `/data/lite3_dev/policy/*.onnx`，改为 `ResolvePolicyModel(...)`，本机可加载仓库 `policy/*.om`，板上仍可放 `/data/lite3_dev/policy/*.om`。

### 3.4 MuJoCo 端到端（无 DDS / 无 ROS2）

本仓库实机主程序依赖 DrDDS sysroot，本机没有完整 `lib_sysroot` 时无法链 `rl_deploy`。为满足「仿真端到端」增加独立目标：

| 文件 | 目的 |
|------|------|
| `interface/robot/simulation/mujoco_interface.hpp` | `RobotInterface` 的 MuJoCo 实现：1 ms 物理 × 5 步对齐 5 ms 控制拍，PD 力矩写入 `ctrl`。 |
| `interface/user_command/scripted_command_interface.hpp` | 脚本指令：0.5 s 起立，4 s 进入 RUN。 |
| `state_machine/quadruped/sim_state_machine.hpp` | 不包含 DDS/Lite3Interface 的状态机接线。 |
| `sim_main.cpp` | 限时跑 idle→standup→run，检查基座高度。 |
| `test/test_ascend_infer.cpp` | 只测 OM 加载与 5 次 ACL 前向。 |

### 3.5 构建与模型转换

| 文件 | 目的 |
|------|------|
| `CMakeLists.txt` | 探测 CANN / MuJoCo；增加 `test_ascend_infer`、`rl_sim`；`USE_ASCEND` 宏；为 `rl_deploy` 在存在 CANN 时同样链 `libascendcl`。 |
| `scripts/compile_onnx_to_om.sh` | ATC 批量转换，`--soc_version=Ascend310P3`，`--precision_mode=must_keep_origin_dtype` 避免默认 FP16 改变 RL 动作。 |

---

## 4. 离线编译（ONNX → OM）

在 **ohdog** 环境中（ATC 需要当前 `python` 能 `import sympy`）：

```bash
conda activate ohdog
export PATH="$CONDA_PREFIX/bin:$PATH"
source /usr/local/Ascend/cann-8.5.0/set_env.sh
bash scripts/compile_onnx_to_om.sh
```

输入形状与代码一致：

| stem | `--input_shape` |
|------|-----------------|
| runpolicy | `obs:1,470` |
| jumppolicy | `obs:1,470` |
| hspolicy | `obs:1,45` |
| mpolicy | `obs:1,282` |
| policy_dwaq | `obs:1,360` |
| bfpolicy | `obs:1,360` |

产物在 `policy/*.om`。本次 ATC 六个模型全部成功。

---

## 5. 编译与运行

```bash
conda activate ohdog
source /usr/local/Ascend/cann-8.5.0/set_env.sh
cmake -S . -B build_sim -DTARGET_PLATFORM=x86 -DCMAKE_BUILD_TYPE=Release
cmake --build build_sim --target test_ascend_infer rl_sim -j

# 冒烟：OM 加载 + 5 次 NPU 前向
./build_sim/test_ascend_infer

# 端到端：MuJoCo 起立后跑行走策略（默认 12 s）
./build_sim/rl_sim 12
```

成功时日志中必须出现类似：

```
[Ascend] ========== NPU runtime initialized ==========
[Ascend] soc_name         : Ascend310P3
[Ascend] ---------- policy offloaded to NPU ----------
[Ascend] backend          : ACL aclmdlExecute
[Ascend] infer#1 ... backend=ACL ... (policy running on Ascend NPU, not CPU)
```

---

## 6. 验证结果（本机）

### 6.1 `test_ascend_infer`

- `soc_name = Ascend310P3`，`device_count = 2`
- `runpolicy.om` `model_id=1`，输入 470×4=1880 B，输出 12×4=48 B
- 5 次前向约 **0.26 ms/次**，正常退出

### 6.2 `rl_sim`（MuJoCo + 五策略加载）

- 五个 OM 均加载到 NPU（model_id 1–5）
- 状态：`idle_state → standup_state → run_control`
- 行走策略在 NPU 上推理 400+ 次，单次约 **0.17–0.52 ms**（远小于 20 ms 控制拍）
- `entered_run=1`，`run_ticks=1601`，`final_height≈0.34 m`（站立高度量级）
- 打印 `[E2E] SUCCESS: policy loaded on Ascend NPU and MuJoCo sim stayed up.`

判定：任务要求的「策略加载 + MuJoCo 仿真端到端」通过，且日志可证明推理在昇腾而非 CPU。

---

## 7. 实机 `rl_deploy` 说明

本机 CMake 在检测到 CANN 时会给 `rl_deploy` 加上 `USE_ASCEND` 并链接 `libascendcl`。交叉编译到无 CANN 的 OHOS 板时，只要找不到 `ASCEND_HOME`，`USE_ASCEND` 为 OFF，策略仍走 ORT CPU。

上板若也有 310P：把 `policy/*.om` 放到 `/data/lite3_dev/policy/`（或设 `POLICY_DIR`），用带 CANN 的工具链编译即可。

---

## 8. 未做 / 注意点

- 未把 DDS 真机接口搬上 NPU（做不到，也不需要）。
- 小 MLP 的 NPU 耗时已被 H2D/D2H 主导；收益是卸载 CPU 与统一 310P 部署路径，不是数量级加速。
- `aclFinalize` 在进程静态析构阶段可能 abort，析构里故意不调用；进程退出由 OS 回收。
- Idle 仍尝试写 `/data/lite_dev/joint_angles.log`，本机无该路径会告警，不影响仿真。
