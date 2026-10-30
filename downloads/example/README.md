# 标准版 Python 控制示例

本目录提供一组通过 ROS 2 话题控制 H1 机器人的 Python 示例脚本，每个 demo
一个文件，位于 `demos/` 目录。

Demo 的定位是**帮你写好话题发布/订阅链路**，让使用者快速体验 SDK 的控制接口：

- **默认演示**：不带参数直接运行，播放一段编排好的动作；
- **动作触发**：`--shake`/`--nod` 等参数单独触发某个编排动作；
- **原生参数**：`--pitch`/`--yaw`、`--joints`、`--up/--down` 等直接下发话题
  原生的控制量（角度/关节/速度），demo 只负责度→弧度换算与发布。

## 获取示例

本组示例归类为**标准版**，推荐升级到最新配套固件，并按下文配置原生 Zenoh。标准版与智能版使用相同通信方式，但这些动作的智能版适用性仍需核对；智能版的硬件与登录入口见[智能版连接与使用](https://onerobot-sdk.github.io/onerobot-H1/getting-started/intelligent.html)。

通过 Git 获取仓库后进入本目录：

```bash
git clone https://github.com/onerobot-sdk/h1-examples.git
cd h1-examples/standard/python-control
```

也可以在 GitHub 仓库选择 **Code → Download ZIP**，解压后进入 `standard/python-control/`。开发手册保留的兼容下载包解压后为 `example/`，进入该目录即可；这两种方式均在本 README 所在目录执行后续 `cd demos`。

## 电脑端 ROS 2 通信配置

H1 使用 ROS 2 Jazzy。在调试电脑上运行示例前，先连接机器人所在网络，查询[设备实际固件版本](https://onerobot-sdk.github.io/onerobot-H1/firmware.html)；如为旧固件，建议按[OTA 说明](https://onerobot-sdk.github.io/onerobot-H1/faq/network-ota.html#q17)升级到最新配套版本。运行 Python 示例、`ros2` 命令或 RViz2 的终端需要加载与设备匹配的通信环境。

### 配置原生 Zenoh

标准版和智能版均使用 **`rmw_zenoh_cpp`**。先按[开发电脑的 Zenoh 配置](https://onerobot-sdk.github.io/onerobot-H1/getting-started/communication.html#pc-zenoh)安装支持包、配置电脑 router，并创建 `~/h1-zenoh-env.sh`。只设置 RMW 环境变量还不够；机器人本机继续使用固件提供的通信配置。

完成上述配置后，先退出电脑上使用旧环境的示例或 RViz2，在新的 Bash 终端执行：

```bash
source ~/h1-zenoh-env.sh
ros2 pkg prefix rmw_zenoh_cpp
echo "$RMW_IMPLEMENTATION"
ros2 daemon stop
ros2 topic list
```

`echo` 应输出 `rmw_zenoh_cpp`。确认能看到机器人话题，再按[通信验证](https://onerobot-sdk.github.io/onerobot-H1/getting-started/communication.html#verify)订阅 `/device_info`，确认持续收到设备数据后运行示例。每个运行示例或 RViz2 的终端都需要加载同一环境；IDE、容器和其他 Shell 也需配置对应环境。

若看不到话题，按通信配置页依次核对机器人 IP、电脑 router、RMW 环境及设备服务。`ros2 daemon stop` 只停止电脑端 ROS 2 CLI 缓存进程，不会重启机器人进程。

<a id="电脑端-ros-2-与-dds-配置"></a>

<details>
<summary>兼容说明：尚未升级的标准版（Cyclone DDS）</summary>

建议优先升级到[最新配套固件](https://onerobot-sdk.github.io/onerobot-H1/firmware.html)，然后使用上面的 Zenoh 配置；升级步骤见[网络配置与 OTA](https://onerobot-sdk.github.io/onerobot-H1/faq/network-ota.html#q17)。暂时无法升级时，再使用本节兼容配置。

以下步骤仅适用于固件早于 **1.0.104.261028**、使用 **Cyclone DDS（`rmw_cyclonedds_cpp`）** 的标准版。智能版不使用本节配置。兼容性说明见[旧版标准版 DDS 配置](https://onerobot-sdk.github.io/onerobot-H1/getting-started/communication.html#legacy-dds)。

#### 首次安装与当前终端配置

以下命令在已安装 ROS 2 Jazzy 的 Ubuntu 电脑上执行：

```bash
# 首次使用时安装 Cyclone DDS 的 ROS 2 支持包
sudo apt install ros-jazzy-rmw-cyclonedds-cpp

source /opt/ros/jazzy/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp

# 确认包已安装、当前终端选择正确
ros2 pkg prefix rmw_cyclonedds_cpp
echo "$RMW_IMPLEMENTATION"
```

`echo` 应输出 `rmw_cyclonedds_cpp`。旧版标准版的默认使用方式**不需要单独设置 `ROS_DOMAIN_ID`**。
机器人内部的 `CYCLONEDDS_URI` 和网卡配置由固件管理，不要把板端配置路径或网卡名直接复制到电脑。

#### Bash 自动生效

如希望新终端自动使用该配置，在电脑的 `~/.bashrc` 中添加下面两行；已有相同配置时不用重复添加。
将 RMW 设置放在 ROS 环境加载语句之后，并避免后续配置覆盖它：

```bash
source /opt/ros/jazzy/setup.bash
export RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
```

保存后执行 `source ~/.bashrc`，或重新打开 Bash 终端。IDE、容器和其他 Shell 中启动的程序也需要加载对应环境。

#### 切换 DDS 后检查通信

如果此前使用过其他 RMW 实现，先退出电脑上旧的示例或 RViz2，重新加载环境，再重启电脑上的 ROS 2 CLI daemon：

```bash
ros2 daemon stop
ros2 daemon start
ros2 topic list
```

确认能看到机器人话题（例如 `/device_info`），再运行下文示例。
若看不到，依次核对电脑与机器人的网络连通性、当前 RMW 配置、机器人对应服务是否运行，以及防火墙/多网卡对 DDS 发现的影响。
daemon 重启只影响电脑端命令行发现服务，不会重启机器人进程。

参考：[ROS 2 官方 RMW 配置说明](https://docs.ros.org/en/humble/How-To-Guides/Working-with-multiple-RMW-implementations.html)。

</details>

## 目录结构

```
python-control/
├── README.md                # 本文档
└── demos/                   # 每个 demo 一个 Python 文件
    ├── arm_action_demo.py   # 双臂预置动作：左臂挥手 -> 右手招手
    ├── movej_demo.py        # MoveJ 关节空间：抬起 -> 回零位
    ├── movep_demo.py        # MoveP 笛卡尔点到点：举起 -> 放下
    ├── movel_demo.py        # MoveL 笛卡尔直线：收胸 -> 长直线举起 -> 放下
    ├── base_move_demo.py    # 底盘：前进/后退/左转/右转
    ├── lift_control_demo.py # 升降：抬臂 -> 下降 -> 上升复位 -> 恢复手臂
    ├── head_control_demo.py # 头部：摇头 + 点头 / 原生角度控制
    └── light_control_demo.py # 灯光：灯板颜色/亮度/闪烁
```

## 控制话题速查

| 功能 | 话题 | 消息类型 | 说明 |
| --- | --- | --- | --- |
| 双臂预置动作 | `/action/execute` | `std_msgs/String`(JSON) | `{"action_id": "...", "speed_scale": 1.0}` |
| 动作状态反馈 | `/action/status` | `std_msgs/String`(JSON) | `{"action_id","state","progress","fail_reason"}` |
| 动作停止 | `/action/stop` | `std_msgs/Empty` | 立即停止当前动作 |
| 左/右臂 MoveJ | `/left_arm/movej` `/right_arm/movej` | `std_msgs/String`(JSON) | `{"joints": [7个弧度], "speed_scale": 1.0}` |
| 左/右臂 MoveP/MoveL | `/left_arm/movep` `/right_arm/movep` `/left_arm/movel` `/right_arm/movel` | `std_msgs/String`(JSON) | `{"pose": {"position": {"x","y","z"}, "orientation": {"x","y","z","w"}}, "speed_scale": 1.0}`；位姿在对应的**臂模型根系**（URDF 根 link，非躯干 `base_link`） |
| 臂运动结果反馈 | `/arm_diagnostics` | `diagnostic_msgs/DiagnosticStatus` | `name`=left_arm/right_arm，`values` 含 `motion_type`，`message`="motion completed" 表示成功 |
| 臂运动停止 | `/left_arm/stop` `/right_arm/stop` | `std_msgs/Empty` | 立即停止对应手臂的 MoveJ/MoveP/MoveL |
| 当前关节角 | `/left_joint_states` `/right_joint_states` | `sensor_msgs/JointState` | `joint1-<l/r>` ~ `joint7-<l/r>`，单位弧度 |
| 末端位姿(latched) | `/acc/left/ee_settled` `/acc/right/ee_settled` 等 | `geometry_msgs/PoseStamped` | movep/movel 到位后的实测末端位姿；`target_ee` 为目标位姿 |
| 底盘 | `/cmd_vel` | `geometry_msgs/Twist` | `linear.x` 前进、`angular.z` 转向 |
| 升降(绝对) | `/lift/joint_states/update` | `sensor_msgs/JointState` | 关节名 `lift_joint`，单位米 |
| 升降(相对) | `/lift/joint_states/update_relative` | `sensor_msgs/JointState` | 正=上升，负=下降 |
| 头部(位置) | `/head/joint_states/update` | `sensor_msgs/JointState` | `head_pitch_joint` / `head_yaw_joint`，单位弧度 |
| 头部(速度) | `/head/cmd_vel` | `geometry_msgs/Twist` | `angular.y` 俯仰、`angular.z` 偏航 |
| 灯板 | `/light_board/control` | `std_msgs/String`(JSON) | `{"boards":[{"color":"#RRGGBB","brightness":0-100,"blink_hz":0-100},...]}`，`null` 跳过对应灯板 |

### 双臂预置动作列表

对应 `armcontrol/config/arm_actions.yaml`：

`wave`（挥手）、`home`（回零位）、`salute`（敬礼）、`open_arms`（张开双臂）、
`guide`（指引）、`point`（指向）、`handshake`（握手）、`handover`（递物）、
`beckon`（招手）、`ready`（准备姿态）。

### 臂关节约定

双臂各 7 个关节（`joint1` ~ `joint7`），MoveJ 的 `joints` 数组按此顺序填写，
单位为弧度。零位 `[0, 0, 0, 0, 0, 0, 0]`。

### MoveJ / MoveP / MoveL 使用注意

MoveP / MoveL 的目标位姿以对应机械臂的模型根坐标系为参考，左右臂模型根坐标系的轴向不同：

| 机械臂 | +X | +Y | +Z |
| --- | --- | --- | --- |
| 左臂 | 向上 | 向前 | 向左 |
| 右臂 | 向上 | 向后 | 向右 |

整机 `base_link` 坐标系采用 X 轴向前、Y 轴向左、Z 轴向上的约定。如果只换算位移方向，不考虑坐标系原点之间的平移，则左臂满足 `(Δx_base, Δy_base, Δz_base) = (Δy_arm, Δz_arm, Δx_arm)`，右臂满足 `(Δx_base, Δy_base, Δz_base) = (-Δy_arm, -Δz_arm, Δx_arm)`。换算绝对位置时，还需要计入对应臂根相对 `base_link` 的平移。

末端工具坐标系（TCP frame）固定在机械臂末端。各关节角均为 0 时，左臂 TCP 坐标系的 +X 轴向前、+Y 轴向右、+Z 轴向下；右臂 TCP 坐标系的 +X 轴向后、+Y 轴向左、+Z 轴向下。机械臂运动时，TCP 坐标系会随末端姿态一起旋转。目标位置和姿态四元数均以对应的臂模型根坐标系为参考。

- MoveJ 是关节空间运动，只要各关节在限位内即可执行；超限会返回
  `joint_limit_violation`（关节限位见 `armcontrol/config/arm_control_node.yaml`）。
- MoveP / MoveL 下发的是末端位姿，节点内部需要通过 IK 将其转换为关节角；
  位姿不可达时返回 `ik_failed`。
- MoveL 额外要求**整条直线路径逐点 IK 有解**，比 MoveP 苛刻：臂完全伸直
  （零位）处于奇异位形，从零位直接做长直线会 `ik_failed`，需要先 MoveJ
  到收拢姿态再做直线（movel_demo 即按此编排）。
- 同一末端位姿的 IK 可能有多解，MoveP/MoveL「回到」某位姿时关节构型
  不一定与初始一致；需要精确复原姿势时用 MoveJ 回关节角（demo 均按此处理）。

## 运行方式

直接用 python 运行（需先 source ROS 环境）：

### 1) 双臂预置动作（arm_action_demo）

```bash
# 先进入本 README 所在目录，再进入 demos/
cd demos

python3 arm_action_demo.py                      # 默认：左臂挥手 -> 右手招手
python3 arm_action_demo.py wave                 # 只左臂挥手
python3 arm_action_demo.py salute open_arms     # 敬礼 -> 张开双臂
python3 arm_action_demo.py --speed-scale 0.8    # 速度 0.8 倍
```

### 2) MoveJ 关节空间（movej_demo）

```bash
python3 movej_demo.py                           # 左臂肩部抬起 -> 回零位
python3 movej_demo.py --arm right               # 右臂对称演示
python3 movej_demo.py --joints 0 0.5 0 -1.2 0 0 0   # 自定义 7 关节目标（弧度，注意限位）
```

### 3) MoveP 笛卡尔点到点（movep_demo）

```bash
python3 movep_demo.py                           # 归零 -> 举到演示位姿 -> MoveJ 复原
python3 movep_demo.py --arm right               # 右臂
python3 movep_demo.py --speed-scale 0.5         # 降低速度
```

### 4) MoveL 笛卡尔直线（movel_demo）

```bash
python3 movel_demo.py                           # 归零 -> 收到胸口 -> 0.46m 长直线举到高位 -> MoveJ 放下
python3 movel_demo.py --arm right               # 右臂
python3 movel_demo.py --speed-scale 0.5         # 降低速度
```

### 5) 底盘（base_move_demo）

转向用**时间**控制（底盘起步加速/停止滞后会导致按角度折算转不到位），
转多少度由实际角速度与时长决定；默认演示左右转向时长对称，误差相互抵消。

```bash
python3 base_move_demo.py                       # 默认演示：前进0.5m -> 后退0.5m -> 左转3s -> 等2s -> 右转3s 复原
python3 base_move_demo.py --forward 0.3         # 前进 0.3m
python3 base_move_demo.py --back 0.2            # 后退 0.2m
python3 base_move_demo.py --left 2.0            # 以 --angular 角速度左转 2 秒
python3 base_move_demo.py --right 3.0 --angular 0.5   # 右转 3 秒（角速度 0.5 rad/s）
python3 base_move_demo.py --forward 0.3 --right 2.0   # 组合：先前进再右转
python3 base_move_demo.py --step-wait 3         # 步间等待改为 3s
```

### 6) 升降（lift_control_demo）

```bash
python3 lift_control_demo.py                    # 默认演示：抬臂 -> 下降0.5m -> 上升复位 -> 恢复手臂
python3 lift_control_demo.py --delta 0.10       # 下降/上升位移改为 0.10m
python3 lift_control_demo.py --down 0.1         # 手动：仅下降 0.1m（不联动手臂）
python3 lift_control_demo.py --up 0.1           # 手动：仅上升 0.1m
```

### 7) 头部（head_control_demo）

```bash
python3 head_control_demo.py                    # 默认演示：摇头来回3次 -> 点头来回3次
python3 head_control_demo.py --shake 2          # 摇头来回 2 次
python3 head_control_demo.py --nod 1            # 点头来回 1 次
python3 head_control_demo.py --pitch 20 --yaw -15   # 原生角度控制：俯仰20°、偏航-15°（度，可单独给）
python3 head_control_demo.py --wait 1.5         # 每步等待时间（头部电机较慢，建议不小于 1.5s）
```

### 8) 灯光（light_control_demo）

```bash
python3 light_control_demo.py                   # 自动播放灯光演示
python3 light_control_demo.py --on              # 恢复白色 100% 常亮
python3 light_control_demo.py --color "#FF0000" --brightness 100   # 四板红色 100% 常亮
python3 light_control_demo.py --color "#0000FF" --blink-hz 10      # 蓝色 10Hz 闪烁
python3 light_control_demo.py --board 1 --color "#00FF00"          # 仅灯板 1 绿色（其余 null 跳过）
python3 light_control_demo.py --off             # 关闭所有灯
```

每个脚本均支持 `--help` 查看完整参数。

## 安全提示

- 手臂 / 底盘 / 升降命令会真实驱动机器人，首次运行建议在仿真或低倍率
  （`--speed-scale` 取较小值、`--linear`/`--angular` 取较小值）下验证。
- 底盘轮速在 `body_control_node` 中默认限速 0.35 m/s，超过会自动缩放。
- 升降/头部命令受硬件限位保护，但仍有必要先在安全区域测试。
- MoveJ / MoveP / MoveL demo 失败或 Ctrl+C 时会自动发布 `/<side>_arm/stop` 停止当前运动；
  也可手动发布 `/action/stop`（Empty）停止预置动作、
  `/left_arm/stop` / `/right_arm/stop` 停止单臂运动。
