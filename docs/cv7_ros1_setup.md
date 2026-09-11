# MicroStrain 3DM-CV7-AR ROS1 Setup

MicroStrain 3DM-CV7-AR を Ubuntu 24.04 + ROS One (ROS1) 環境で USB 接続して使用するためのセットアップメモ。

## Tested Environment

- Ubuntu 24.04
- ROS One (ROS1)
- catkin_tools
- MicroStrain 3DM-CV7-AR
- USB connection
- Workspace: `~/ros/m_jaxon`

---

## 1. Clone `microstrain_inertial`

```bash
cd ~/ros/m_jaxon/src

git clone --recursive --branch ros \
  https://github.com/LORD-MicroStrain/microstrain_inertial.git
```

すでに clone 済みの場合は submodule を取得する。

```bash
cd ~/ros/m_jaxon/src/microstrain_inertial
git submodule update --init --recursive
```

---

## 2. Install ROS1 dependencies not provided by ROS One

ROS One 環境では、一部の依存パッケージが apt から取得できないため、
workspace 内に source package として追加する。

### `nmea_msgs`

`nmea_msgs` の default branch は ROS2 / `ament_cmake` なので、
ROS1 用の `1.1.0` を明示的に使用する。

```bash
cd ~/ros/m_jaxon/src

git clone https://github.com/ros-drivers/nmea_msgs.git
cd nmea_msgs

git fetch --all --tags
git checkout -f 1.1.0
```

ROS1版になっていることを確認する。

```bash
grep -E "catkin|rosidl|message_generation" package.xml
```

`catkin` / `message_generation` が含まれ、
`rosidl_default_generators` が含まれていなければよい。

### `rtcm_msgs`

```bash
cd ~/ros/m_jaxon/src
git clone https://github.com/tilk/rtcm_msgs.git
```

---

## 3. Install Remaining Dependencies

```bash
source /opt/ros/one/setup.bash

cd ~/ros/m_jaxon

rosdep install \
  --from-paths src \
  --ignore-src \
  -r -y
```

---

## 4. ROS One / ROS1 Compatibility Patch

Ubuntu 24.04 + ROS One 環境では、
`microstrain_inertial_driver` のビルド時に以下のようなエラーが発生する場合がある。

```text
error: cannot bind non-const lvalue reference of type ‘int&’
to a value of type ‘unsigned char’
```

ROS1 parameter server では整数 parameter を `int` として扱うため、
`uint8_t` parameter を一度 `int` として読み出すように修正する。

対象ファイル:

```text
microstrain_inertial/
└── microstrain_inertial_driver/
    └── microstrain_inertial_driver_common/
        └── include/
            └── microstrain_inertial_driver_common/
                └── utils/
                    └── ros_compat.h
```

generic な `getParam()` の後に以下を追加する。

```cpp
template <>
inline void getParam<uint8_t>(
  RosNodeType* node,
  const std::string& param_name,
  uint8_t& param_val,
  const uint8_t& default_val)
{
  int value;
  node->param<int>(
    param_name,
    value,
    static_cast<int>(default_val)
  );
  param_val = static_cast<uint8_t>(value);
}
```

---

## 5. Build

```bash
source /opt/ros/one/setup.bash

cd ~/ros/m_jaxon
catkin build
```

ビルド後:

```bash
source ~/ros/m_jaxon/devel/setup.bash
```

---

## 6. Check USB Device

CV7をUSB接続してデバイスを確認する。

```bash
ls -l /dev/ttyACM*
```

通常は以下のように認識される。

```text
/dev/ttyACM0
```

権限エラーになる場合:

```bash
sudo usermod -aG dialout $USER
```

実行後、一度ログアウトして再ログインする。

---

## 7. CV7-AR Configuration

設定ファイルを以下に作成する。

```text
microstrain_inertial/
└── microstrain_inertial_driver/
    └── config/
        └── cv7.yaml
```

必要ならディレクトリを作る。

```bash
mkdir -p \
  ~/ros/m_jaxon/src/microstrain_inertial/microstrain_inertial_driver/config
```

`cv7.yaml`:

```yaml
port: "/dev/ttyACM0"

# CV7
filter_declination_source: 1

# CV7-AR does not use an external heading source
filter_heading_source: 0

# CV7-AR has no GNSS PPS source
filter_pps_source: 0

# Disable GNSS aiding
filter_enable_gnss_pos_vel_aiding: false
filter_enable_gnss_heading_aiding: false
```

### PPS Source Error

