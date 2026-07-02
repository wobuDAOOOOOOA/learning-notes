先确定一下，那些工业库是怎么放到板子上的，我的板子好像还没联网，还有那些工具像candump和mqtt工具这些
要去搞联网好还是我从虚拟机拖过去好


联网直接一个命令全部安装完，试一下联网
ping 114.114.114.114失败，我自己的电脑ping都超时，而且我的板子跑的是buildroot构建的文件系统，用不了apt命令


具体报错
# apt update
-bash: apt: command not found
一是我连不了外网，而是我的板子是
跑的是buildroot构建的文件系统，不是Ubuntu那种带包管理器的发行版。buildroot的目标是精简——只保留你需要的功能，不装任何多余的东西。
Ubuntu/Debian系统有apt，是因为它们的目标是通用桌面或服务器，需要方便用户随时安装软件。buildroot的目标是嵌入式设备，系统镜像在编译时就确定了所有内容，不支持在线安装新软件。
你现在要用的库和工具，必须在虚拟机里用交叉编译工具链编译好，然后把二进制文件直接拖到板子上。这是嵌入式Linux开发的标准操作方式，你的板子没有少任何东西，只是它不叫Ubuntu。
你的板子就像一个计算器，出厂时所有功能都焊死在电路板上了。你想让它多做一件事，只能拆开外壳、自己焊一个新按钮上去。Ubuntu系统就像你的手机，想用什么功能，去应用商店下载一个App就行了，随时装、随时卸。
你的板子跑的是buildroot系统，它选择做计算器，因为计算器省电、稳定、不会乱装东西导致死机。代价就是不能像手机一样下载App。
你现在要做的事情，就是在虚拟机里用交叉编译工具链，把你要用的库和工具“焊”成ARM架构的二进制文件，然后通过MobaXterm拖到板子上运行。这就是嵌入式Linux开发的标准操作——不是下载，是交叉编译后部署。



find / -name "libmodbus*" 2>/dev/null
find / -name "candump" 2>/dev/null
find / -name "mosquitto_pub" 2>/dev/null
发现我的板子也没这些库，所以我要在虚拟机用交叉编译工具编译好然后放到板子上
下载源码 → 配置交叉编译 → 编译 → 把生成的 .so 或可执行文件拖到板子上



确认交叉编译工具链可用
which arm-rockchip830-linux-uclibcgnueabihf-gcc
应该输出 /opt/arm-rockchip830-linux-uclibcgnueabihf/bin/arm-rockchip830-linux-uclibcgnueabihf-gcc。如果没有，先设置PATH：
export PATH=/opt/arm-rockchip830-linux-uclibcgnueabihf/bin:$PATH



下载并交叉编译libmodbus
报错
bash: ./configure: 没有那个文件或目录
make: *** 没有指明目标并且找不到 makefile。 停止。
make: *** 没有规则可制作目标“install”。 停止。
ibmodbus源码目录里没有configure，说明这个版本可能用cmake或autogen
看到 autogen.sh，就先执行 ./autogen.sh 再 ./configure；如果看到 CMakeLists.txt，就用 cmake 编译。
看到了./autogen.sh
执行./autogen.sh
报错
root@wdz-VMware-Virtual-Platform:/home/wdz/libmodbus-3.1.10# ./autogen.sh
./autogen.sh: 2: autoreconf: not found

--------------------------
Running autoreconf failed.
--------------------------
缺少工具
sudo apt update
sudo apt install autoconf automake libtool pkg-config -y
重新执行
cd /home/wdz/libmodbus-3.1.10
./autogen.sh
./configure --host=arm-rockchip830-linux-uclibcgnueabihf --prefix=/home/wdz/libmodbus_arm
make
make install
然后在/home/wdz/libmodbus-3.1.10/src/.libs
下面把
libmodbus.so
libmodbus.so.5
libmodbus.so.5.1.0
libmodbus.la
libmodbus.lai
ibmodbus* 开头的就行，.o 和 .la 文件不需要。







下载并交叉编译libmosquitto

cd /home/wdz
wget https://github.com/eclipse/mosquitto/archive/refs/tags/v2.0.18.tar.gz
tar -xzf v2.0.18.tar.gz
cd mosquitto-2.0.18
make CC=arm-rockchip830-linux-uclibcgnueabihf-gcc CXX=arm-rockchip830-linux-uclibcgnueabihf-g++ WITH_STATIC_LIBRARIES=yes
报错
arm-rockchip830-linux-uclibcgnueabihf-gcc  -I. -I.. -I../include -I../../include -DWITH_TLS -DWITH_TLS_PSK -DWITH_THREADING -DWITH_SOCKS -DWITH_UNIX_SOCKETS -I../deps -Wall -ggdb -O2 -Wconversion -Wextra -fPIC -c mosquitto.c -o mosquitto.o
In file included from mosquitto.c:19:
../config.h:61:12: fatal error: openssl/opensslconf.h: No such file or directory
 #  include <openssl/opensslconf.h>
            ^~~~~~~~~~~~~~~~~~~~~~~
