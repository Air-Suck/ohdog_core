# 昇腾 310B 双板推理支持文档

对应任务：`task/310b_a.md`、`task/310b_b.md`。  
本文合并阶段 A（产物）、阶段 B（端到端）与板 B 调试踩坑。

---

## 1. 架构总览

```
板 A (10.10.10.1)                         板 B (10.10.10.2)
  控制 / MuJoCo / 状态机                     UDP 薄服务器 npu_infer_server
  interface/npu/board_a/  ──────UDP──────>  interface/npu/board_b/
  PolicyInfer(USE_REMOTE_NPU)               ACL + policy/310b/*.om
  （不链 ACL）                               Ascend310B1
```

板 A / 板 B 源码解耦，仅共享 `protocol.hpp`、`policy_catalog.hpp`。  
板 A 本地 CANN 仅用于 ATC 离线转 OM（版本须与板 B 运行时匹配，见 §8）。

| 宏 | 含义 |
|----|------|
| `USE_REMOTE_NPU`（本分支默认） | UDP → 板 B 310B |
| `USE_ASCEND` | 本机 ACL 310P（`-DUSE_REMOTE_NPU=OFF`） |
| 无上述宏 | ORT CPU |

本机 310P 历史路径见 `docs/gpu_surpport.md`。

---

## 2. 阶段 A：产物生成

### 2.1 板 A：310B OM 离线编译

环境（conda `robotpi`）：

```bash
conda activate robotpi
conda config --add channels https://repo.huaweicloud.com/ascend/repos/conda/
conda install -y ascend::cann-toolkit==8.5.0 ascend::cann-310b-ops==8.5.0
pip install sympy psutil decorator attrs protobuf scipy
```

若板 B 运行时是 **CANN 7.0**，不要用板 A 的 8.5 conda ATC 产物上板；应在板 B 用本机 `atc` 重转（见 §8.1）。

```bash
conda activate robotpi
bash scripts/compile_onnx_to_om_310b.sh
```

产物：`policy/310b/*.om`（SOC=`Ascend310B1`）。

| stem | input_shape |
|------|-------------|
| runpolicy | obs:1,470 |
| jumppolicy | obs:1,470 |
| hspolicy | obs:1,45 |
| mpolicy | obs:1,282 |
| policy_dwaq | obs:1,360 |
| bfpolicy | obs:1,360 |

### 2.2 板 A：程序编译

| 依赖 | 用途 |
|------|------|
| `cmake` ≥ 3.10、`g++`（C++17） | 构建 |
| conda `robotpi` | OM 离线编译；`rl_sim` 需另装 `mujoco` |
| `$LIB_SYSROOT` | 仅编 `rl_deploy` |

`test_npu_udp_client` **不链接** CANN/ACL。

```bash
cd /path/to/OHDog
cmake -S . -B build_npu \
  -DTARGET_PLATFORM=x86 \
  -DCMAKE_BUILD_TYPE=Release \
  -DUSE_REMOTE_NPU=ON

cmake --build build_npu --target test_npu_udp_client -j
# 阶段 B：
# cmake --build build_npu --target rl_sim -j
```

| 目标 | 说明 |
|------|------|
| `test_npu_udp_client` | UDP 冒烟（HELLO + 一次 Infer） |
| `rl_sim` | MuJoCo 端到端 |
| `rl_deploy` | 实机部署（可选） |

运行冒烟：

```bash
export NPU_SERVER_IP=10.10.10.2
export NPU_UDP_PORT=9527
./build_npu/test_npu_udp_client 10.10.10.2
```

期望：`Ping OK soc=Ascend310B1` 与 `Infer OK`。

### 2.3 UDP 协议

固定报头 16 字节（`interface/npu/protocol.hpp`）：

| 字段 | 类型 | 说明 |
|------|------|------|
| magic | uint32 | `0x4F484E50` ("OHNP") |
| version | uint16 | 1 |
| msg_type | uint16 | 见下表 |
| seq | uint32 | 请求序号，应答须一致 |
| payload_len | uint32 | payload 字节数 |

| msg_type | 方向 | payload |
|----------|------|---------|
| HELLO (0x0001) | A→B | 空 |
| HELLO_ACK (0x0002) | B→A | soc_name[32] |
| INFER (0x0010) | A→B | policy_id + reserved + obs_dim + obs[] |
| INFER_RESP (0x0011) | B→A | policy_id + status + act_dim + infer_ms + soc_name[32] + action[] |
| PING/PONG | 双向 | 同 HELLO |

默认端口 **9527**。

| 变量 | 默认 | 说明 |
|------|------|------|
| NPU_SERVER_IP | 10.10.10.2 | 板 B 地址 |
| NPU_CLIENT_IP | （空） | 板 A 可选 bind |
| NPU_UDP_PORT | 9527 | 端口 |
| NPU_POLICY_DIR | /opt/ohdog/policy/310b | 板 B OM 目录 |
| NPU_SERVER_BIND_IP | 0.0.0.0 | 板 B 监听 |