`filter_pps_source` を無効化していない場合、
CV7-ARでは以下のエラーでdriverが終了する場合がある。

```text
Setting PPS source to 0x0001
Failed to configure PPS source
Error(3): Invalid Parameter
```

その場合は以下を設定する。

```yaml
filter_pps_source: 0
```

また、以下のようなメッセージはCV7-ARに存在しない機能についての
informational message なので、基本的には無視してよい。

```text
The device does not support publishing the topic gnss_...
```

---

## 8. Launch

環境をsourceする。

```bash
source /opt/ros/one/setup.bash
source ~/ros/m_jaxon/devel/setup.bash
```

CV7-ARを起動する。

```bash
roslaunch microstrain_inertial_driver microstrain.launch \
  params_file:=$(rospack find microstrain_inertial_driver)/config/cv7.yaml
```

---

## 9. Check ROS Topics

IMU関連topicを確認する。

```bash
rostopic list | grep -E 'imu|mip'
```

IMUデータを確認する。

```bash
rostopic echo /imu/data
```

または:

```bash
rostopic echo /imu/data_raw
```

publish frequency の確認:

```bash
rostopic hz /imu/data
```

姿勢を使用する場合は、`orientation` quaternion が入っているtopicを使用する。

```bash
rostopic echo -n 1 /imu/data
```

例えば以下が有効な値になっていることを確認する。

```text
orientation:
  x: ...
  y: ...
  z: ...
  w: ...
```

---

## 10. Optional: RViz IMU Visualization

RVizで姿勢を可視化する場合は `rviz_imu_plugin` を使用できる。

```bash
cd ~/ros/m_jaxon/src

git clone -b noetic \
  https://github.com/CCNYRoboticsLab/imu_tools.git
```

ビルド:

```bash
cd ~/ros/m_jaxon

catkin build rviz_imu_plugin
source ~/ros/m_jaxon/devel/setup.bash
```

RVizを起動する。

```bash
rviz
```

RViz上で:

1. `Add`
2. `By display type`
3. `rviz_imu_plugin`
4. `Imu`
5. CV7の `sensor_msgs/Imu` topicを指定

姿勢表示には `/imu/data_raw` よりも、
有効なorientation quaternionを含むtopicを使用する。

---

## Troubleshooting

### `Could NOT find nmea_msgs`

```text
Could NOT find nmea_msgs
```

ROS Oneではapt packageが提供されていない場合があるため、
ROS1版をsourceから追加する。

```bash
cd ~/ros/m_jaxon/src

git clone https://github.com/ros-drivers/nmea_msgs.git
cd nmea_msgs

git fetch --all --tags
git checkout -f 1.1.0
```

---

### `nmea_msgs` is detected as `ament_cmake`

```text
Skipping package `nmea_msgs` because it has an unsupported
package build type: `ament_cmake`
```

ROS2版をcloneしている。

ROS1版へ切り替える。

```bash
cd ~/ros/m_jaxon/src/nmea_msgs

git fetch --all --tags
git reset --hard
git clean -fd
git checkout -f 1.1.0
```

---

### `Could NOT find rtcm_msgs`

```bash
cd ~/ros/m_jaxon/src
git clone https://github.com/tilk/rtcm_msgs.git
```

---

### `cannot bind non-const lvalue reference ... unsigned char`

`ros_compat.h` の `uint8_t` parameter handling がROS1環境と合っていない。

本READMEの
`ROS One / ROS1 Compatibility Patch`
を適用する。

---

### `Failed to configure PPS source`

```text
Setting PPS source to 0x0001
Failed to configure PPS source
Error(3): Invalid Parameter
```

CV7-ARにはGNSS由来のPPS sourceがないため、
`cv7.yaml` に以下を設定する。

```yaml
filter_pps_source: 0
```

---

## Notes

この手順は以下の構成で確認したもの。

- Ubuntu 24.04
- ROS One / ROS1
- MicroStrain 3DM-CV7-AR
- Direct USB connection
- `catkin_tools`

特に再構築時に注意する点:

1. `nmea_msgs` はROS1版 `1.1.0` を使用する
2. ROS One環境では `uint8_t getParam()` のpatchが必要になる場合がある
3. CV7-ARでは `filter_pps_source: 0` を設定する
4. `nmea_msgs` / `rtcm_msgs` はROS Oneのapt packageではなくsourceから導入する場合がある

将来的なdriverやROS Oneの更新によって、
一部のworkaroundは不要になる可能性がある。