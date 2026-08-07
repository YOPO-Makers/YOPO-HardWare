# PX4 将 MAVROS IMU 频率提高到约 200 Hz

## 1. 在 QGroundControl 中修改 IMU 内部频率

设备：**安装 QGroundControl 的电脑**

连接：**电脑通过 USB 连接飞控**

打开：

**QGroundControl → Analyze Tools → MAVLink Console**

输入：

```bash
param set IMU_INTEG_RATE 400
param save
reboot
```

飞控重启并重新连接后，在 MAVLink Console 输入：

```bash
param show IMU_INTEG_RATE
```

确认显示：

```text
IMU_INTEG_RATE : 400
```

再输入：

```bash
uorb top vehicle_imu
```

确认 `vehicle_imu` 频率大于 200 Hz。

本机实测约：

```text
293~294 Hz
```

## 2. 临时测试机载电脑 MAVLink IMU 频率

设备：**安装 QGroundControl 的电脑**

界面：

**QGroundControl → Analyze Tools → MAVLink Console**

机载电脑连接的飞控串口为：

```text
/dev/ttyS1 @ 921600
```

输入：

```bash
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
```

检查：

```bash
mavlink status streams
```

确认 `instance #0` 中显示：

```text
HIGHRES_IMU    250.00 (250.000)
```

再输入：

```bash
mavlink status
```

确认 `instance #0`：

```text
txerr: 0.0 B/s
tx rate mult: 1.000
```

## 3. 在机载电脑确认实际频率

设备：**机载电脑**

先启动 MAVROS。

然后终端输入：

```bash
rostopic hz /mavros/imu/data_raw
```

确认实际频率约为：

```text
200 Hz
```

## 4. 设置飞控每次开机自动配置 250 Hz

设备：**安装 QGroundControl 的电脑**

界面：

**QGroundControl → Analyze Tools → MAVLink Console**

先确认 SD 卡：

```bash
ls /fs/microsd
```

创建启动脚本目录：

```bash
mkdir /fs/microsd/etc
```

创建自动启动文件：

```bash
echo "set +e" > /fs/microsd/etc/extras.txt
echo "mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250" >> /fs/microsd/etc/extras.txt
echo "set -e" >> /fs/microsd/etc/extras.txt
```

检查文件：

```bash
cat /fs/microsd/etc/extras.txt
```

应该显示：

```text
set +e
mavlink stream -d /dev/ttyS1 -s HIGHRES_IMU -r 250
set -e
```

然后重启飞控：

```bash
reboot
```

## 5. 重启后最终检查

设备：**安装 QGroundControl 的电脑**

进入：

**QGroundControl → Analyze Tools → MAVLink Console**

不要再手动设置频率，直接输入：

```bash
param show IMU_INTEG_RATE
```

确认：

```text
IMU_INTEG_RATE : 400
```

输入：

```bash
mavlink status streams
```

确认 `instance #0`：

```text
HIGHRES_IMU    250.00 (250.000)
```

然后在机载电脑输入：

```bash
rostopic hz /mavros/imu/data_raw
```

确认：

```text
约 200 Hz
```

## 最终配置

```text
IMU_INTEG_RATE = 400

机载电脑 MAVLink：
/dev/ttyS1 @ 921600

HIGHRES_IMU：
目标频率 = 250 Hz

MAVROS：
/mavros/imu/data_raw ≈ 200 Hz
```
