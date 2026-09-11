# MicroStrain 3DM-CV7-AR: Ubuntu 24.04 + ROS One (ROS1)

USB接続の3DM-CV7-ARを使うための手順です。Ubuntu 24.04にROS One (ROS1)、
`catkin_tools`、`rosdep`が導入済みであることを前提にします。
workspaceは `~/ros_ws` とします。

## 1. このbranchを取得

```bash
mkdir -p ~/ros_ws/src
cd ~/ros_ws/src
git clone --recursive --branch m-jaxon-dev \
  https://github.com/makabe0510/microstrain_inertial.git
cd microstrain_inertial
```

既存のcloneでは、対象branchに切り替えたうえで次を実行します。

```bash
git submodule update --init --recursive
```

## 2. ROS1互換性patchを適用

ROS Oneで `uint8_t` parameterの読み取りをビルドすると、次のエラーが
発生する場合があります。

```text
cannot bind non-const lvalue reference of type ‘int&’ to a value of type ‘unsigned char’
```

`patches/ros1-uint8-param.patch` は、ROS1側の `getParam<uint8_t>()` を追加し、
一度 `int` として読み取ってから `uint8_t` に変換します。
対象はdriver_commonの `include/microstrain_inertial_driver_common/utils/ros_compat.h` です。

親repoはupstreamのsubmoduleコミット
`4c04698ff31d8ea2f99300dcbb3f87657c023fd5` を固定参照し、patchを別途管理します。
submoduleのforkは不要です。clone後、ビルド前に次を実行してください。

```bash
cd ~/ros_ws/src/microstrain_inertial
patch_file="$PWD/patches/ros1-uint8-param.patch"
common_dir="microstrain_inertial_driver/microstrain_inertial_driver_common"
git -C "$common_dir" apply --check "$patch_file" && \
  git -C "$common_dir" apply "$patch_file"
```

適用後に親repoの `git diff` でsubmoduleが `-dirty` と表示されるのは正常です。
修正内容は親repoのpatchに保存されています。修正済みsubmoduleのコミットを
親repoの参照として登録する必要はありません。

再実行時に適用チェックが失敗したら、適用済みか次で確認します。

```bash
git -C "$common_dir" apply --reverse --check "$patch_file"
```

成功すればpatchの変更はすでに存在します。両方のチェックが失敗する場合は、
submoduleのコミットと `git -C "$common_dir" diff` を確認してください。
submoduleの更新・再取得後は、ビルド前にpatchの適用状態を確認します。

## 3. ROS1依存パッケージをsourceで追加

`nmea_msgs` はROS1版 **1.1.0** を明示して取得します。
`rtcm_msgs` もworkspace内にsourceで追加します。
以下は、同名ディレクトリがまだ存在しない場合の手順です。

```bash
cd ~/ros_ws/src
git clone --branch 1.1.0 --depth 1 https://github.com/ros-drivers/nmea_msgs.git
git clone https://github.com/tilk/rtcm_msgs.git
```

既存の `nmea_msgs` がROS2版の場合は、ローカル変更を保存してから
`git fetch --tags` と `git switch --detach 1.1.0` で切り替えます。
`ament_cmake` に関するエラーが出る場合は、ROS1版を使用しているか確認してください。

残りの依存を解決します。

```bash
source /opt/ros/one/setup.bash
cd ~/ros_ws
rosdep install --from-paths src --ignore-src -r -y
```

依存解決エラーが残った場合は、内容を確認して解消してからビルドします。

## 4. ビルド

```bash
source /opt/ros/one/setup.bash
cd ~/ros_ws
catkin build
source ~/ros_ws/devel/setup.bash
```

## 5. USBとCV7-AR設定

デバイスを接続してportを確認します。

```bash
ls -l /dev/ttyACM*
```

アクセス権限が足りない場合は次を実行し、ログアウトして再ログインします。

```bash
sudo usermod -aG dialout "$USER"
```

このbranchに含まれる `microstrain_inertial_driver/config/cv7.yaml` を使用します。
既定のportは `/dev/ttyACM0` です。実際のportが異なる場合のみ変更してください。

CV7-ARでは **`filter_pps_source: 0` が必要**です。この設定は既存の
`cv7.yaml` に含まれています。`Failed to configure PPS source` が出た場合は、
この設定ファイルを読み込んでいることを確認してください。
heading sourceとGNSS aidingの設定も同ファイルに含まれます。

## 6. 起動とデータ確認

```bash
source /opt/ros/one/setup.bash
source ~/ros_ws/devel/setup.bash
roslaunch microstrain_inertial_driver microstrain.launch \
  params_file:="$(rospack find microstrain_inertial_driver)/config/cv7.yaml"
```

別のterminalで環境を読み込み、topicを確認します。

```bash
source /opt/ros/one/setup.bash
source ~/ros_ws/devel/setup.bash
rostopic list | grep -E 'imu|mip'
rostopic echo -n 1 /imu/data
rostopic hz /imu/data
```

実際に公開されているtopic名を使用してください。姿勢を利用する場合は、
`orientation` と `orientation_covariance` を確認します。
`orientation_covariance[0]` が `-1` の場合は姿勢データが提供されていません。

## 再現性の範囲

互換性修正はこのrepo内のpatchと固定submoduleコミットで再現できます。
`nmea_msgs` は1.1.0を指定していますが、ROS Oneのシステムパッケージや
`rtcm_msgs` のバージョンまで固定する手順ではありません。
