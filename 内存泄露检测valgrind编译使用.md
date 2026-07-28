

报错信息记录
eam_status=0, lastYoloFrameNo=0,json={"alarm":2,"boxId":"hhkj","channelId":"hhkj01","deviceId":"hhkj01","sendTime":"2026-07-28 09:03:18.559","state":0},packid=1785200597551,sendpacketsize=152,finalpacksize=120,imagepacketsize=0,jsonsize=114,img_width=0,img_height=0
7月 28 09:03:18 RK3588-Tronlong videoia_ptz[640227]: [2026-07-28:09:03:18] total fps=0.0; ch=0, speed=0.0
7月 28 09:03:19 RK3588-Tronlong videoia_ptz[640227]: Move To PreSet : "1"   suceess
7月 28 09:03:19 RK3588-Tronlong videoia_ptz[640227]: Init:total stream num:1
7月 28 09:03:19 RK3588-Tronlong videoia_ptz[640227]: [2026-07-28 09:03:19] avformat_opend_callback: sourceId=0, seq=0
7月 28 09:03:19 RK3588-Tronlong videoia_ptz[640227]: find decoder, codec_id=27, gpu=1
7月 28 09:03:19 RK3588-Tronlong mpp[640227]: mpp_info: mpp version: e34f0dd1 author: Herman Chen   2023-07-17 [hal_vp8e]: Fix crash on unsupport input format
7月 28 09:03:19 RK3588-Tronlong mpp[640227]: mpp_dma_heap: ioctl alloc failed for Cannot allocate memory
7月 28 09:03:19 RK3588-Tronlong mpp[640227]: mpp_dma_heap: os_allocator_dma_heap_alloc dma_heap_alloc failed ret -1
7月 28 09:03:19 RK3588-Tronlong mpp[640227]: mpp_buffer: mpp_buffer_create failed to create buffer with size 5136384
7月 28 09:03:19 RK3588-Tronlong systemd[1]: xlfhvia_ptz.service: Main process exited, code=killed, status=11/SEGV
7月 28 09:03:19 RK3588-Tronlong systemd[1]: xlfhvia_ptz.service: Failed with result 'signal'.
系统日志信息记录
2] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[277]: 0454: 0x00140222
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009007] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[278]: 0458: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009035] rk_iommu fdc48700.iommu: fault addr 0xffffffffffffffff status 0 arg 00000000d62cbb7f
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009043] rk_vcodec: mpp_task_dump_hw_reg:2070: --- dump hardware register ---
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009047] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[000]: 0000: 0x53813f05
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009051] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[001]: 0004: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009056] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[002]: 0008: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009060] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[003]: 000c: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009065] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[004]: 0010: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009070] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[005]: 0014: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009074] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[006]: 0018: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009079] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[007]: 001c: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009083] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[008]: 0020: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009088] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[009]: 0024: 0x00000001
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009092] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[010]: 0028: 0x00000001
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009097] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[011]: 002c: 0x01000062
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009101] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[012]: 0030: 0x00000182
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009106] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[013]: 0034: 0x00048003
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009111] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[014]: 0038: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009116] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[015]: 003c: 0x00000000
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009120] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[016]: 0040: 0x00001a70
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009125] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[017]: 0044: 0x00003fff
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009130] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[018]: 0048: 0x00000078
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009134] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[019]: 004c: 0x00000078
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009139] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[020]: 0050: 0x0001fe00
Jul 28 07:26:43 RK3588-Tronlong kernel: [316172.009144] rk_vcodec: mpp_task_dump_hw_reg:2074: reg[021]: 0054: 0x00000004



valgrind 下载安装
1.valgrind下载：  
http://valgrind.org/downloads/valgrind-3.12.0.tar.bz2  
2.valgrind安装：

tar -jxvf valgrind-3.12.0.tar.bz2  
cd  valgrind-3.12.0  
./configure  
make  
sudo make install

3.查看valgrind是否安装成功

执行 valgrind --help查看是否有帮助参数
4valgrind 使用
valgrind --tool=memcheck --leak-check =full --show-reachable=yes --trace-children=yes    ./videoia_ptz

问题记录

hhkj@RK3588-Tronlong:/data/xlxf/xlfhvia_ptz$ valgrind --tool=memcheck --leak-check=full --show-reachable=yes --trace-children=yes  ./videoia_ptz
==666588== Memcheck, a memory error detector
==666588== Copyright (C) 2002-2022, and GNU GPL'd, by Julian Seward et al.
==666588== Using Valgrind-3.19.0 and LibVEX; rerun with -h for copyright info
==666588== Command: ./videoia_ptz
==666588==

