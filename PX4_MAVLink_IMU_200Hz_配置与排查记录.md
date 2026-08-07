# PX4 MAVLink IMU 200 Hz 配置与排查记录

## 1. 目标

将 PX4 飞控通过 MAVLink 输出到机载电脑的 IMU 数据频率提高到约 **200 Hz**，并使配置在飞控重启后自动生效。

最终使用的话题为：

```bash
/mavros/imu/data_raw
```

最终实测结果：

- `vehicle_imu`：约 **293~294 Hz**
- `HIGHRES_IMU` 配置目标频率：**250 Hz**
- 机载电脑 `/mavros/imu/data_raw` 实测：约 **200 Hz**

---

## 2. 初始现象

最开始即使在软件端将 MAVLink `HIGHRES_IMU` 设置为 200 Hz，实际接收频率仍然只能达到约：

```text
173 Hz
```

因此最初怀疑：

- SD 卡是否存在频率限制
- MAVLink 带宽是否不足
- 串口波特率是否不足
- PX4 默认固件是否限制 HIGHRES_IMU

---

## 3. 检查 PX4 内部 `vehicle_imu` 发布频率

在 QGroundControl 的 MAVLink Console 中执行：

```bash
uorb top vehicle_imu
```

初始结果：

```text
vehicle_imu 0    172 Hz
vehicle_imu 1    171 Hz
```

这说明问题并不在 MAVLink，而是在 MAVLink 上游。

`HIGHRES_IMU` 的数据来源是 PX4 内部的 `vehicle_imu`。如果 `vehicle_imu` 本身只有约 172 Hz，那么即使执行：

```bash
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 200
```

也不可能稳定获得 200 Hz 的新 IMU 数据。

因此原来的数据链路实际为：

```text
原始 IMU
   ↓
vehicle_imu ≈ 172 Hz
   ↓
HIGHRES_IMU 请求 200 Hz
   ↓
实际只能 ≈ 173 Hz
```

---

## 4. 检查 `IMU_INTEG_RATE`

执行：

```bash
param show IMU_INTEG_RATE
```

初始参数为：

```text
IMU_INTEG_RATE = 200
```

随后将其修改为：

```bash
param set IMU_INTEG_RATE 400
param save
reboot
```

重启后确认：

```bash
param show IMU_INTEG_RATE
```

结果：

```text
x + IMU_INTEG_RATE : 400
```

其中 `+` 表示参数已经保存。

再次执行：

```bash
uorb top vehicle_imu
```

得到：

```text
vehicle_imu 0    294 Hz
vehicle_imu 1    293 Hz
```

之前约 172 Hz 的内部瓶颈已经被突破。

新的数据链路变成：

```text
原始 IMU
   ↓
IMU_INTEG_RATE = 400
   ↓
vehicle_imu ≈ 293~294 Hz
   ↓
可以支撑 200 Hz 以上 HIGHRES_IMU 输出
```

需要注意：

`IMU_INTEG_RATE=400` 并不代表 `vehicle_imu` 一定严格等于 400 Hz。PX4 会根据真实 IMU 采样周期和积分采样数生成 `vehicle_imu`，因此本机最终约为 294 Hz。

---

## 5. 检查 MAVLink 链路

执行：

```bash
mavlink status
```

飞控存在两个 MAVLink 实例。

### instance #0：机载电脑

```text
/dev/ttyS1 @ 921600
mode: Onboard
```

主要状态：

```text
tx rate mult: 1.000
txerr: 0.0 B/s
tx rate max: 100000 B/s
```

说明：

- 没有因为带宽不足自动降频
- 没有发送错误
- 921600 串口带宽足够

### instance #1：QGC USB

```text
/dev/ttyACM0 @ 2000000
mode: Config
```

同样：

```text
tx rate mult: 1.000
txerr: 0.0 B/s
```

因此可以排除 MAVLink 带宽不足的问题。

---

## 6. 检查默认 HIGHRES_IMU 频率

执行：

```bash
mavlink status streams
```

重启后两个实例默认均为：

```text
HIGHRES_IMU  50.00 (50.000)
```

这说明手动执行 `mavlink stream` 修改的是运行时配置，重启后会恢复默认值。

---

## 7. 将 HIGHRES_IMU 设置为 200 Hz

测试命令：

```bash
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 200
mavlink stream -d /dev/ttyACM0 -s HIGHRES_IMU -r 200
```

检查：

```bash
mavlink status streams
```

