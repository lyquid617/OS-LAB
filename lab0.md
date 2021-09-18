# Lab 0: RV64 内核调试

## 1 实验目的

安装虚拟机及Docker，通过在QEMU模拟器上运行Linux来熟悉如何从源代码开始将内核运行在QEMU模拟器上，学习使用GDB跟QEMU对代码进行联合调试，为后续实验打下基础。

## 2 实验内容及要求

* 安装虚拟机软件、Ubuntu镜像，自行学习Linux基础命令。
* 安装Docker，下载并导入Docker镜像，创建并运行容器。

* 编译内核并用 gdb + QEMU 调试，在内核初始化过程中设置断点，对内核的启动过程进行跟踪，并尝试使用gdb的各项命令。

请各位同学独立完成实验，任何抄袭行为都将使本次实验判为0分。请查看文档尾部附录部分的背景知识介绍，**跟随实验步骤完成实验并以截图的方式记录实验过程**，如有需要请对每一步的命令以及结果进行必要的解释。

最终提交的实验报告中**请删除附录部分**，并命名为“**学号_姓名\_lab0.pdf**"，以pdf格式上传至学在浙大平台。

## 3 操作方法和实验步骤

### 3.1 通过虚拟机安装Linux系统(20%)

请参考【附录A.LINUX 基础】了解相关背景知识。如果同学先前已经安装过Linux系统，可跳过该步骤（分数照给）。

