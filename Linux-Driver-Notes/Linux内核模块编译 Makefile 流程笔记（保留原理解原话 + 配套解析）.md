先定义好内核源码的文件位置，然后告诉内核目标架构以及指定交叉编译器前缀把这两个变量导出，export命令可以将这两个传给内核。然后就是模块定义`obj-m := helloworld.o`，之后执行make命令时：`-C`改变路径到内核，`M=`告诉内核当前makefile的路径，然后加上`modules`告诉内核我要编译`M=xxx`这个目标。make命令是外置makefile的指令，modules命令内核makefile的指令。

//lsmod可以查看当前加载的所有内核模块
//insmod 后要跟内核模块的路径
//rmmod 后面直接跟内核模块的文件名
//dmesg可以查看所有内核日志

这个是之前写的，现在来理解一下，
obj-m ：编译出来的名字
KDIR：内核源码位置
PWD：此makefile也就是外部makefile的位置
all：
$(MAKE)【外部makefile的命令，在shell里面执行make的时候体现】 -C【改变路径】$(KDIR)【变到内核源码所在位置】 M= 【告诉内核我要编译的目标的路径，pwd就是当前路径的命令】 modules 最后的这个是给内核makefile的命令，告诉他我要编译

obj-m += hello_drv.o

KDIR := /lib/modules/$(shell uname -r)/build
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean





# Linux内核模块编译 Makefile 流程笔记（保留原理解原话 + 配套解析）

## 一、个人理解原话（一字未改）

先定义好内核源码的文件位置，然后告诉内核目标架构以及指定交叉编译器前缀把这两个变量导出，export命令可以将这两个传给内核。然后就是模块定义`obj-m := helloworld.o`，之后执行make命令时：`-C`改变路径到内核，`M=`告诉内核当前makefile的路径，然后加上`modules`告诉内核我要编译`M=xxx`这个目标。make命令是外置makefile的指令，modules命令内核makefile的指令。

## 二、完整代码示例（对应上述流程）

```Makefile

# 1. 定义内核源码目录
KERNEL_DIR=/home/pi/build

# 2. 指定目标架构、交叉编译器前缀
ARCH=arm
CROSS_COMPILE=arm-linux-gnueabihf-
# 3. 导出变量，供内核Makefile读取
export ARCH CROSS_COMPILE

# 4. 定义要编译的内核模块
obj-m := helloworld.o

# 编译目标
all:
 $(MAKE) -C $(KERNEL_DIR) M=$(CURDIR) modules

# 清理目标
clean:
 $(MAKE) -C $(KERNEL_DIR) M=$(CURDIR) clean

# 伪目标，避免和同名文件冲突
.PHONY: clean
```

## 三、逐段流程详解

### 1. 基础路径与编译环境配置

1. `KERNEL_DIR=/home/pi/build`

手动指定**内核源码/内核编译目录**，编译模块必须依赖对应版本的内核源码。

1. `ARCH=arm`

声明目标硬件架构为ARM，告知编译系统最终模块运行在ARM设备上。

1. `CROSS_COMPILE=arm-linux-gnueabihf-`

指定交叉编译器前缀，编译出适配ARM架构的程序。

1. `export ARCH CROSS_COMPILE`

将`ARCH`、`CROSS_COMPILE`两个变量导出为**环境变量**。后续调用内核自身的Makefile时，内核可以直接读取这两个变量，沿用架构和编译器配置。

### 2. 模块声明

`obj-m := helloworld.o`

内核编译体系专属语法，表示将当前目录下的`helloworld.c`编译为**动态内核模块**，最终自动生成`helloworld.ko`文件。

### 3. 核心编译指令解析

指令：`$(MAKE) -C $(KERNEL_DIR) M=$(CURDIR) modules`

1. `$(MAKE)`：调用GNU make工具，属于**当前自定义（外置）Makefile**的指令。

2. `-C $(KERNEL_DIR)`：切换工作目录到内核源码目录，去执行**内核自带的Makefile**。

3. `M=$(CURDIR)`：`CURDIR`代表当前目录，作用是告诉内核编译系统：**待编译的模块源码、自定义Makefile所在路径**。

4. `modules`：**内核自带Makefile的内置目标**，含义为：编译`M=`指定目录下的所有内核模块。

### 4. 清理指令

`$(MAKE) -C $(KERNEL_DIR) M=$(CURDIR) clean`

调用内核Makefile的`clean`目标，清理编译产生的`.ko`、`.o`、临时文件等编译产物。

### 5. 伪目标 `.PHONY: clean`

- 作用：标记`clean`为伪目标。

- 原理：如果目录中存在名为`clean`的普通文件，不加`.PHONY`时，`make clean`会判定目标已存在，**不会执行清理命令（短路问题）**；添加后，无论是否存在同名文件，都会强制执行清理动作。

## 四、整体流程总览

1. 自定义Makefile：配置内核路径、架构、交叉编译器，导出环境变量；

2. 声明需要编译的内核模块；

3. 执行`make`：外置Makefile调用内核Makefile，切换到内核目录、指定模块目录；

4. 内核Makefile根据`modules`目标，完成`.ko`模块编译；

5. 执行`make clean`：调用内核清理规则，清空编译文件。

## 五、核心总结

1. 存在**两份Makefile**：自己编写的外置Makefile（发起编译请求）、内核自带Makefile（实际完成编译工作）；

2. `export` 实现变量跨Makefile传递，让内核读取架构与编译器配置；

3. `-C` 切换到内核目录，`M=` 指定模块源码目录，`modules` 是内核专属编译模块指令。