得到：

```text
HIGHRES_IMU  200.00 (200.000)
```

同时：

```bash
mavlink status
```

仍然显示：

```text
tx rate mult: 1.000
txerr: 0
```

说明 200 Hz 的目标频率设置成功，而且没有出现带宽限制。

但是在 QGC MAVLink Inspector 中，`HIGHRES_IMU` 实际接收频率约为：

```text
188~192 Hz
```

因此 `-r 200` 只是目标调度频率，并不意味着接收端一定严格获得 200 Hz。

---

## 8. 将 HIGHRES_IMU 目标提高到 250 Hz

继续测试：

```bash
mavlink stream -d /dev/ttyACM0 -s HIGHRES_IMU -r 250
```

QGC MAVLink Inspector 实测：

```text
201~210 Hz
```

说明通过适当提高 HIGHRES_IMU 的目标调度频率，可以使接收端达到约 200 Hz。

因此最终机载电脑链路设置为：

```bash
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
```

检查：

```bash
mavlink status streams
```

结果：

```text
instance #0
HIGHRES_IMU  250.00 (250.000)
```

执行：

```bash
mavlink status
```

结果仍为：

```text
tx rate mult: 1.000
txerr: 0.0 B/s
```

机载电脑最终通过 MAVROS 实测：

```bash
rostopic hz /mavros/imu/data_raw
```

实际频率约为：

```text
200 Hz
```

至此目标完成。

---

## 9. 最终推荐配置

### PX4 参数

```text
IMU_INTEG_RATE = 400
```

该参数已保存，因此飞控重启后仍然生效。

### 机载电脑 MAVLink 端口

```text
/dev/ttyS1 @ 921600
```

### HIGHRES_IMU 目标频率

```bash
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
```

### 最终实测

```text
vehicle_imu              ≈ 293~294 Hz
HIGHRES_IMU target       = 250 Hz
/mavros/imu/data_raw     ≈ 200 Hz
```

---

## 10. 使用 SD 卡自动执行 HIGHRES_IMU 配置

因为 `mavlink stream` 是运行时设置，飞控重启以后会恢复默认 50 Hz，因此使用 SD 卡启动脚本自动执行。

飞控 SD 卡挂载路径：

```text
/fs/microsd
```

首先检查：

```bash
ls /fs/microsd
```

本机正常显示：

```text
APM/
dataman
log/
parameters_backup.bson
```

说明 SD 卡正常挂载。

创建目录：

```bash
mkdir /fs/microsd/etc
```

创建启动脚本：

```bash
echo "set +e" > /fs/microsd/etc/extras.txt
echo "mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250" >> /fs/microsd/etc/extras.txt
echo "set -e" >> /fs/microsd/etc/extras.txt
```

检查文件：

```bash
cat /fs/microsd/etc/extras.txt
```

内容应为：

```text
set +e
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
set -e
```

本机执行：

```bash
sync
```

时提示：

```text
sync: command not found
```

这是因为当前 NuttX 固件未提供 `sync` 命令，不影响已经通过 `echo` 写入并关闭的文件。

随后重启：

```bash
reboot
```

重启后不要手动执行 `mavlink stream`，直接检查：

```bash
mavlink status streams
```

如果看到：

```text
instance #0
HIGHRES_IMU  250.00 (250.000)
```

说明 `/fs/microsd/etc/extras.txt` 已经自动执行成功。

机载电脑随后再次实测 `/mavros/imu/data_raw`，频率约为：

```text
200 Hz
```

---

## 11. 最终启动流程

```text
飞控上电
   ↓
读取已保存参数
IMU_INTEG_RATE = 400
   ↓
vehicle_imu ≈ 293~294 Hz
   ↓
PX4 创建 MAVLink instance #0
/dev/ttyS1 @ 921600
   ↓
执行 /fs/microsd/etc/extras.txt
   ↓
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
   ↓
HIGHRES_IMU target = 250 Hz
   ↓
机载电脑 MAVROS
   ↓
/mavros/imu/data_raw ≈ 200 Hz
```

---

## 12. 为什么原来只能到 173 Hz

最终排查结果说明，原来的 173 Hz 不是以下原因导致：

- 不是 SD 卡存在 173 Hz 限制
- 不是串口 921600 带宽不足
- 不是 MAVLink `tx rate max` 不够
- 不是必须修改 `mavlink_main.cpp` 才能突破

真正原因是：

