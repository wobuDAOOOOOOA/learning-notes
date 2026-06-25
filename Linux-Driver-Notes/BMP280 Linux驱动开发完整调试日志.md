# BMP280 Linux驱动开发完整调试日志（实操排错复盘手册）

## 一、前期硬件与设备树准备

### 1. 设备树文件定位

目标设备树路径：`/home/wdz/sdk/sysdrv/source/kernel/scripts/dtc/include-prefixes/arm/rv1106g-evb1-v10.dts`

最终生效设备树：**rv1106g-lubancat-rv06.dts**（前期选错DTS文件导致编译不生效）

### 2. 设备树节点配置（最终正确配置）

挂载硬件I2C2总线，BMP280设备地址经STM32最小系统板验证：**7位从机地址 0x77**

```dts
&i2c3 {
    status = "okay";
    clock-frequency = <100000>;

    /* BMP280气压温度传感器 */
    bmp280@77 {
        compatible = "bmp280,press-temp-sensor";
        reg = <0x77>;
        status = "okay";
    };
};
```

## 二、驱动模块首次编译报错（函数签名不匹配）

### 1. 报错信息

- `open  = bmp280_open`：指针类型不匹配，内核open函数签名参数不匹配

- `probe  = bmp280_probe`：i2c驱动probe函数签名与高版本内核不兼容

- 核心原因：内核将警告强制转为错误（`-Werror`），函数参数写法不符合当前RV1106内核标准

**关键完整报错日志：**

`home/wdz/test_drv/bmp280_drv.c:191:14: error: initialization of 'int (*)(struct inode *, struct file *)' from incompatible pointer type 'int (*)(struct file *)' [-Werror=incompatible-pointer-types]
 .open  = bmp280_open,
          ^~~~~~~~~~~
home/wdz/test_drv/bmp280_drv.c:191:14: note: (near initialization for 'bmp280_fops.open')
home/wdz/test_drv/bmp280_drv.c:295:15: error: initialization of 'int (*)(struct i2c_client *, const struct i2c_device_id *)' from incompatible pointer type 'int (*)(struct i2c_client *)' [-Werror=incompatible-pointer-types]
 .probe  = bmp280_probe,
           ^~~~~~~~~~~~
home/wdz/test_drv/bmp280_drv.c:295:15: note: (near initialization for 'bmp280_i2c_driver.probe')
` `cc1: all warnings being treated as errors`

### 2. 报错根因

高版本Linux内核收紧了驱动函数校验：

- 字符设备open函数必须携带 `struct inode *, struct file *` 双参数

- I2C probe函数必须携带 `struct i2c_client *, const struct i2c_device_id *` 双参数

- 旧写法单参数会直接编译失败

## 三、内核编译报错：源码树不清洁（The source tree is not clean）

### 1. 报错原因

内核源码残留历史编译的`.config`、缓存.o文件，SDK编译脚本要求**纯净源码树**，禁止残留编译产物。

**关键完整报错日志：**

`***
***   { The source tree is not clean},   please run 'make ARCH=arm mrproper'
*** in /home/wdz/sdk/sysdrv/source/kernel
***
` `make[2]: *** [/home/wdz/sdk/sysdrv/source/kernel/Makefile:575：outputmakefile] 错误 1`

### 2. 根治清理命令

```bash
# 进入内核目录执行纯净清理
cd /home/wdz/sdk/sysdrv/source/kernel
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- mrproper
```

### 3. 恢复内核关键配置（mrproper会清空.config）

清理后会丢失SMP、Thumb2配置，必须手动恢复，否则驱动编译异常：

```bash
cd /home/wdz/sdk/sysdrv/source/kernel
sed -i 's/CONFIG_SMP=y/CONFIG_SMP=n/' .config
sed -i 's/# CONFIG_THUMB2_KERNEL is not set/CONFIG_THUMB2_KERNEL=y/' .config
make ARCH=arm CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf- modules_prepare

# 清理驱动目录并重新编译驱动
cd /home/wdz/test_drv && make clean && make
```

### 4. 单独编译设备树（不破坏系统配置）

```bash
cd /home/wdz/sdk
./build.sh kernel 2>&1 | tee build.log
```

生成设备树产物路径：`/home/wdz/sdk/sysdrv/out/image_uclibc_rv1106/boot.img`

注意：单独编译kernel生成的boot.img会覆盖RNDIS配置，不建议单独烧录

## 四、全量编译报错：禁止root用户编译buildroot

### 1. 报错原因

buildroot构建系统安全限制：**不允许root账户执行configure配置脚本**

**关键完整报错日志：**

`configure: error: you should not run configure as root (set FORCE_UNSAFE_CONFIGURE=1 in environment to bypass this check)
See `config.log' for more details
` `make[1]: *** [package/pkg-generic.mk:273: /home/wdz/sdk/sysdrv/source/buildroot/buildroot-2024.02.10/output/build/host-tar-1.34/.stamp_configured] Error 1`

### 2. 解决方案

```bash
# 清理旧编译残留
cd /home/wdz/sdk
./build.sh clean