compilation terminated.
make[1]: *** [Makefile:102：mosquitto.o] 错误 1
make[1]: 离开目录“/home/wdz/mosquitto-2.0.18/lib”
make: *** [Makefile:66：mosquitto] 错误 2
root@wdz-VMware-Virtual-Platform:/home/wdz/mosquitto-2.0.18#
mosquitto编译依赖OpenSSL，你的交叉编译工具链里没有ARM版的OpenSSL头文件
MQTT上云用的是华为云IoTDA的密钥认证，不需要TLS加密。直接编译一个不带TLS的mosquitto：
make clean
make CC=gcc CXX=g++ CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- WITH_TLS=no WITH_STATIC_LIBRARIES=yes
报错
arm-rockchip830-linux-uclibcgnueabihf-gcc  -I.. -I../include -DWITH_THREADING -DWITH_SOCKS -DWITH_UNIX_SOCKETS -Wall -ggdb -O2 -Wconversion -Wextra -DVERSION="\"2.0.18\"" -DWITH_CJSON -c pub_client.c -o pub_client.o
In file included from pub_client.c:19:
../config.h:86:12: fatal error: cjson/cJSON.h: No such file or directory
 #  include <cjson/cJSON.h>
            ^~~~~~~~~~~~~~~
compilation terminated.
make[1]: *** [Makefile:42：pub_client.o] 错误 1
make[1]: 离开目录“/home/wdz/mosquitto-2.0.18/client”
make: *** [Makefile:66：mosquitto] 错误 2
mosquitto的客户端工具也依赖cJSON，交叉编译链里也没有。继续跳过，只编译核心库：
cd /home/wdz/mosquitto-2.0.18
make clean
make CC=gcc CXX=g++ CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- WITH_TLS=no WITH_CJSON=no WITH_STATIC_LIBRARIES=yes
之前编译报错是因为缺少依赖，直接关掉所有不需要的功能，只编译核心库：
cd /home/wdz/mosquitto-2.0.18
make clean
make CC=gcc CXX=g++ CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- WITH_TLS=no WITH_CJSON=no WITH_SHARED_LIBRARIES=no WITH_STATIC_LIBRARIES=yes
报错
make[1]: 离开目录“/home/wdz/mosquitto-2.0.18/src”
set -e; for d in man; do make -C ${d}; done
make[1]: 进入目录“/home/wdz/mosquitto-2.0.18/man”
xsltproc --nonet libmosquitto.3.xml
make[1]: xsltproc: 没有那个文件或目录
make[1]: *** [Makefile:90：libmosquitto.3] 错误 127
make[1]: 离开目录“/home/wdz/mosquitto-2.0.18/man”
make: *** [Makefile:57：docs] 错误 2
mosquitto卡在生成文档这一步，对你的需求没用，跳过它，直接只编译库：
cd /home/wdz/mosquitto-2.0.18/lib
make clean
make CC=gcc CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- WITH_TLS=no
libmosquitto.so.1 拖到板子 /lib/ 目录下，和 libmodbus 放一起。三件套齐了







下载并交叉编译can-utils
cd /home/wdz
git clone https://github.com/linux-can/can-utils.git
cd can-utils
make CC=arm-rockchip830-linux-uclibcgnueabihf-gcc
在/home/wdz/can-utils/目录下找到candump和cansend文件，没有任何后缀
要把这两个文件放到系统默认从 /usr/bin/ 和 /bin/ 找命令
然后敲命令就行了。


重新写一个makefile用交叉编译工具来编译
指定的头文件路径要用我交叉编译出来的那些库
# 交叉编译工具链
CC = arm-rockchip830-linux-uclibcgnueabihf-gcc

# 目录
SRC_DIR = src
INC_DIR = include
LIB_DIR = /home/wdz/libmodbus_arm/lib
MOSQ_DIR = /home/wdz/mosquitto-2.0.18/lib

# 源文件
SRCS = $(SRC_DIR)/main.c \
       $(SRC_DIR)/modbus_rtu.c \
       $(SRC_DIR)/modbus_tcp.c \
       $(SRC_DIR)/can.c \
       $(SRC_DIR)/mqtt_huawei.c \
       $(SRC_DIR)/config.c \
      # $(SRC_DIR)/log.c