```text
IMU_INTEG_RATE = 200
        ↓
vehicle_imu ≈ 171~172 Hz
        ↓
HIGHRES_IMU 上游只有约 172 份新数据/秒
        ↓
即使目标设置 200 Hz
实际也只能约 173 Hz
```

修改后：

```text
IMU_INTEG_RATE = 400
        ↓
vehicle_imu ≈ 293~294 Hz
        ↓
HIGHRES_IMU target = 250 Hz
        ↓
接收端实际 ≈ 200 Hz
```

---

## 13. 常用检查命令

### 查看内部 IMU 发布频率

```bash
uorb top vehicle_imu
```

### 查看积分频率参数

```bash
param show IMU_INTEG_RATE
```

### 修改并保存参数

```bash
param set IMU_INTEG_RATE 400
param save
```

### 查看 MAVLink 状态

```bash
mavlink status
```

重点关注：

```text
tx
txerr
tx rate mult
tx rate max
```

正常情况应满足：

```text
txerr = 0
tx rate mult = 1.000
```

### 查看 MAVLink stream 配置

```bash
mavlink status streams
```

### 临时设置机载电脑 HIGHRES_IMU

```bash
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
```

### 临时设置 USB/QGC HIGHRES_IMU

```bash
mavlink stream -d /dev/ttyACM0 -s HIGHRES_IMU -r 250
```

### 查看 SD 卡

```bash
ls /fs/microsd
```

### 查看自动启动脚本

```bash
cat /fs/microsd/etc/extras.txt
```

### 机载电脑测试实际 ROS 频率

```bash
rostopic hz /mavros/imu/data_raw
```

---

## 14. 故障排查顺序

以后如果再次遇到 IMU 频率达不到目标，可以按以下顺序排查：

1. 先执行：

   ```bash
   uorb top vehicle_imu
   ```

   如果 `vehicle_imu` 本身低于目标频率，优先检查 `IMU_INTEG_RATE` 和 IMU 驱动，不要先改 MAVLink。

2. 如果 `vehicle_imu` 足够快，再执行：

   ```bash
   mavlink status streams
   ```

   确认 `HIGHRES_IMU` 的 Config/Current 是否达到目标。

3. 再执行：

   ```bash
   mavlink status
   ```

   检查：

   ```text
   tx rate mult
   txerr
   ```

   如果 `rate mult < 1`，说明 MAVLink 可能正在因为带宽不足而自动降频。

4. 最后在接收端实测：

   ```bash
   rostopic hz /mavros/imu/data_raw
   ```

   接收端实测频率才是最终结果。

---

## 15. 关于两个 IMU

本飞控存在两个 `vehicle_imu` 实例：

```text
vehicle_imu 0
vehicle_imu 1
```

但 `/mavros/imu/data_raw` 并不是把两个 IMU 的数据同时发布出来。

PX4 会根据当前传感器选择结果，从选中的主 IMU 生成 `HIGHRES_IMU`，因此 MAVROS 的：

```text
/mavros/imu/data_raw
```

每一帧仍然是一套：

```text
angular_velocity x/y/z
linear_acceleration x/y/z
```

两个 IMU 主要用于冗余和故障切换。

---

## 16. `data_raw` 与 `data` 的简单区别

### `/mavros/imu/data_raw`

主要是 IMU 测量数据：

- angular velocity
- linear acceleration
- 不包含 PX4 已估计出的有效姿态 quaternion

适合需要高频 IMU 测量的算法。

### `/mavros/imu/data`

主要包含：

- PX4 已估计的 orientation
- angular velocity
- linear acceleration

其发布频率通常还受到 `ATTITUDE` / `ATTITUDE_QUATERNION` MAVLink stream 频率影响，因此不一定等于 `data_raw` 的约 200 Hz。

---

## 17. 当前最终配置摘要

```text
PX4 / NuttX
IMU_INTEG_RATE = 400

vehicle_imu 0 ≈ 294 Hz
vehicle_imu 1 ≈ 293 Hz

MAVLink instance #0
/dev/ttyS1 @ 921600
HIGHRES_IMU target = 250 Hz
rate mult = 1.000
txerr = 0

SD 卡启动脚本：
/fs/microsd/etc/extras.txt

内容：
set +e
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
set -e

机载电脑：
/mavros/imu/data_raw ≈ 200 Hz
```

最终不需要为了达到 200 Hz 专门修改 PX4 `mavlink_main.cpp` 并重新编译固件。
