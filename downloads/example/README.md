# 机器人控制 Demo

本目录提供一组通过 ROS 2 话题控制 H1 机器人的 Python 示例脚本，每个 demo
一个文件，位于 `demos/` 目录。

Demo 的定位是**帮你写好话题发布/订阅链路**，让使用者快速体验 SDK 的控制接口：

- **默认演示**：不带参数直接运行，播放一段编排好的动作；
- **动作触发**：`--shake`/`--nod` 等参数单独触发某个编排动作；
- **原生参数**：`--pitch`/`--yaw`、`--joints`、`--up/--down` 等直接下发话题
  原生的控制量（角度/关节/速度），demo 只负责度→弧度换算与发布。

## 目录结构

```
example/
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
| 左/右臂 MoveP/MoveL | `/left_arm/movep` `/right_arm/movep` `/left_arm/movel` `/right_arm/movel` | `std_msgs/String`(JSON) | `{"pose": {"position": {"x","y","z"}, "orientation": {"x","y","z","w"}}, "speed_scale": 1.0}`；位姿在**臂模型根系**（URDF 根 link，非躯干 base_link），零位末端约 `(-0.7265, 0, 0.158)` |
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

- MoveJ 是关节空间运动，只要各关节在限位内即可执行；超限会返回
  `joint_limit_violation`（关节限位见 `armcontrol/config/arm_control_node.yaml`）。
- MoveP/MoveL 下发的是末端位姿，节点内部需要 IK 逆解成关节角；
  位姿不可达时返回 `ik_failed`。位姿在臂模型根系下表示。
- MoveL 额外要求**整条直线路径逐点 IK 有解**，比 MoveP 苛刻：臂完全伸直
  （零位）处于奇异位形，从零位直接做长直线会 `ik_failed`，需要先 MoveJ
  到收拢姿态再做直线（movel_demo 即按此编排）。
- 同一末端位姿的 IK 可能有多解，MoveP/MoveL「回到」某位姿时关节构型
  不一定与初始一致；需要精确复原姿势时用 MoveJ 回关节角（demo 均按此处理）。

## 运行方式

直接用 python 运行（需先 source ROS 环境）：

### 1) 双臂预置动作（arm_action_demo）

```bash
# 从 SDK 仓库根目录进入示例目录
cd example/demos

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
- 手臂 demo 失败或 Ctrl+C 时会自动发布 `/<side>_arm/stop` 停止当前运动；
  也可手动发布 `/action/stop`（Empty）停止预置动作、
  `/left_arm/stop` / `/right_arm/stop` 停止单臂运动。
