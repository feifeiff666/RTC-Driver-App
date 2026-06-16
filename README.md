# Atlas RTC Driver Modification Demo

本项目用于课程实验：基于 Atlas 200I DK A2 上的 Linux RTC 驱动文档，对 RTC 驱动做一个可验证的小修改，并提供开发板端测试应用。。

## 实验目标

- 修改 Atlas 200I DK A2 使用的 RTC 驱动 `rtc-rv8803.c`。
- 给 `probe`、`read_time`、`set_time` 路径增加可观察日志。
- 编译生成 `rtc-rv8803.ko`。
- 在开发板上加载驱动并运行用户态程序读取 `/dev/rtc0`。
- 用 Git 管理代码，开发板可以 `git clone` 后运行测试。

## 文档依据

课程文档中说明 Atlas 200I DK A2 的 RTC 芯片节点类似：

```dts
&i2c5_ao {
    rtc@32 {
        compatible = "epson,rx8900";
        reg = <0x32>;
    };
};
```

RTC 驱动路径为：

```text
drivers/rtc/rtc-rv8803.c
```

该驱动对应：

```text
Micro Crystal RV8803, Epson RX8900
```

用户态通过：

```text
/dev/rtc0
ioctl(fd, RTC_RD_TIME, ...)
```

最终会进入 RTC 子系统，再调用具体驱动的 `read_time`。

## 项目结构

```text
atlas-rtc-driver-demo/
├── README.md
├── Makefile
├── src/
│   └── rtc_check.c
├── scripts/
│   ├── patch_rtc_rv8803.py
│   ├── build_kernel_module_on_host.sh
│   ├── load_module_on_board.sh
│   └── run_on_board.sh
└── notes/
    └── report.md
```

## 在编译主机上修改并编译驱动

假设源码目录是：

```text
/opt/Ascend310B-source
```

执行：

```bash
cd atlas-rtc-driver-demo
sudo bash scripts/build_kernel_module_on_host.sh /opt/Ascend310B-source
```

脚本会做这些事：

1. 备份 `drivers/rtc/rtc-rv8803.c`。
2. 插入带有 `[atlas-rtc-demo]` 标记的调试日志。
3. 提示使用 `bash build.sh kernel` 编译 RTC 驱动。

根据课程文档，编译时需要在 menuconfig 中选择：

```text
Device Drivers
  -> Real Time Clock
    -> Micro Crystal RV8803, Epson RX8900
```

建议编译为模块：

```text
CONFIG_RTC_DRV_RV8803=m
```

编译成功后，模块通常位于：

```text
Ascend310B-source/output/kernel_modules/rtc-rv8803.ko
```

## 在开发板上运行

把项目上传到 Git 后，在 Atlas 开发板上执行：

```bash
git clone <你的仓库地址>
cd atlas-rtc-driver-demo
make
sudo bash scripts/run_on_board.sh
```

如果你已经把新的 `rtc-rv8803.ko` 上传到了开发板 `/run` 目录，可执行：

```bash
sudo bash scripts/load_module_on_board.sh /run/rtc-rv8803.ko
sudo ./rtc_check
```

## 验收命令

```bash
ls -l /dev/rtc*
lsmod | grep -E 'rtc|rv8803'
dmesg | grep atlas-rtc-demo
./rtc_check
cat /proc/driver/rtc
hwclock -r -f /dev/rtc0
```

## 预期现象

运行 `rtc_check` 后能看到类似输出：

```text
RTC device: /dev/rtc0
RTC time: 2026-06-10 15:20:30
```

查看内核日志能看到新增驱动日志：

```text
[atlas-rtc-demo] rv8803 probe: compatible rtc device detected
[atlas-rtc-demo] read_time called
```

## 和 myhello.ko 的区别

`myhello.ko` 是 dummy driver，只验证新增内核模块的编译、加载、卸载流程；它不访问硬件。

本项目修改的是实际 RTC 驱动路径。用户程序读取 `/dev/rtc0` 时，会通过 Linux RTC 子系统进入具体 RTC 驱动，因此能体现“驱动修改 + 用户态验证”的完整链路。

