
目录* [0 环境说明](https://github.com)
* [1 内核构建](https://github.com)
* [2 库编译](https://github.com)
	+ [方式1 交叉编译](https://github.com)
	+ [方式2 本地编译](https://github.com)
* [3 测试](https://github.com)
	+ [单元测试](https://github.com)
	+ [hectic:EVL 上下文切换](https://github.com)
	+ [latmus：latency测试](https://github.com)
* [4 RK3588 xenomai4实时性能](https://github.com)
* [5 总结](https://github.com)

Xenomai4自2021年首个稳定版本发布以来已经相当长一段时间，那时仅在x86架构上简单的跑了跑，之后就再没关注过。近期，我愈发好奇Xenomai 4在ARM64平台上的实时性能表现，尤其是与Xenomai 3相比如何。恰逢周末空闲，盘一盘它。


本文不算教程，而仅仅是对在ARM64（具体为RK3588平台）上构建Xenomai 4 EVL过程的一次简要记录，希望能为感兴趣的读者提供些许参考。


## 0 环境说明


目标平台如下;



> 硬件平台 NanoPi R6C/R6S/T6
> 
> 
> * SoC: Rockchip RK3588S、RK3588
> 	+ CPU: Quad\-core ARM Cortex\-A76(up to 2\.4GHz) and quad\-core Cortex\-A55 CPU (up to 1\.8GHz)
> 	+ GPU: Mali\-G610 MP4, compatible with OpenGLES 1\.1, 2\.0, and 3\.2, OpenCL up to 2\.2 and Vulkan1\.2
> 	+ VPU: 8K@60fps H.265 and VP9 decoder, 8K@30fps H.264 decoder, 4K@60fps AV1 decoder, 8K@30fps H.264 and H.265 encoder
> 	+ NPU: 6TOPs, supports INT4/INT8/INT16/FP16
> * RAM: 64\-bit 8GB LPDDR4X at 2133MHz
> * Flash: 32GB eMMC, at HS400 mode
> * Ethernet: one Native Gigabit Ethernet, and two PCIe 2\.5G Ethernet
> 
> 
> 由于瑞芯微rk35xx系列SDK基本一致，所以也适用于其他rk3568、rk3562等硬件。
> 
> 
> 桌面发行版：ubuntu 24\.04\.1 LTS (Noble Numbat)


交叉编译环境：ubuntu 24\.04 \+ rk3588\-sdk\-1\.30(有需要该SDK的私信领取)


## 1 内核构建


基于瑞芯微rk3588构建evl内核。与xenomai3一样，xenomai4由以下部分组成：


* linux内核：linux内核一般由厂商sdk提供。
* dovetail补丁：这部分从xenomai官方仓库提取。
* evl内核：这部分取自xenomai官方仓库(已包含dovetail和主线linux)。


	+ [git@git.xenomai.org](https://github.com):Xenomai/xenomai4/linux\-evl.git
	+ [https://git.xenomai.org/xenomai4/linux\-evl.git](https://github.com)
* libevl：用户空间库，供实时任务编程接口(目前还不支持posix)。


	+ [git@git.xenomai.org](https://github.com):Xenomai/xenomai4/libevl.git
	+ [https://git.xenomai.org/xenomai4/libevl.git](https://github.com)![](https://v4.xenomai.org/images/overview-build-process.png)


dovetail和evl是基于linux主线内核开发的，xenomai官方仓库已经提供了包含dovetail补丁\+evl实时内核的仓库：[https://source.denx.de/Xenomai/xenomai4/linux\-evl](https://github.com):[westworld加速器](https://xbsj9.com)。所以，**如果厂商BSP内核已经完全上游化，我们直接拉这个仓库配置、编译安装内核即可**，比如X86平台。


但是，我们使用的硬件是瑞芯微rk3588，目前瑞芯微rk3588的BSP还没有上游化，无法直接xenomai4的仓库，需要在厂商sdk linux内核上一步步添加，除linux内核外其余部分均从xenomai仓库中提取，这个方式就是xenomai4文档中说的树外构建。详细的参考xenomai4移植指南[https://v4\.xenomai.org/ports/](https://github.com)和xenomai4开发流程[https://v4\.xenomai.org/devprocess/](https://github.com).


1. 移植dovetail补丁,详见[https://v4\.xenomai.org/dovetail/porting/](https://github.com)，同样由于ARM外设的多样性，对于主线内核没有的部分dovetail需要调整，编译安装\-测试\-修改\-编译安装测试，这是一步一完善的过程。
2. dovetail补丁测试通过后，从evl仓库中对应版本提取evl内核补丁，在1的基础上打evl内核补丁，解决补丁冲突。
3. 编译配置，关于evl内核的配置说明详见[https://v4\.xenomai.org/core/build\-steps/\#core\-kconfig](https://github.com)



```
-> Kernel Features                                                                                              -> [*]EVL real-time core
     [*]   Enable quota-based scheduling 
     [*]   Enable temporal partitioning policy
     (4)     Number of partitions
     [ ]   Optimize for intra-core concurrency 
     [*]   Collect runtime statistics
     [ ]   Out-of-band networking (EXPERIMENTAL)
     Fixed sizes and limits  --->
     Pre-calibrated latency  --->
     [*]   Debug support  --->          

```

4. 编译安装内核



```
pi@NanoPi-R6S:~$demsg
...
[    2.119261] IRQ pipeline: high-priority EVL stage added.
[    2.119619] CPU6: proxy tick device registered (24.00MHz)
[    2.119625] CPU0: proxy tick device registered (24.00MHz)
[    2.119627] CPU7: proxy tick device registered (24.00MHz)
[    2.119633] CPU3: proxy tick device registered (24.00MHz)
[    2.119635] CPU4: proxy tick device registered (24.00MHz)
[    2.119638] CPU5: proxy tick device registered (24.00MHz)
[    2.119643] CPU2: proxy tick device registered (24.00MHz)
[    2.119649] CPU1: proxy tick device registered (24.00MHz)
[    2.120423] EVL: core started [DEBUG]
...
pi@NanoPi-R6S:~$ uname  -a
Linux NanoPi-R6S 6.1.57-evl #2 SMP EVL Sun Dec 15 10:22:29 CST 2024 aarch64 aarch64 aarch64 GNU/Linux

```

关于实时外设，xenomai4的实时驱动与xenomai3不同，xenomai3内部有RTDM实时驱动框架，如果需要用到实时外设需要全新开发实时驱动，而xenomai4则是在linux原始驱动上添加实时处理路径(oob)来达到实时，这样不用重写整个驱动，有好处也有坏处，坏处是，每次更新升级内核，linux驱动可能会发生一些改变，这时候实时驱动也要跟着调整才能保证原来的实时性。


确认evl内核正常启动后，编译安装evl库，然后从应用层面再跑完整的测试，确定内核各组件工作是否正常。


## 2 库编译


构建libevl参考[官方文档](https://github.com)，以下基于官方文档操作。


`libevl`使用[meson](https://github.com)构建系统。[Meson 项目](https://github.com)为每个阶段（从编写构建规则到使用构建规则）发布了一份写得很好且有用的文档，ubuntu使用如下命令安装：



```
$ sudo apt install meson
$ pip3 install  ninja --break-system-packages

```

`meson`基于通常的*configure*、*compile*和*install*步骤，启用通用的构建和安装过程。这些步骤始终在*单独的*构建树中进行，并由 强制执行`meson`（换句话说，不要尝试直接从`libevl`源代码树构建 \- 这样做行不通，此外，这样做是个坏主意TM）。


首先，我们需要配置构建树`libevl`。通用语法如下：



```
$ meson setup [--cross-file ] [-Duapi=] [-Dbuildtype=] [-Dprefix=] $buildir $srcdir

```



| 参数变量 | 描述 |
| --- | --- |
| $buildir | 构建目录的路径（与 $srcdir 分开） |
| $srcdir | `libevl`源树的路径 |
|  | 安装信息前缀（安装路径为DESTDIR/DESTDIR/prefix） |
|  | 构建类型，例如*debug*、*debugoptimized*或*release* |
|  | 一个`meson` [交叉编译文件](https://github.com)定义构建环境 |
|  | 包含以下内容的内核源代码树的路径[UAPI 标头](https://github.com) |


### 方式1 交叉编译


建立一个`meson`交叉配置文件`rk3588-aarch64-linux-gnu.txt`



```
[binaries]
c = 'aarch64-linux-gnu-gcc'
cpp = 'aarch64-linux-gnu-g++'
ar = 'aarch64-linux-gnu-ar'
strip = 'aarch64-linux-gnu-strip'
pkgconfig = '/usr/bin/pkg-config'

[host_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'aarch64'
endian = 'little'

[target_machine]
system = 'linux'
cpu_family = 'aarch64'
cpu = 'armv8a'
endian = 'little'

[properties]
platform = 'generic_aarch64'
needs_exe_wrapper = false
#sys_root = '/opt/FriendlyARM/toolchain/11.3-aarch64/aarch64-cortexa53-linux-gnu/sysroot'

[built-in options]
#c_args = []
#c_link_args = []
#cpp_args = []
#cpp_link_args = []

```

配置



```
$ mkdir /tmp/build-rk3588 && cd /tmp/build-rk3588
$ meson setup --cross-file /work/github/xenomai4/libevl/rk3588-aarch64-linux-gnu.txt -Dbuildtype=release -Dprefix=/usr -Duapi=/work/github/rk3588/kernel-rockchip . /work/github/xenomai4/libevl

```


> /work/github/xenomai4/libevl：是libevl源码所在的位置
> 
> 
> /work/github/xenomai4/kernel\-rockchip：是内核源码所在位置
> 
> 
> \-Dprefix\=/usr：安装到该目录下，不然后面要配置环境变量，比较麻烦


编译安装



```
# Build the whole thing
$ meson compile

# Eventually, install the result
$ mkdir /tmp/rk3588-rootfs
$ DESTDIR=/tmp/rk3588-rootfs ninja install

```

如果编译中出现错误`evl-net.c:17:10: fatal error: bpf/bpf.h: No such file or directory`和`libbpf.so`,这是交叉编译环境中没有没有内核bpf相关环境，修改`libevl/utils/meson.build`,将`evl-net`部分注释，不编译即可。


### 方式2 本地编译


在rk3588上编译



```
# 安装编译工具
$ sudo apt install meson
$ pip3 install  ninja --break-system-packages
$ git clone  https://source.denx.de/Xenomai/xenomai4/libevl.git

# Prepare the build directory
$ mkdir /tmp/build-native && cd /tmp/build-native
$ meson setup -Dbuildtype=release -Dprefix=/usr -Duapi=~/kernel-rockchip . ~/libevl

# Build it
$ meson compile

# Install the result
$ ninja install

```


> \~/kernel\-rockchip:为内核源码路径


如果要将libevl编译生成debian安装包，可以参考[https://blog.devgenius.io/how\-to\-build\-debian\-packages\-from\-meson\-ninja\-d1c28b60e709](https://github.com)


现在已经将libevl安装到\`/usr/下



```
pi@NanoPi-R6S:~$tree /usr/
/usr/
|-- bin
|   |-- evl
|   |-- hectic
|   |-- latmus
|   |-- oob-net-icmp
|   |-- oob-net-udp
|   `-- oob-spi
|-- include
|   `-- evl
|       |-- atomic.h
|       ...
|       |-- timer.h
|       `-- xbuf.h
|-- lib
|   `-- aarch64-linux-gnu
|       |-- libevl.a
|       |-- libevl.so -> libevl.so.4
|       |-- libevl.so.4 -> libevl.so.4.0.0
|       |-- libevl.so.4.0.0
|       `-- pkgconfig
|           `-- evl.pc
`-- libexec
    `-- evl
        |-- evl-check
        |-- evl-gdb
        |-- evl-help
        |-- evl-net
        |-- evl-ps
		...
        |   `-- thread-mode-bits
        |-- trace.evl
        |-- trace.irq
        `-- trace.timer

```

## 3 测试


EVL 附带了一系列测试，运行这些测试来确保内核在目标系统上正常运行，测试evl参考[官方说明](https://github.com)。


### 单元测试


作为构建`libevl`的一部分，在`$prefix/tests`中生成了一系列单元测试程序。您应该运行它们中的每一个以确保一切正常。最简单的方法如下：



```
pi@NanoPi-R6S:/tmp/build-native$ sudo evl test
basic-xbuf: OK
clock-timer-periodic: OK
clone-fork-exec: OK
detach-self: OK
duplicate-element: OK
element-visibility: OK
fault: OK
fpu-preload: OK
fpu-stress: OK
heap-torture: OK
mapfd: OK
....
ring-spray: OK
rwlock-read: OK
rwlock-write: OK
sched-quota-accuracy: 100.0%
sched-tp-accuracy: OK
sched-tp-overrun: OK
sem-close-unblock: OK
sem-flush: OK
sem-timedwait: OK
sem-wait: OK
simple-clone: OK
stax-lock: OK
stax-warn: OK
thread-mode-bits: OK

```

![Screenshot from 2024-12-15 10-41-56](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/Screenshot%20from%202024-12-15%2010-41-56.png)


### hectic:EVL 上下文切换


默认情况下， `hectic`程序会在用户空间和内核空间中运行大量 EVL 线程，以锻炼自治核心的调度程序。此外，该测试还可以特别强调浮点管理代码，以确保 FPU 在带外和带内线程上下文之间完美共享。



> 要运行此测试，需要在内核配置中启用`CONFIG_EVL_HECTIC` 。



```
pi@NanoPi-R6S:/tmp/build-native$ sudo hectic -s 200
== Testing FPU check routines...
== FPU check routines: OK.
== Threads: switcher_ufps0-0 rtk0-1 rtk0-2 rtup0-3 rtup0-4 rtup_ufpp0-5 rtup_ufpp0-6 rtus0-7 rtus0-8 rtus_ufps0-9 rtus_ufps0-10 rtuo0-11 rtuo0-12 rtuo_ufpp0-13 rtuo_ufpp0-14 rtuo_ufps0-15 rtuo_ufps0-16 rtuo_ufpp_ufps0-17 rtuo_ufpp_ufps0-18 fpu_stress_ufps0-19 switcher_ufps1-0 rtk1-1 rtk1-2 rtup1-3 rtup1-4 rtup_ufpp1-5 rtup_ufpp1-6 rtus1-7 rtus1-8 rtus_ufps1-9 rtus_ufps1-10 rtuo1-11 rtuo1-12 rtuo_ufpp1-13 rtuo_ufpp1-14 rtuo_ufps1-15 rtuo_ufps1-16 rtuo_ufpp_ufps1-17 rtuo_ufpp_ufps1-18 fpu_stress_ufps1-19 switcher_ufps2-0 rtk2-1 rtk2-2 rtup2-3 rtup2-4 rtup_ufpp2-5 rtup_ufpp2-6 rtus2-7 rtus2-8 rtus_ufps2-9 rtus_ufps2-10 rtuo2-11 rtuo2-12 rtuo_ufpp2-13 rtuo_ufpp2-14 rtuo_ufps2-15 rtuo_ufps2-16 rtuo_ufpp_ufps2-17 rtuo_ufpp_ufps2-18 fpu_stress_ufps2-19 switcher_ufps3-0 rtk3-1 rtk3-2 rtup3-3 rtup3-4 rtup_ufpp3-5 rtup_ufpp3-6 rtus3-7 rtus3-8 rtus_ufps3-9 rtus_ufps3-10 rtuo3-11 rtuo3-12 rtuo_ufpp3-13 rtuo_ufpp3-14 rtuo_ufps3-15 rtuo_ufps3-16 rtuo_ufpp_ufps3-17 rtuo_ufpp_ufps3-18 fpu_stress_ufps3-19 switcher_ufps4-0 rtk4-1 rtk4-2 rtup4-3 rtup4-4 rtup_ufpp4-5 rtup_ufpp4-6 rtus4-7 rtus4-8 rtus_ufps4-9 rtus_ufps4-10 rtuo4-11 rtuo4-12 rtuo_ufpp4-13 rtuo_ufpp4-14 rtuo_ufps4-15 rtuo_ufps4-16 rtuo_ufpp_ufps4-17 rtuo_ufpp_ufps4-18 fpu_stress_ufps4-19 switcher_ufps5-0 rtk5-1 rtk5-2 rtup5-3 rtup5-4 rtup_ufpp5-5 rtup_ufpp5-6 rtus5-7 rtus5-8 rtus_ufps5-9 rtus_ufps5-10 rtuo5-11 rtuo5-12 rtuo_ufpp5-13 rtuo_ufpp5-14 rtuo_ufps5-15 rtuo_ufps5-16 rtuo_ufpp_ufps5-17 rtuo_ufpp_ufps5-18 fpu_stress_ufps5-19 switcher_ufps6-0 rtk6-1 rtk6-2 rtup6-3 rtup6-4 rtup_ufpp6-5 rtup_ufpp6-6 rtus6-7 rtus6-8 rtus_ufps6-9 rtus_ufps6-10 rtuo6-11 rtuo6-12 rtuo_ufpp6-13 rtuo_ufpp6-14 rtuo_ufps6-15 rtuo_ufps6-16 rtuo_ufpp_ufps6-17 rtuo_ufpp_ufps6-18 fpu_stress_ufps6-19 switcher_ufps7-0 rtk7-1 rtk7-2 rtup7-3 rtup7-4 rtup_ufpp7-5 rtup_ufpp7-6 rtus7-7 rtus7-8 rtus_ufps7-9 rtus_ufps7-10 rtuo7-11 rtuo7-12 rtuo_ufpp7-13 rtuo_ufpp7-14 rtuo_ufps7-15 rtuo_ufps7-16 rtuo_ufpp_ufps7-17 rtuo_ufpp_ufps7-18 fpu_stress_ufps7-19
RTT|  00:00:01
RTH|---------cpu|ctx switches|-------total
RTD|           4|        4672|        4672
RTD|           5|        4672|        4672
RTD|           1|        4501|        4501
RTD|           3|        4501|        4501
RTD|           0|        4501|        4501
RTD|           2|        4501|        4501
RTD|           6|        4729|        4729
RTD|           7|        4729|        4729
RTD|           4|        4674|        9346
RTD|           5|        4674|        9346
RTD|           7|        4676|        9405
RTD|           6|        4676|        9405
RTD|           0|        4503|        9004
RTD|           1|        4503|        9004
RTD|           3|        4503|        9004
RTD|           2|        4503|        9004
RTD|           4|        4674|       14020
RTD|           5|        4674|       14020
RTD|           7|        4674|       14079
RTD|           6|        4674|       14079
RTD|           2|        4448|       13452
RTT|  00:00:03
RTH|---------cpu|ctx switches|-------total
RTD|           3|        4450|       13454
RTD|           0|        4560|       13564
RTD|           1|        4503|       13507
RTD|           5|        4678|       18698
RTD|           7|        4672|       18751
RTD|           4|        4731|       18751
RTD|           6|        4729|       18808
RTD|           2|        4501|       17953
RTD|           3|        4499|       17953
RTD|           0|        4560|       18124
RTD|           1|        4503|       18010
RTD|           5|        4727|       23425
RTD|           4|        4678|       23429
RTD|           7|        4731|       23482
RTD|           6|        4731|       23539
RTD|           2|        4503|       22456
RTD|           3|        4503|       22456
RTD|           0|        4560|       22684
RTD|           1|        4503|       22513
RTD|           5|        4674|       28099
RTD|           4|        4670|       28099
....

```

### latmus：latency测试



> 如果您计划测量目标系统上最坏情况下的延迟，则应在此类系统上运行[evl check](https://github.com)命令，以便及早检测内核的任何明显错误配置，该检查需要内核配置中启用`CONFIG_IKCONFIG_PROC`。


* **`latmus -t` 校准gravity**



```
pi@NanoPi-R6S:/tmp/build-native$ sudo latmus -t
== latmus started for core tuning, period=1000 microseconds (may take a while)
irq gravity...500 ns
kernel gravity...1000 ns
user gravity...1500 ns
== tuning completed after 21s

```
* *\-i \-\-irq*


从内核中断处理程序的上下文中收集延迟数据（测试irq响应时间）。


* *\-k \-\-kernel*


从基于内核的 EVL 线程的上下文中收集延迟数据或调整 EVL 核心计时器（测试evl内核线程响应时间）。


* *\-u \-\-user*


从用户空间中运行的 EVL 线程的上下文中收集延迟数据或调整 EVL 核心计时器。默认模式（测试evl用户线程响应时间）。


* *\-s \-\-sirq*


测量从带外阶段发出[合成中断的](https://github.com)时刻与带内处理程序最终接收该中断之间的延迟。当在巨大的工作负载压力下测量时，这给出了**带内内核**由于本地中断禁用（即*停止*带内管道阶段）而经历的最坏情况中断延迟。因此，这与 EVL 应用从带外阶段观察到的更短且有限的中断延迟无关。


其他参数详见`--help`输出和[latmus参数说明](https://github.com)



```
pi@NanoPi-R6S:/tmp/build-native$ sudo latmus --help
usage: latmus [options]:
-m --measure            measure latency on timer event [default]
-t --tune               tune the EVL core timer
-i --irq                measure/tune interrupt latency
-k --kernel             measure/tune kernel scheduling latency
-u --user               measure/tune user scheduling latency
    [ if none of --irq, --kernel or --user is given,
      tune for all contexts ]
-s --sirq               measure in-band response time to synthetic irq
-p --period=        sampling period
-P --priority=    responder thread priority [=90]
-c --cpu=            pin responder thread to CPU [=current]
-C --force-cpu=      similar to -c, accept non-isolated CPU
-r --reset              reset core timer gravity to factory default
-b --background         run in the background (daemon mode)
-K --keep-going         keep going on unexpected switch to in-band mode
-A --max-abort=     abort if maximum latency exceeds threshold
-T --timeout=[dhms]  stop measurement after  [d(ays)|h(ours)|m(inutes)|s(econds)]
-v --verbose[=level]    set verbosity level [=1]
-q --quiet              quiet mode (i.e. --verbose=0)
-l --lines=        result lines per page, 0 = no pagination [=21]
-H --histogram[=]   set histogram size to  cells [=200]
-g --plot=    dump histogram data to file (gnuplot format)
-Z --oob-gpio=    measure EVL response time to GPIO event via 
-z --inband-gpio= measure in-band response time to GPIO event via 
-I --gpio-in=     input GPIO line configuration
   with  = gpiochip-devname,pin-number[,rising-edge|falling-edge]
-O --gpio-out=    output GPIO line configuration
   with  = gpiochip-devname,pin-number

pi@NanoPi-R6S:/tmp/build-native$ sudo latmus
warming up on CPU4...
RTT|  00:00:01  (user, 1000 us period, priority 98, CPU4)
RTH|----lat min|----lat avg|----lat max|-overrun|---msw|---lat best|--lat worst
RTD|      0.531|      1.041|      1.403|       0|     0|      0.531|      1.403
RTD|      0.528|      1.027|      2.871|       0|     0|      0.528|      2.871
RTD|      0.528|      1.013|      1.431|       0|     0|      0.528|      2.871
RTD|      0.531|      1.003|      1.402|       0|     0|      0.528|      2.871
RTD|      0.530|      0.997|      1.403|       0|     0|      0.528|      2.871
....

```

## 4 RK3588 xenomai4实时性能


这里还是老方式，内存和cpu压力使用`stress -m 16 -c 16`来产生。


图形压力启用4个终端运行`glmark2-es2-wayland`产生。



```
while true; do glmark2-es2-wayland; done

```

![Screenshot from 2024-12-15 10-35-40](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/Screenshot%20from%202024-12-15%2010-35-40.png)


测试后结果如下：



```
pi@NanoPi-R6S:/tmp/build-native$ sudo latmus -t
pi@NanoPi-R6S:/tmp/build-native$ sudo latmus --plot=evl-latency.txt
pi@NanoPi-R6S:/tmp/build-native$cat evl-latency.txt
# test started on: Sun Dec 15 08:34:03 2024
# Linux version 6.1.57-evl (wsg1100@wsg1100-virtual-machine) (aarch64-linux-gnu-gcc (ctng-1.25.0-119g-FA) 11.3.0, GNU ld (GNU Binutils) 2.38) #2 SMP EVL Sun Dec 15 10:22:29 CST 2024
# libevl version: evl.0.50 -- #c39165f (2024-09-25 09:25:01 +0200)
# sampling period: 1000 microseconds
# clock gravity: 500i 1000k 1500u
# clocksource: arch_sys_counter
# vDSO access: architected
# context: user
# thread priority: 98
# thread affinity: CPU4
# duration (hhmmss): 02:01:50
# peak (hhmmss): 01:52:07
# min latency: 0.432
# avg latency: 1.012
# max latency: 8.970
# sample count: 7309414
0 3560945
1 3741581
2 5998
3 745
4 99
5 33
6 9
7 3
8 1

```

由于时间关系，仅跑了2小时(主要是怕板子没散热挂了，实在太烫了)，这latency分布，不得不说，相比xenomai3稳得一批。


![微信图片_20241215185229](https://wsg-blogs-pic.oss-cn-beijing.aliyuncs.com/xenomai/%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20241215185229-1734260054782.jpg)


## 5 总结


通过对xenomai4初步测试，从latency数据上看，比xenomai3好一些，波动不频繁，虽然xenomai3已经很好了（满载2周 了latency≤15us）。


目前，Xenomai 4面临的主要问题是其编程API接口与Xenomai 3不兼容，同时尚未支持POSIX接口。若能实现POSIX接口的支持，将有助于构建更为完善的生态系统，并简化从Xenomai 3到Xenomai 4的迁移过程。所以，Xenomai 4目前的受众群体相对较小。


xenomai4与xenomai3工作原理一致，除API接口不同外其他注意事项基本一致。笔者目前还广泛使用xenomai4，若后续实际项目上用到了，再更新一些关于xenomai4的相关文章，敬请关注！