策略 ID（`policy_catalog.hpp`）：

| ID | stem | obs_dim | act_dim |
|----|------|---------|---------|
| 0 | runpolicy | 470 | 12 |
| 1 | jumppolicy | 470 | 12 |
| 2 | hspolicy | 45 | 12 |
| 3 | mpolicy | 282 | 12 |
| 4 | policy_dwaq | 360 | 12 |
| 5 | bfpolicy | 360 | 12 |

### 2.4 板 B：openEuler 原生编译与部署

拷贝到板 B：

```
interface/npu/protocol.hpp
interface/npu/policy_catalog.hpp
interface/npu/board_b/
policy/310b/*.om   # 须与板端 CANN 版本匹配
```

编译（香橙派 AI Pro 常见路径）：

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh   # 或 cann / 7.0.0/set_env.sh

cd interface/npu/board_b
mkdir -p build && cd build
# acl.h 常在 aarch64-linux/include，勿写错根路径
cmake .. -DASCEND_HOME=/usr/local/Ascend/ascend-toolkit/latest/aarch64-linux
make -j
```

部署：

```bash
sudo mkdir -p /opt/ohdog/policy/310b /opt/ohdog/bin
sudo cp /path/to/policy/310b/*.om /opt/ohdog/policy/310b/
sudo chown HwHiAiUser:HwHiAiUser /opt/ohdog/policy/310b/*.om
sudo chmod 644 /opt/ohdog/policy/310b/*.om
sudo cp build/npu_infer_server /opt/ohdog/bin/
```

手动运行：

```bash
source /usr/local/Ascend/ascend-toolkit/set_env.sh
export NPU_POLICY_DIR=/opt/ohdog/policy/310b
export NPU_UDP_PORT=9527
/opt/ohdog/bin/npu_infer_server
```

成功标志：

```
[Ascend310B] soc_name      : Ascend310B1
[Ascend310B] all 6 policy OM loaded on NPU
[NpuServer] listening on 0.0.0.0:9527 soc=Ascend310B1
```

systemd 自启：

```bash
sudo cp interface/npu/board_b/npu-infer-server.service /etc/systemd/system/
# 按实际 CANN 路径改 Environment / ExecStartPre
sudo systemctl daemon-reload
sudo systemctl enable --now npu-infer-server
# 查看日志
sudo journalctl -u npu-infer-server -f
# 检查服务是否在运行
sudo systemctl status npu-infer-server --no-pager -l
```

### 2.5 阶段 A 文件清单

| 路径 | 目的 |
|------|------|
| `policy/310b/*.om` | 310B1 模型 |
| `interface/npu/protocol.hpp` | UDP 包格式 |
| `interface/npu/policy_catalog.hpp` | 策略 ID / 维度 |
| `interface/npu/board_a/` | 板 A UDP 客户端 |
| `interface/npu/board_b/` | 板 B ACL 服务 + CMake + systemd |
| `scripts/compile_onnx_to_om_310b.sh` | 310B OM 批量转换 |
| `test/test_npu_udp_client.cpp` | 客户端冒烟 |

---

## 3. 阶段 B：端到端验证

### 3.1 目标与实测结果

| 项 | 结果 |
|----|------|
| 策略位置 | 板 B Ascend **310B1**（UDP 远程 ACL） |
| 控制 / 仿真 | 板 A：`rl_sim`（MuJoCo） |
| 通信 | `10.10.10.1` → `10.10.10.2:9527` |
| e2e | **通过**：`entered_run=1`，`run_ticks=1601`，`final_height≈0.34 m` |

数据流：

```
板 A (rl_sim)                          板 B (npu_infer_server)
  状态机拼 obs                           预加载 *.om
  PolicyInfer(USE_REMOTE_NPU) ─UDP─>   ACL aclmdlExecute
  收 action / soc_name        <─UDP─   Ascend310B1
  PD + MuJoCo step
```

### 3.2 阶段 B 修改清单

| 文件 | 目的 |
|------|------|
| `run_policy/policy_infer.hpp` | `USE_REMOTE_NPU`：`RemoteNpuBridge` + UDP；校验 soc 含 `310B` |
| `run_policy/policy_path.hpp` | 搜索 `policy/310b/`；远程逻辑路径 |
| `sim_main.cpp` | e2e 文案区分 remote 310B |
| `CMakeLists.txt` | 默认 `USE_REMOTE_NPU=ON`；`rl_sim` 不链 ACL |
| conda `robotpi` | `pip install "numpy<2" mujoco` |

### 3.3 端到端验证复现指令

板 B 已运行 `npu_infer_server` 时，在板 A 执行：

```bash
conda activate robotpi
cd /path/to/OHDog
cmake -S . -B build_npu -DTARGET_PLATFORM=x86 -DCMAKE_BUILD_TYPE=Release -DUSE_REMOTE_NPU=ON
cmake --build build_npu --target rl_sim -j
export NPU_SERVER_IP=10.10.10.2 NPU_UDP_PORT=9527
./build_npu/rl_sim 12
```

### 3.4 如何证明用了 310B

须同时满足（证据来自板 B ACL，非板 A 伪造）：

1. `[Remote310B] soc_name : Ascend310B1`（HELLO）  
2. 各策略加载：`probe_infer_ms` + `INFER_RESP.soc_name from board-B ACL`  
3. 运行中：`infer#N ... soc=Ascend310B1 backend=UDP+ACL`  
4. e2e SUCCESS 文案含 `Ascend310B (board B via UDP)`  

soc 不含 `310B` 时 `RemoteNpuBridge` 会抛错，不会进入 SUCCESS。

### 3.5 实测摘要

| 项 | 值 |
|----|-----|
| 板 B soc | Ascend310B1 |
| 加载策略 | run / hs / backflip / mid / jump |
| 状态 | idle → standup → run_control |
| run 推理 | 350+ 次，约 0.14–0.30 ms/次 |
| 最终高度 | ≈ 0.34 m |

注意：控制环 5 ms；UDP 超时 50 ms、最多重试 2 次。Idle 写 `/data/lite_dev/joint_angles.log` 失败可忽略。

---

## 4. 板 B 调试踩坑速查

### 4.1 OM 加载失败（CANN 版本不匹配）

**现象：** `aclmdlLoadFromFile failed`，文件在且 `soc_name=Ascend310B1`。  
**原因：** OM 用 CANN **8.5** 转，板端运行时是 **7.0**。  
**处理：** 在板 B 用本机 `atc` 重转，不要用板 A 8.5 conda 产物。

```bash
POLICY_ONNX_DIR=$PWD/policy_bk POLICY_OM_DIR=$PWD/policy_310b \
SOC_VERSION=Ascend310B1 bash compile_onnx_to_om.sh
```

编 server 时：

```text
-DASCEND_HOME=/usr/local/Ascend/ascend-toolkit/latest/aarch64-linux
```

（`acl.h` 在 `.../aarch64-linux/include/`，不是 `/usr/local/Ascend/cann`。）

### 4.2 编译错误

| 错误 | 处理 |
|------|------|
| `policy_catalog.hpp` not found | `#include "../policy_catalog.hpp"` |
| `sockaddr_in` does not name a type | `udp_infer_server.hpp` 加 `#include <netinet/in.h>` |
| `acl/acl.h` not found | 修正 `ASCEND_HOME` 见上 |

### 4.3 ATC 缺 Python / AuthenticationError

缺 `numpy`：

```bash
conda deactivate
python3 -m pip install --user numpy decorator sympy protobuf attrs scipy cloudpickle "synr==0.5.0"
```

`digest sent was rejected`（conda Python + 系统 TBE 混用）：

```bash
conda deactivate
export TE_PARALLEL_COMPILER=1
export MAX_COMPILE_CORE_NUMBER=1
```

再用系统 `python3` + 板端 `atc`。**不必**为板端转 OM 再装 cann-toolkit 8.5。

### 4.4 无外网：路由被板间网/USB 抢走

**现象：** DNS 失败 / `Destination Host Unreachable`；WiFi 本可上网。  
**原因：** `eth1` 写了 `gateway=10.10.10.1`，或 `usb0` 默认路由 metric 更小。

期望：

```text
default via 192.168.12.1 dev wlan0 ...
10.10.10.0/24 dev eth1 ...    # 不要有 default via 10.10.10.1
```

板间网只设地址、**不设默认网关**：

```bash
sudo nmcli connection modify "static-eth1" ipv4.gateway "" ipv4.dns "" ipv4.never-default yes
sudo nmcli connection modify "usb0-static" ipv4.never-default yes ipv4.gateway ""
sudo nmcli connection down "usb0-static"
sudo nmcli connection modify "IPADS-2F" ipv4.route-metric 100 ipv4.never-default no
sudo nmcli connection up "IPADS-2F"
```

### 4.5 有网但 pip SSL 失败

**现象：** `certificate is not yet valid` + 系统时间异常。  
**处理：** 修好路由后开 NTP；确认 `date` 为当前年份再装包。

### 4.6 新 OM 已转好仍 load 失败

**原因：** `/opt/ohdog/policy/310b` 下文件常为 **root + 600**。  

```bash
sudo chown HwHiAiUser:HwHiAiUser /opt/ohdog/policy/310b/*.om
sudo chmod 644 /opt/ohdog/policy/310b/*.om
# 或：export NPU_POLICY_DIR=$PWD/policy_310b
```

### 4.7 路径速记

| 用途 | 路径 |
|------|------|
| ONNX 源 | `policy_bk/*.onnx` |
| 板 A ATC（8.5）产物 | `policy/310b/*.om` |
| 板 B 本机 ATC 产物 | `policy_310b/*.om`（示例） |
| 部署目录 | `/opt/ohdog/policy/310b/` |
| 可执行文件 | `/opt/ohdog/bin/npu_infer_server` |
| ACL include/lib | `.../ascend-toolkit/latest/aarch64-linux/` |
