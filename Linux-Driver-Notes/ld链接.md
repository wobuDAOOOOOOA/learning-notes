**对，这是你这个板子的固定格式，但只是你当前项目的固定格式。** 换一块不同的芯片（比如全志、NXP），交叉编译前缀就会变。ARM只是个大的架构名，具体的指令集版本和依赖库都不一样。

---

## 为什么这两项要写死

- `ARCH=arm`：告诉内核编译系统，目标平台是ARM架构。内核源码支持几十种CPU架构，它要知道该用哪一套汇编代码和头文件。
- `CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf-`：指定工具链的前缀。内核Makefile实际会调用 `${CROSS_COMPILE}gcc`、`${CROSS_COMPILE}ld`，这个前缀就是用来拼出完整工具名的。

你用的是瑞芯微RV1106，厂家专门适配了这个前缀的工具链。换一块板子前缀就变了。

---

## 可以在Makefile里用export

直接写在Makefile里更省事，每次不用手动敲：

```makefile
export ARCH := arm
export CROSS_COMPILE := arm-rockchip830-linux-uclibcgnueabihf-

obj-m += hello_drv.o

KDIR := /home/wdz/sdk/sysdrv/source/kernel
PWD  := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

`export` 写在Makefile里，会把这些变量传递给子进程。你执行 `make` 时，Makefile里调用内核的 `$(MAKE)`，内核的Makefile会自动继承这两个变量。效果和手动敲完全一样。

---

## 但PATH还是要在终端里设

Makefile里一般只设 `ARCH` 和 `CROSS_COMPILE`，不设 `PATH`。因为 `PATH` 是shell级的，不是编译系统的变量，而且写死路径会让Makefile在别人电脑上用不了。一劳永逸的方法是把 `export PATH=...` 写到 `~/.bashrc`，以后每次开终端自动加载。把这三行加到文件末尾：

```bash
export PATH=/opt/arm-rockchip830-linux-uclibcgnueabihf/bin:$PATH
export ARCH=arm
export CROSS_COMPILE=arm-rockchip830-linux-uclibcgnueabihf-
```

保存后执行 `source ~/.bashrc`，以后每个新终端都能直接用。