# 强制绕过root限制，全量编译
FORCE_UNSAFE_CONFIGURE=1 ./build.sh all 2>&1 | tee build.log
```

全量编译产物：`output/images/update.img`、`boot.img`，支持整板/分区烧录

## 五、RNDIS网络配置恢复（编译后网络失效修复）

单独编译kernel会覆盖USB配置，导致RNDIS虚拟网卡失效，修复步骤：

```bash
# 修改USB模式为从机模式（适配RNDIS）
vi /etc/usb_config

# 修改参数
OTG_MODE=peripheral

# 重启生效
reboot
```

重启后可通过 `ssh root@192.168.137.100` 正常连接开发板

## 六、应用层代码编译报错：u32类型未定义

### 1. 报错场景

内核代码可直接使用的`u32` 类型，**用户层应用程序无定义**，编译报错：unknown type name 'u32'

**关键完整报错日志：**

`bmp280.c:24:38: error: unknown type name 'u32'
 void bmp280_calc_data(u32 raw_press, u32 raw_temp, float *temp, float *press)
                                       ^~~
bmp280.c:48:5: error: unknown type name 'u32'
     u32 raw_p, raw_t;
     ^~~
` `bmp280.c:63:9: warning: implicit declaration of function 'bmp280_calc_data' [-Wimplicit-function-declaration]`

### 2. 解决方案

将应用层所有 `u32` 替换为标准用户层类型：**u_int32_t**

修改函数定义与变量定义：

```c
// 报错写法（内核专属）
void bmp280_calc_data(u32 raw_press, u32 raw_temp, float *temp, float *press)
u32 raw_p, raw_t;

// 正确写法（用户层通用）
void bmp280_calc_data(u_int32_t raw_press, u_int32_t raw_temp, float *temp, float *press)
u_int32_t raw_p, raw_t;
```

### 3. 应用层交叉编译命令（适配RV1106）

```bash
cd /home/wdz/test_drv
arm-rockchip830-linux-uclibcgnueabihf-gcc -static -o bmp280 bmp280.c
```

## 七、开发板运行报错：权限不足

### 1. 报错信息

`-bash: ./bmp280: Permission denied`

**关键报错说明：**交叉编译生成的可执行文件默认没有Linux可执行权限，直接运行会被系统拦截，属于板端运行最基础的权限报错。

### 2. 解决方案

```bash
# 添加可执行权限
chmod +x bmp280

# 运行测试程序
./bmp280
```

## 八、最终成功流程汇总

1. 验证BMP280硬件地址：STM32测试确认7位地址为0x77

2. 修改正确的lubancat对应DTS设备树节点，配置I2C2设备

3. 修正驱动函数签名，适配高版本内核接口规范

4. mrproper纯净清理内核，恢复内核编译配置

5. 编译内核、更新设备树、恢复RNDIS网络配置

6. 应用层替换用户层标准数据类型，交叉编译静态程序

7. 开发板赋权运行应用程序，驱动+应用整套流程调试成功

## 九、本次调试核心知识点总结（工程重点）

- **内核/用户层类型隔离**：`u32` 是内核专属类型，应用层必须使用 `u_int32_t` 等标准POSIX类型

- **内核函数签名严格匹配**：高版本内核严格校验probe、open函数参数，旧写法直接报错

- **内核编译缓存坑**：多次编译内核必须mrproper净化源码树，否则编译失败

- **设备树文件匹配**：必须对应开发板实际型号DTS，选错文件设备树不生效

- **root编译限制**：buildroot禁止root编译，需手动设置环境变量绕过

- **单独编译kernel副作用**：会覆盖USB网络配置，需手动恢复RNDIS模式




应用层程序的输出和内核驱动不一样。你用的是 `printf`，它会输出到**当前终端**。你在串口终端里运行 `./bmp280`，输出就会出现在串口终端；如果你在SSH终端里运行，输出就出现在SSH终端。
你的应用层程序里先 `open("/dev/bmp280_dev")`，然后 `read()`。如果 `read` 是阻塞的，或者驱动返回数据失败，程序就会卡在那里不动，也不会有任何输出。
在板子上执行时，加个超时或者直接按 `Ctrl+C` 终止，看有没有反应。
**确认设备节点存在**：
ls -la /dev/bmp280_dev
**确认驱动已加载**：
lsmod | grep bmp280
**用 `dmesg` 看内核日志**，确认 `probe` 是否成功、`read` 是否被调用：
dmesg | grep bmp280
**在应用层程序里加一行调试输出**，在 `main` 函数最开头加：
printf("bmp280 app started\n");
fflush(stdout);
`fflush(stdout)` 确保输出立刻刷新到终端，不会被缓冲。重新交叉编译，传上去运行，看终端有没有输出。
先执行 `ls -la /dev/bmp280_dev` 和 `dmesg | grep bmp280`，把结果发给我。