##### **1.下载并安装虚拟机软件**

 VMware Workstation Player：[VMware Workstation Player - VMware Customer Connect](https://customerconnect.vmware.com/cn/downloads/info/slug/desktop_end_user_computing/vmware_workstation_player/16_0#product_downloads)

##### **2.下载Ubuntu镜像**

建议下载Ubuntu18.04/ubuntu-18.04.5-desktop-amd64.iso：[Index of /ubuntu-releases/18.04/ (zju.edu.cn)](https://mirrors.zju.edu.cn/ubuntu-releases/18.04/)

##### **3.通过虚拟机软件安装Ubuntu镜像**

自行搜索相应教程进行安装，建议将用户名设置为姓名缩写，**注意将硬盘大小设置到30G以上方便后续实验**。

##### **4.自行学习Linux命令行的使用，了解基本命令**

推荐资源：

* [课程概览与 shell · the missing semester of your cs education (missing-semester-cn.github.io)](https://missing-semester-cn.github.io/2020/course-shell/)（这门课程的其他小节也建议大家学习一下）
* [TLCL (billie66.github.io)](http://billie66.github.io/TLCL/book/index.html)（建议阅读第二~五章、第十二章、十三章相关内容）

请依照第一行的例子，解释以下命令并附上运行截图（推荐截图工具：[Snipaste](https://www.snipaste.com)），适当缩放图片大小。

| 命令                    | 作用                     | 截图                                                         |
| ----------------------- | ------------------------ | ------------------------------------------------------------ |
| pwd                     | 打印出当前工作目录的名称 | ![image-20210913102609974](https://raw.githubusercontent.com/HHtttterica/Pics/main/image-20210913102609974.png) |
| ls                      |                          |                                                              |
| ls -al                  |                          |                                                              |
| cd ~                    |                          |                                                              |
| mkdir oslab             |                          |                                                              |
| vi test.c               |                          |                                                              |
| gedit test.c            |                          |                                                              |
| rm test.c               |                          |                                                              |
| sudo apt  install curl  |                          |                                                              |
| touch a.txt             |                          |                                                              |
| cat a.txt \| tail -n 10 |                          |                                                              |

### 3.2 安装 Docker 环境并创建容器(20%)

请参考【附录B.Docker使用基础】了解相关背景知识。

通常情况下，`$` 提示符表示当前运行的用户为普通用户，`#` 代表当前运行的用户为特权用户。但在下文的示例中，以 `###` 开头的行代表注释，`$` 开头的行代表在虚拟机上运行的命令，`#` 开头的行代表在 `docker` 中运行的命令，`(gdb)` 开头的行代表在 `gdb` 中运行的命令。

**在执行每一条命令前，请你对将要进行的操作进行思考，给出的命令不需要全部执行，并且不是所有的命令都可以无条件执行，请不要直接复制粘贴命令去执行。**

##### **1.安装docker**

```shell
### 使用官方安装脚本自动安装docker
$ curl -fsSL https://get.docker.com | bash -s docker --mirror Aliyun

### 将用户加入docker组，免 sudo
$ sudo usermod -aG docker $USER   ### 注销后重新登陆生效
```

**【请将该步骤中命令的输入及运行结果截图附在此处，示例如下。接下来每一步同理，不作赘述，】**

![image-20210913102636356](https://raw.githubusercontent.com/HHtttterica/Pics/main/image-20210913102636356.png)





##### 2.下载并导入docker镜像

为便于开展实验，我们在[docker镜像](https://pan.zju.edu.cn/share/d10c6d076e60a89964df83a178)中提前安装好了实验所需的环境（RISC-V工具链、QEMU模拟器），并设置好了`$PATH`变量以直接访问`qemu-system-riscv64`等程序。请下载该docker镜像文件并将其传至虚拟机中（可以直接拖拽到文件夹中），输入命令导入镜像。

```bash
### 首先进入oslab.tar所在的文件夹，然后使用该命令导入docker镜像
$ cat oslab.tar | docker import - oslab:2020
### 执行命令后若出现以下错误提示
### ERROR: Got permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock
### 可以使用下面命令为该文件添加权限来解决
### $ sudo chmod a+rw /var/run/docker.sock

### 查看docker镜像
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
oslab               2020                d7046ea68221        5 seconds ago       2.89G
```



 

##### 3.从镜像中创建一个容器并进入该容器

```shell
### 从镜像创建一个容器
$ docker run -it oslab:2020 /bin/bash    ### -i:交互式操作 -t:终端
root@368c4cc44221:/#                     ### 提示符变为 '#' 表明成功进入容器 后面的字符串根据容器而生成，为容器id
root@368c4cc44221:/# exit (或者CTRL+D）   ### 从容器中退出 此时运行docker ps，运行容器的列表为空

### 查看当前运行的容器
$ docker ps 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
### 查看所有存在的容器
$ docker ps -a 
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
368c4cc44221        oslab:2020          "/bin/bash"         54 seconds ago      Exited (0) 30 seconds ago                       relaxed_agnesi

### 启动处于停止状态的容器
$ docker start 368c     ### 368c 为容器id的前四位，id开头的几位便可标识一个容器
$ docker ps             ### 可看到容器已经启动
CONTAINER ID        IMAGE               COMMAND             CREATED              STATUS              PORTS               NAMES
368c4cc44221        oslab:2020          "/bin/bash"         About a minute ago   Up 16 seconds                           relaxed_agnesi

### 进入已经运行的容器 oslab的密码为2020
$ docker exec -it -u oslab -w /home/oslab 36 /bin/bash
oslab@368c4cc44221:~$
```



**请解释这两行命令：**

* `docker run -it oslab:2020 /bin/bash`

  答：`docker run`指创建一个新的容器并运行一个命令，`-it`指交互模式，`oslab:2020`指定了所使用的镜像，`/bin/bash`指在容器内执行`/bin/bash`命令。

* `docker exec -it -u oslab -w /home/oslab 36 /bin/bash`

  答：

  

### 3.3 编译 linux 内核(20%)

请参考【附录E.LINUX 内核编译基础】了解相关背景知识。

再次强调，以 `###` 开头的行代表注释，`$` 开头的行代表在虚拟机上运行的命令，**`#` 开头的行代表在 `docker` 中运行的命令，请勿将`#`一并复制入命令行中执行。**

```shell
### 进入实验目录并设置环境变量
# pwd
/home/oslab
# cd lab0
# export TOP=`pwd`
# mkdir -p build/linux
# make -C linux O=$TOP/build/linux \
          CROSS_COMPILE=riscv64-unknown-linux-gnu- \
          ARCH=riscv CONFIG_DEBUG_INFO=y \
          defconfig all -j$(nproc)

### 编译内核需要一定时间，请耐心等待。
```



### 3.4 使用QEMU运行内核(20%)

请参考【附录C.QEMU使用基础】了解相关背景知识。

**注意，QEMU的退出方式较为特殊，需要先按住`ctrl+a`，放开后再按一次`x`。**

```shell
### 用户名root，没有密码
# qemu-system-riscv64 -nographic -machine virt -kernel build/linux/arch/riscv/boot/Image  \
 -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0"   \
 -bios default -drive file=rootfs.ext4,format=raw,id=hd0 \
 -netdev user,id=net0 -device virtio-net-device,netdev=net0
```



### 3.5 使用 gdb 调试内核(20%)

请参考【附录D.GDB使用基础】了解相关背景知识。学会调试将在后续实验中为你提供帮助，推荐同学们跟随[GDB调试入门指南](https://zhuanlan.zhihu.com/p/74897601)教程完成相应基础练习，熟悉gdb调试的使用。

```shell
### Terminal 1
# qemu-system-riscv64 -nographic -machine virt -kernel build/linux/arch/riscv/boot/Image  \
 -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0"   \
 -bios default -drive file=rootfs.ext4,format=raw,id=hd0 \
 -netdev user,id=net0 -device virtio-net-device,netdev=net0 -S -s

```

再打开一个终端窗口，从宿主机进入该容器，并切换到`lab0`文件夹，进入gdb调试。

```shell
###Terminal2
# riscv64-unknown-linux-gnu-gdb build/linux/vmlinux
```

 顺序执行下列gdb命令，写出每条命令的含义并附上执行结果的截图。

```shell
(gdb) target remote localhost:1234
```

* 含义：target remote 命令表示远程调试，而1234是默认的用于调试连接的端口号。

* 执行结果：

  ![image-20210913102959055](https://raw.githubusercontent.com/HHtttterica/Pics/main/image-20210913102959055.png)

```shell
(gdb) b start_kernel 
(gdb) b *0x80000000
(gdb) b *0x80200000
(gdb) info breakpoints
(gdb) delete 2
(gdb) info breakpoints
```

* 含义：
* 执行结果：



``` 
(gdb) continue
(gdb) delete 3
(gdb) continue
(gdb) step
(gdb) s
(gdb) (不做输入，直接回车)
(gdb) next
(gdb) n
(gdb) (不做输入，直接回车)
```

* 含义：

* 执行结果：

  

```shell
(gdb) disassemble
(gdb) nexti
(gdb) n
(gdb) stepi
(gdb) s
```

* 含义：

* 执行结果：

  

* **请回答：nexti和next的区别在哪里？stepi和step的区别在哪里？next和step的区别是什么？**

  答：

```shell
(gdb) continue
#由于此时无断点，continue将一直执行下去，为了退出gdb，可以：①在gdb中按住ctrl+c退出当前正在运行的gdb命令，然后再用quit退出②在qemu中先按ctrl+a，松开后再按x先退出qemu，然后在gdb中quit退出。
(gdb) quit
```

* 含义： 
* 执行结果：

 

## 4 讨论和心得

请在此处填写实验过程中遇到的问题及相应的解决方式。

由于本实验为新实验，可能存在不足之处，欢迎同学们对本实验提出建议。

## 附录

### A.Linux 使用基础

#### Linux简介

Linux 是一套免费使用和自由传播的类 Unix 操作系统，是一个基于 POSIX 和 UNIX 的多用户、多任务、支持多线程和多 CPU 的操作系统。 

在Linux环境下，人们通常使用命令行接口来完成与计算机的交互。终端（Terminal）是用于处理该过程的一个应用程序，通过终端你可以运行各种程序以及在自己的计算机上处理文件。在类Unix的操作系统上，终端可以为你完成一切你所需要的操作。我们仅对实验中涉及的一些概念进行介绍，你可以通过下面的链接来对命令行的使用进行学习：

1. [The Missing Semester of Your CS Education](https://missing-semester-cn.github.io/2020/shell-tools)[&gt;&gt;Video&lt;&lt;](https://www.bilibili.com/video/BV1x7411H7wa?p=2)
2. [GNU/Linux Command-Line Tools Summary](https://tldp.org/LDP/GNU-Linux-Tools-Summary/html/index.html)
3. [Basics of UNIX](https://github.com/berkeley-scf/tutorial-unix-basics)

#### Linux中的环境变量

当我们在终端输入命令时，终端会找到对应的程序来运行。我们可以通过 `which`命令来做一些小的实验：

```bash
$ which gcc
/usr/bin/gcc
$ ls -l /usr/bin/gcc
lrwxrwxrwx 1 root root 5 5月  21  2019 /usr/bin/gcc -> gcc-7
```

可以看到，当我们在输入 `gcc`命令时，终端实际执行的程序是 `/usr/bin/gcc`。实际上，终端在执行命令时，会从 `PATH`环境变量所包含的地址中查找对应的程序来执行。我们可以将 `PATH`变量打印出来来检查一下其是否包含 `/usr/bin`。

```bash
$ echo $PATH
/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/home/phantom/.local/bin
```

如果你想直接访问 `riscv64-unknown-linux-gnu-gcc`、`qemu-system-riscv64`等程序，那么你需要把他们所在的目录添加到目录中，镜像中已完成了这步操作，可在相应容器中使用`echo $PATH`命令检查是否包含`/opt/riscv/bin`目录。

```bash
$ export PATH=$PATH:/opt/riscv/bin
```

### B. Docker 使用基础

#### Docker 基本介绍

在生产开发环境中，常常会遇到应用程序和系统环境变量以及一些依赖的库文件不匹配，导致应用无法正常运行的问题，因此出现了Docker。Docker 是一种利用容器（container）来进行创建、部署和运行应用的工具。Docker把一个应用程序运行需要的二进制文件、运行需要的库以及其他依赖文件打包为一个包（package），然后通过该包创建容器并运行，由此被打包的应用便成功运行在了Docker容器中。

比如，你在本地用Python开发网站后台，开发测试完成后，就可以将Python3及其依赖包、Flask及其各种插件、Mysql、Nginx等打包到一个容器中，然后部署到任意你想部署到的环境。

Docker中有三个重要概念：

1. 镜像（Image）：类似于虚拟机中的镜像，是一个包含有文件系统的面向Docker引擎的只读模板。任何应用程序运行都需要环境，而镜像就是用来提供这种运行环境的。
2. 容器（Container）：Docker引擎利用容器来运行、隔离各个应用。**容器是镜像创建的应用实例**，可以创建、启动、停止、删除容器，各个容器之间是是相互隔离的，互不影响。镜像本身是只读的，容器从镜像启动时，Docker在镜像的上层创建一个可写层，镜像本身不变。
3. 仓库（Repository）：镜像仓库是Docker用来集中存放镜像文件的地方，一般每个仓库存放一类镜像，每个镜像利用tag进行区分，比如Ubuntu仓库存放有多个版本（12.04、14.04等）的Ubuntu镜像。

#### Docker基本命令

实验中主要需要了解docker中容器生命周期管理及容器操作的相关命令及选项，可以通过[Docker 命令大全 | 菜鸟教程 (runoob.com)](https://www.runoob.com/docker/docker-command-manual.html)进行初步了解。

```shell
#列出所有镜像
docker image ls

#创建/启动容器
docker run -it oslab:2020 /bin/bash    
#启动容器时，使用-v参数指定挂载宿主机目录，启动一个centos容器将宿主机的/test目录挂载到容器的/soft目录
docker run -it -v /test:/soft centos /bin/bash
#进入运行中的容器，执行bash:
docker exec -it 9df70f9a0714 /bin/bash
#停止容器
docker stop ID
#启动处于停止状态的容器
docker start ID

#重命名容器
docker rename oldname newname
#列出所有的容器
docker ps -a
#删除容器
docker rm ID
```

### C. QEMU 使用基础

#### 什么是QEMU

QEMU最开始是由法国程序员Fabrice Bellard等人开发的可执行硬件虚拟化的开源托管虚拟机。

QEMU主要有两种运行模式。**用户模式下，QEMU能够将一个平台上编译的二进制文件在另一个不同的平台上运行。**如一个ARM指令集的二进制程序，通过QEMU的TCG（Tiny Code Generator）引擎的处理之后，ARM指令被转化为TCG中间代码，然后再转化为目标平台（如Intel x86）的代码。

**系统模式下，QEMU能够模拟一个完整的计算机系统，该虚拟机有自己的虚拟CPU、芯片组、虚拟内存以及各种虚拟外部设备。**它使得为跨平台编写的程序进行测试及除错工作变得容易。

#### 如何使用 QEMU（常见参数介绍）

以该命令为例，我们简单介绍QEMU的参数所代表的含义

```bash
$ qemu-system-riscv64 -nographic -machine virt -kernel build/linux/arch/riscv/boot/Image  \
 -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0"   \
 -bios default -drive file=rootfs.ext4,format=raw,id=hd0 \
 -netdev user,id=net0 -device virtio-net-device,netdev=net0 -S -s
```

* **-nographic**: 不使用图形窗口，使用命令行
* **-machine**: 指定要emulate的机器，可以通过命令 `qemu-system-riscv64 -machine help`查看可选择的机器选项
* **-kernel**: 指定内核image
* **-append cmdline**: 使用cmdline作为内核的命令行
* **-device**: 指定要模拟的设备，可以通过命令 `qemu-system-riscv64 -device help`查看可选择的设备，通过命令 `qemu-system-riscv64 -device <具体的设备>,help`查看某个设备的命令选项
* **-drive, file=<file_name>**: 使用'file'作为文件系统
* **-netdev user,id=str**: 指定user mode的虚拟网卡, 指定ID为str
* **-S**: 启动时暂停CPU执行(使用'c'启动执行)
* **-s**: -gdb tcp::1234 的简写
* **-bios default**: 使用默认的OpenSBI firmware作为bootloader

更多参数信息可以参考官方文档[System Emulation — QEMU documentation (qemu-project.gitlab.io)](https://qemu-project.gitlab.io/qemu/system/index.html)

### D. GDB 使用基础

#### 什么是 GDB

GNU调试器（英语：GNU Debugger，缩写：gdb）是一个由GNU开源组织发布的、UNIX/LINUX操作系统下的、基于命令行的、功能强大的程序调试工具。借助调试器，我们能够查看另一个程序在执行时实际在做什么（比如访问哪些内存、寄存器），在其他程序崩溃的时候可以比较快速地了解导致程序崩溃的原因。被调试的程序可以是和gdb在同一台机器上，也可以是不同机器上。

总的来说，gdb可以有以下4个功能：

- 启动程序，并指定可能影响其行为的所有内容
- 使程序在指定条件下停止
- 检查程序停止时发生了什么
- 更改程序中的内容，以便纠正一个bug的影响

#### GDB 基本命令介绍

在gdb命令提示符“(gdb)”下输入“help”可以查看所有内部命令及使用说明。

* (gdb) start：单步执行，运行程序，停在第一执行语句
* (gdb) next：单步调试（逐过程，函数直接执行）,简写n
* (gdb) run：重新开始运行文件（run-text：加载文本文件，run-bin：加载二进制文件），简写r
* (gdb) backtrace：查看函数的调用的栈帧和层级关系，简写bt
* (gdb) break 设置断点。比如断在具体的函数就break func；断在某一行break filename:num
* (gdb) finish：结束当前函数，返回到函数调用点
* (gdb) frame：切换函数的栈帧，简写f
* (gdb) print：打印值及地址，简写p
* (gdb) info：查看函数内部局部变量的数值，简写i；查看寄存器的值i register xxx
* (gdb) display：追踪查看具体变量值

**学会调试将在后续实验中为你提供帮助，推荐同学们跟随[GDB调试入门指南](https://zhuanlan.zhihu.com/p/74897601)教程完成相应基础练习，熟悉gdb调试的使用。**

### E. LINUX 内核编译基础

#### 交叉编译

交叉编译指的是在一个平台上编译可以在另一个平台运行的程序，例如在x86机器上编译可以在arm平台运行的程序，交叉编译需要交叉编译工具链的支持。

#### 内核配置

内核配置是用于配置是否启用内核的各项特性，内核会提供一个名为 `defconfig`(即default configuration) 的默认配置，该配置文件位于各个架构目录的 `configs` 文件夹下，例如对于RISC-V而言，其默认配置文件为 `arch/riscv/configs/defconfig`。使用 `make ARCH=riscv defconfig` 命令可以在内核根目录下生成一个名为 `.config` 的文件，包含了内核完整的配置，内核在编译时会根据 `.config` 进行编译。配置之间存在相互的依赖关系，直接修改defconfig文件或者 `.config` 有时候并不能达到想要的效果。因此如果需要修改配置一般采用 `make ARCH=riscv menuconfig` 的方式对内核进行配置。

#### 常见参数

**ARCH** 指定架构，可选的值包括arch目录下的文件夹名，如x86,arm,arm64等，不同于arm和arm64，32位和64位的RISC-V共用 `arch/riscv` 目录，通过使用不同的config可以编译32位或64位的内核。

**CROSS_COMPILE** 指定使用的交叉编译工具链，例如指定 `CROSS_COMPILE=aarch64-linux-gnu-`，则编译时会采用 `aarch64-linux-gnu-gcc` 作为编译器，编译可以在arm64平台上运行的kernel。

**CC** 指定编译器，通常指定该变量是为了使用clang编译而不是用gcc编译，Linux内核在逐步提供对clang编译的支持，arm64和x86已经能够很好的使用clang进行编译。

#### 常用编译选项

```shell
$ make defconfig	        ### 使用当前平台的默认配置，在x86机器上会使用x86的默认配置
$ make -j$(nproc)	        ### 编译当前平台的内核，-j$(nproc)为以机器硬件线程数进行多线程编译

$ make ARCH=riscv defconfig	### 使用RISC-V平台的默认配置
$ make ARCH=riscv CROSS_COMPILE=riscv64-linux-gnu- -j$(nproc)     ### 编译RISC-V平台内核

$ make clean	                ### 清除所有编译好的object文件
$ make mrproper	                ### 清除编译的配置文件，中间文件和结果文件

$ make init/main.o	        ### 编译当前平台的单个object文件init/main.o（会同时编译依赖的文件）
```
