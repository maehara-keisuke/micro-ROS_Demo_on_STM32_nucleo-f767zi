# micro-ROS_Demo_on_STM32_nucleo-f767zi
note : Sorry, this document is written in Japanese.

## このリポジトリについて
[ROS Japan UG #36 LT@Google Meet](https://rosjp.connpass.com/event/174104/)にて"micro-ROS Demo on STM32 nucleo-f767zi"というタイトルで発表した内容を補完し，nucleo-f767ziを所持している方が追試できるようにするために，パッチの公開とチュートリアルを示すためのリポジトリである．

パッチの中身についてはほぼmicro-ROSのソースファイル由来のものであるので，基本的なライセンス条項はそれを継承するものとする．

環境としてUbuntu18.04(amd64)を用いる．なお，クロスコンパイルのためのツールチェーンやフラッシュメモリへの書き込みツール，およびMicro XRCE-DDS Agentのインストールは済んでいるものとし，本チュートリアルでは説明しない．

## ゴールイメージ
![goal image](https://github.com/maehara-keisuke/micro-ROS_Demo_on_STM32_nucleo-f767zi/blob/images/micro-ROS_Demo_on_STM32_nucleo-f767zi.gif)

nucleo boardは剥き出しだと裏面のピンがチクチクするので，スマホのバンパーケースみたいなものをFusion360でモデリングして，3Dプリントサービスで造形してみた．塗装していないので色ムラがあるが，強度や質感は良い感じである．これだけで随分取り回しやすくなった．

## チュートリアル

### Micro-ROS configuration for NuttX

1. ターミナルを起動する

2. ROS2の環境変数を読み込む．パスは手元の環境に従って適宜読み替えること．

```sh
cd ~

# ROS2 Dashing Diademataを公式のチュートリアルに従ってソースからビルドしている場合の例
source ros2_dashing/install/setup.bash

# aptからバイナリインストールしている場合の例
source /opt/ros/dashing/setup.bash
```

3. 作業用ディレクトリの作成

```sh
cd ~
mkdir uros_ws
```
このuros_wsを作業用ディレクトリとしても問題ないが，ある命名規則に従ってサブディレクトリを作成し，そこを作業用ディレクトリとすることを推奨する．

```sh
cd uros_ws
mkdir nuttx__nucleo-144__f767-netnsh
cd nuttx__nucleo-144__f767-netnsh
```

本チュートリアルにおいては，このnuttx__nucleo-144__f767-netnshを作業ディレクトリとする．

なぜこのようなサブディレクトリを作成するかと言うと，あるボードでmicro-ROSを動作させることに成功すると，他のボードや他のRTOSでも動作させたくなってくるからである（なるよね？）．

チェックアウトしたソースを使いまわしながら他のボードの設定ファイルを上書きしたりすると，訳がわからなくなるので推奨できない（やろうと思う人もいないと思うが）．

uros_wsをmicro-ROS関連のルートディレクトリと位置付け，その直下にRTOSの種類（これをRTOS nameなどと呼ぶ）やボードの種類（これをplatformなどと呼ぶ），および基本設定（これをconfigurationなどと呼ぶ）を組み合わせてサブディレクトリを作成しておけば，後々ディレクトリの目的がわかりやすい．

大抵のボードについては，ボードの名前ごとにplatformが設定され，その下に"nsh"や"usbnsh"などのconfigurationがぶら下がるという命名規則に従っている．

例としてnucleo-f446reを挙げると

|RTOS name|platform|configuration|
|:---:|:---:|:---:|
|nuttx|nucleo-f446re|nsh|

という組み合わせが可能である．

なおnucleo-f767ziを含むいくつかのボードについては，platformやconfigurationの命名規則がやや変則的ということに注意が必要である．
platformとしてnucleo-144という名前に集約され，その下にボード名を含む形で"f767-evalos"や"f722-nsh"などのconfigurationがぶら下がっている．

本チュートリアルにおいては

|RTOS name|platform|configuration|
|:---:|:---:|:---:|
|nuttx|nucleo-144|f767-netnsh|

となるので，上記のようなサブディレクトリ名を用いる．

4. Workspace set-up

```sh
git clone -b $ROS_DISTRO https://github.com/micro-ROS/micro-ros-build.git src/micro-ros-build
rosdep update && rosdep install --from-path src --ignore-src -y
colcon build
source install/local_setup.bash
```

5. Set the base configuration

```sh
# シェルスクリプトの引数は RTOS name と platform
ros2 run micro_ros_setup create_firmware_ws.sh nuttx nucleo-144

# この段階で，firmwareディレクトリ以下に実際のNuttXのソース（NuttX）やXRCE DDS関連のソース（mcu_ws）がチェックアウトされているので，どんなplatformやconfigurationがサポートされている（可能性が高い）か，firmware/NuttX/configs以下を覗いてみると良い

# シェルスクリプトの引数は configuration
ros2 run micro_ros_setup configure_firmware.sh f767-netnsh
```

6. RMWまわりの不具合回避

2020.05.15現在，チュートリアル通りにファームウェアのビルドを実行していくと，ビルドは通るものの実行時に若干の不具合らしき現象が見られる．nsh上でpublisher exampleを1度実行し，正常終了した後に再びpublisher exampleを実行するとダンプを吐いて死んでしまうという現象が起きている．

2020.05.06にrmw_microxrceddsについて，メモリ初期化に関する変更が行われた影響と思われる（コミットID : f6d2ef61aaef7a93734bcd848bf1566c02ecee6d）．これを回避するため，rmw_microxrceddsについて1つ前のコミット（コミットID : 34ed379e4a838201e1bfe36325074c1629db2372）を指定する．

```sh
cd firmware/mcu_ws/uros/rmw_microxrcedds
git checkout 34ed379e4a838201e1bfe36325074c1629db2372
```
将来的には，本節の操作は必要なくなる可能性がある．

7. Micro-ROS Configuration

```sh
cd ~/uros_ws/nuttx__nucleo-144__f767-netnsh/firmware/NuttX
make menuconfig
```

以下に，make menuconfigで設定する項目を挙げる．

なお今回のデモンストレーションの目的に必須ではないが，ボード上のLEDをユーザアプリから使えるようにする設定と，そのサンプルアプリも有効にする（デフォルトではLEDはシステムの状態を表示するためにカーネルがハンドリングしている）．これは，将来的にmicro-ROS側でのsubscribeを確認するための"Lチカ"を実装できるようにしたり，ホストPCからtopicが届く周期の正確性を測定できるようにするための布石である．

区分にverifyと記されているものについては確認のみで良い．addと記されている項目は，公式サイトのチュートリアルに加えて設定するべき項目である．

|区分|設定箇所|設定値|
|---|---|---|
|verify|System Type > STM32 Peripheral Support > USART3|有効|
|add|Board Selection > Board LED Status support|無効|
|add|Board Selection > Button interrupt support|有効|
|add|Board Selection > Enable boardctl() interface|有効|
|   |RTOS Features > Clocks and Timers > Support CLOCK_MONOTONIC|有効|
|add|Device Drivers > Input Device Support|有効|
|add|Device Drivers > Input Device Support > Button Inputs|有効|
|add|Device Drivers > Input Device Support > Button Inputs　> Generic Lower Half Button Driver|有効|
|add|Device Drivers > LED Support > LED Driver|有効|
|add|Device Drivers > LED Support > LED Driver > Generic Lower Half LED Driver|有効|
|   |Device Drivers > Serial Driver Support > Serial TERMIOS support|有効|
|   |Library Routines > Standard Math Library|有効|
|   |Library Routines > sizeof(_Bool) is a 8-bits|有効|
|   |Library Routines > Build uClibc++ (must be installed)|有効|
|add|Application Configuration > NSH Library > Have architecture-specific initialization|有効|
|add|Application Configuration > NSH Library > Network Configuration > Network initialization > IP Address Configuration > Use DHCP to get IP address|有効|
|add|Application Configuration > NSH Library > Network Configuration > Network initialization > IP Address Configuration > Router IPv4 address|DHCPサーバとなるルーターのIPアドレスを16進表記で設定|
|add|Application Configuration > NSH Library > Network Configuration > Network initialization > IP Address Configuration > IPv4 Network mask|ネットマスクを16進表記で設定．大抵の場合はデフォルトの0xffffff00(= 255.255.255.0)で問題ない|
|add|Application Configuration > NSH Library > Network Configuration > Network initialization > IP Address Configuration > Hardware has no MAC address|有効|
|   |Application Configuration > micro-ROS|有効|
|verify|Application Configuration > micro-ROS > Transport > UDP Transport|有効|
|add|Application Configuration > micro-ROS > IP address of the agent|DDS agentを立ち上げるホストPCのIPアドレスを10進表記で設定|
|add|Application Configuration > Examples > Buttons driver example|有効|
|add|Application Configuration > Examples > Buttons driver example > Show Buttons Names|有効|
|add|Application Configuration > Examples > LED driver example|有効|
|   |Application Configuration > Examples > microROS Publisher|有効|

### Adding Micro-ROS to a NuttX board configuration

8. Add C++ standard library includes

firmware/NuttX/Make.defsにuClibc++のヘッダへのパスを追加する．

```Makefile
else
  # Linux/Cygwin-native toolchain
  MKDEP = $(TOPDIR)/tools/mkdeps$(HOSTEXEEXT)
  ARCHINCLUDES = -I. -isystem $(TOPDIR)/include
  ARCHXXINCLUDES = -I. -isystem $(TOPDIR)/include -isystem $(TOPDIR)/include/cxx
  ARCHXXINCLUDES+=-isystem $(TOPDIR)/include/uClibc++                               # <- here!
  ARCHSCRIPT = -T$(TOPDIR)/configs/$(CONFIG_ARCH_BOARD)/scripts/$(LDSCRIPT)
endif
```

9. Add C++ atomics builtins

libatomicの追加．

```sh
cd ~/uros_ws/nuttx__nucleo-144__f767-netnsh/
cp firmware/NuttX/configs/olimex-stm32-e407/src/libatomic.c firmware/NuttX/configs/nucleo-144/src
```

firmware/NuttX/configs/nucleo-144/src/Makefileにlibatomic.cのエントリを追加する．

```Makefile
CSRCS = stm32_boot.c libatomic.c
                     #  ^
                     #  |
                     # here!
```

### ボタンデバイス（/dev/buttons）のレジストレーションとpublisher exampleの改造

10. ボタンデバイス（/dev/buttons）のレジストレーション

システム起動時にボタンデバイスを登録するコードが抜けていたので，他のボードを参考に書いたパッチを適用する．
```sh
cd ~/uros_ws/nuttx__nucleo-144__f767-netnsh/firmware/NuttX/configs/nucleo-144/src
wget https://raw.githubusercontent.com/maehara-keisuke/micro-ROS_Demo_on_STM32_nucleo-f767zi/master/firmware/NuttX/configs/nucleo-144/src/stm32_appinitialize.patch
patch stm32_appinitialize.c stm32_appinitialize.patch
```

11. publisher exampleの改造

micro-ROSに同梱されているpublisher exampleは，指定回数ループでagentにtopicを送りつけるものである．またNuttXに同梱されているbuttons exampleはボタンを押す or 離すといったイベントを待ち受け，イベントが発生するとコンソールに表示するものである．publisher exampleにbuttons exampleのコードを付け加えることで，ボタンのイベントが発生するたびにagentにtopicを送りつけるように改造した．

```sh
cd ~/uros_ws/nuttx__nucleo-144__f767-netnsh/firmware/apps/examples/publisher
wget https://raw.githubusercontent.com/maehara-keisuke/micro-ROS_Demo_on_STM32_nucleo-f767zi/master/firmware/apps/examples/publisher/publisher_main.patch
patch publisher_main.c publisher_main.patch
```

### ファームウェアのビルドと書き込み

12. ファームウェアのビルド

```sh
cd ~/uros_ws/nuttx__nucleo-144__f767-netnsh/
ros2 run micro_ros_setup build_firmware.sh
```

13. ファームウェアの書き込み

st-linkの側のmicroUSBポートをホストPCと接続し，以下の書き込みツールで書き込む．nucleo-f767ziはmbedにも対応しているので，デスクトップ上に現れるUSB Mass Storageにドラッグ・アンド・ドロップするのでも良い．

```sh
st-flash write firmware/NuttX/nuttx.bin 0x8000000
```

### デモアプリの実行

好みのターミナルアプリでNuttXのシリアルコンソールに接続する．デバイスファイルの名前は通常"/dev/ttyACM0"となり，ボーレートは"115200"を設定する．
DHCPサーバ機能を持つルータとEthernetケーブルで接続し，いったんボード上のリセットボタンによりシステムをリセットする．IPの配布が正常に行われれば，ターミナルアプリにNuttXのプロンプトが表示される．

ホストPCの側でMicro XRCE-DDS Agentを立ち上げる．

```sh
/usr/local/bin/MicroXRCEAgent udp --port 8888
```

NuttXのシリアルコンソールでpublisherを実行する

```sh
publisher
```

この段階でMicro XRCE-DDS Agentの画面にセッションが確立されたというメッセージが表示されていればOK．

ホストPCの側で別のターミナルを立ち上げ，topicが見えているか確認する．

```sh
# ROS2 Dashing Diademataを公式のチュートリアルに従ってソースからビルドしている場合の例
source ros2_dashing/install/setup.bash

# aptからバイナリインストールしている場合の例
source /opt/ros/dashing/setup.bash

ros2 topic list
/parameter_events
/rosout
/std_msgs_msg_Int32
```

/std_msgs_msg_Int32が見えていればOK．

topicを待ち受ける．

```sh
ros2 topic echo /std_msgs_msg_Int32
```

後はゴールイメージで示したように，ボタンを押したり離したりしたタイミングでtopicを受信できることを確認できればOK．

以上