valgrind:  Fatal error at startup: a function redirection
valgrind:  which is mandatory for this platform-tool combination
valgrind:  cannot be set up.  Details of the redirection are:
valgrind:
valgrind:  A must-be-redirected function
valgrind:  whose name matches the pattern:      strlen
valgrind:  in an object with soname matching:   ld-linux-aarch64.so.1
valgrind:  was not found whilst processing
valgrind:  symbols from the object with soname: ld-linux-aarch64.so.1
valgrind:
valgrind:  Possible fixes: (1, short term): install glibc's debuginfo
valgrind:  package on this machine.  (2, longer term): ask the packagers
valgrind:  for your Linux distribution to please in future ship a non-
valgrind:  stripped ld.so (or whatever the dynamic linker .so is called)
valgrind:  that exports the above-named function using the standard
valgrind:  calling conventions for this platform.  The package you need
valgrind:  to install for fix (1) is called
valgrind:
valgrind:    On Debian, Ubuntu:                 libc6-dbg
valgrind:    On SuSE, openSuSE, Fedora, RHEL:   glibc-debuginfo
valgrind:
valgrind:  Note that if you are debugging a 32 bit process on a
valgrind:  64 bit system, you will need a corresponding 32 bit debuginfo
valgrind:  package (e.g. libc6-dbg:i386).
valgrind:
valgrind:  Cannot continue -- exiting now.  Sorry.

问题分析
RK3588 是 **AArch64(arm64)** 平台，报错核心：

> valgrind 找不到动态链接器 `ld-linux-aarch64.so.1` 内的 `strlen` 符号，**动态链接器 ld.so 被 strip（去除符号表）**，Valgrind 3.19.0 在 aarch64 上无法完成函数重定向，直接启动失败。

## 两点关键背景

1. 创龙 RK3588 固件（麒麟 / OpenWrt/Yocto 定制系统）为缩减固件体积，**默认把 ld-linux-aarch64.so.1 做了 strip**；
2. Valgrind aarch64 版本对动态链接器符号极其敏感，**stripped ld.so 直接触发该致命错误**；单纯安装 glibc 调试符号很多嵌入式板子无效。


hhkj@RK3588-Tronlong:~$
Message from syslogd@RK3588-Tronlong at Jul 28 10:56:45 ...
 kernel:[ 2980.977264] Code: 97fff2ba d2800001 d5384100 f9800271 (c85ffe62)

Message from syslogd@RK3588-Tronlong at Jul 28 10:56:45 ...
 kernel:[ 2980.973668] Disabling IRQ #60

Message from syslogd@RK3588-Tronlong at Jul 28 10:56:45 ...
 kernel:[ 2980.974231] Internal error: Oops: 96000006 [#1] SMP


我已经改成了cpu解码器，为什么还有占用

Dma-buf Objects:
size            flags           mode            count           exp_name        ino
00004096        00000002        00080007        00000003        drm     00010531        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

01228800        00000002        00080007        00000003        drm     00010530        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

00028672        00000002        00080007        00000003        drm     00010529        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

00102400        00000002        00080007        00000003        drm     00010528        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

00409600        00000002        00080007        00000003        drm     00010527        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

01228800        00000002        00080007        00000003        drm     00010526        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

04505600        00000002        00080007        00000003        drm     00010524        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

02326528        00000002        00080007        00000003        drm     00010523        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

00036864        00000002        00080007        00000003        drm     00010522        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

06144000        00000002        00080007        00000003        drm     00010521        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

08056832        00000002        00080007        00000003        drm     00010520        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

00045056        00000002        00080007        00000003        drm     00010519        28998-videoia_ptz
        Attached Devices:
Total 0 devices attached

03145728        00000000        00080005        00000006        rockchipdrm     00000011        1354-Xorg
        Attached Devices:
        fb000000.gpu
        fb000000.gpu
Total 2 devices attached

03145728        00000000        00080005        00000006        rockchipdrm     00000010        1354-Xorg
        Attached Devices:
        fb000000.gpu
        fb000000.gpu
Total 2 devices attached

04096000        00000000        00080005        00000006        rockchipdrm     00000007        1354-Xorg
        Attached Devices:
        fb000000.gpu
        fb000000.gpu
Total 2 devices attached

04096000        00000000        00080005        00000006        rockchipdrm     00000006        1354-Xorg
        Attached Devices:
        fb000000.gpu
        fb000000.gpu
Total 2 devices attached

09338880        00000000        00080005        00000004        rockchipdrm     00000001        1354-Xorg
        Attached Devices:
        fb000000.gpu
Total 1 devices attached