# 输出
TARGET = gateway

# 编译选项
CFLAGS = -Wall -O2 -I$(INC_DIR) -I/home/wdz/libmodbus_arm/include -I/home/wdz/mosquitto-2.0.18/include
LDFLAGS = -L$(LIB_DIR) -L$(MOSQ_DIR) -lmodbus -lmosquitto -lpthread

all: $(TARGET)

$(TARGET): $(SRCS)
	$(CC) $(CFLAGS) -o $@ $^ $(LDFLAGS)

clean:
	rm -f $(TARGET)

.PHONY: all clean

报错
/opt/arm-rockchip830-linux-uclibcgnueabihf/bin/../lib/gcc/arm-rockchip830-linux-uclibcgnueabihf/8.3.0/../../../../arm-rockchip830-linux-uclibcgnueabihf/bin/ld.bfd: cannot find -lmosquitto
链接器找不到libmosquitto。你刚才编译mosquitto时只生成了libmosquitto.so.1，但链接器需要libmosquitto.so这个不带版本号的符号链接
cd /home/wdz/mosquitto-2.0.18/a
ln -sf libmosquitto.so.1 libmosquitto.so
ls -la libmosquitto*




vi /etc/usb_config

insmod /lib/can.ko
insmod /lib/can-raw.ko 2>&1 > /dev/null &
insmod /lib/can-dev.ko
insmod /lib/gs_usb.ko
lsmod | grep can

ip addr add 192.168.2.100/24 dev eth0
ip link set eth0 up


source venv/bin/activate
python slave.py
source /home/wdz/gateway/modbus_tcp_slave/venv/bin/activate   # 如果虚拟环境在 gateway 下
python slave.py

ncpa.cpl

ls
chmod +x gateway
./gateway





第一步：在虚拟机里彻底清理，只编译一次
cd /home/wdz/sdk/sysdrv/source/kernel
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- distclean
cd /home/wdz/sdk
rm -rf sysdrv/source/objs_kernel

然后重新配置并编译：
cd /home/wdz/sdk/sysdrv/source/kernel
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- rv1106_lbc_defconfig
sed -i 's/CONFIG_SMP=y/CONFIG_SMP=n/' .config
sed -i 's/# CONFIG_THUMB2_KERNEL is not set/CONFIG_THUMB2_KERNEL=y/' .config
echo "CONFIG_CAN=m" >> .config
echo "CONFIG_CAN_RAW=m" >> .config
echo "CONFIG_CAN_DEV=m" >> .config
echo "CONFIG_CAN_GS_USB=m" >> .config
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- olddefconfig
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- -j$(nproc) vmlinux modules

第二步：编译完成后，立即备份所有产物
mkdir -p /home/wdz/can_production
cp /home/wdz/sdk/sysdrv/out/image_uclibc_rv1106/boot.img /home/wdz/can_production/
find /home/wdz/sdk/sysdrv/source/kernel -name "can.ko" -o -name "can-raw.ko" -o -name "can-dev.ko" -o -name "gs_usb.ko" | while read f; do cp "$f" /home/wdz/can_production/; done

第三步：板子上彻底清理旧模块
rm -f /lib/can*.ko /lib/gs_usb.ko
find /lib/modules -name "*.ko" -delete
reboot

第四步：板子重启后，传文件并加载
/home/wdz/can_production/ 下的 boot.img


这次编译时，CONFIG_CAN_VCAN 没开
cd /home/wdz/sdk/sysdrv/source/kernel
echo "CONFIG_CAN_VCAN=m" >> .config
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- olddefconfig
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- modules_prepare
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- M=drivers/net/can modules
编译完后：
find drivers/net/can -name "vcan.ko"
vcan模块正常加载但是用需要用到can.KO和can_raw.ko,问题就绕回来了，insmod之后内核就会崩溃。

从“源码树不干净”到“vermagic不匹配”再到“二进制不兼容”，根本原因始终是：这块板子的出厂内核不支持动态加载任何与网络子系统相关的第三方模块

行吧，我发现这个用网线不开外设模式他也能用ssh，但是每次重新上电又要输入ip addr add 192.168.2.100/24 dev eth0
ip link set eth0 up，这有什么能永久解决吗
vi /etc/init.d/rcS
在文件末尾添加
# 设置静态IP
ip addr add 192.168.2.100/24 dev eth0
ip link set eth0 up
reboot重启板子验证


查看命令的设备
ls -la /dev/ttyS*
dmesg | grep -i "tty\|uart"

