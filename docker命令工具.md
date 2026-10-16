```
#查看正在运行容器 docker ps 
#查看所有容器 docker ps -a 
#查看最新创建的容器 docker ps -l

#创建容器 docker create 镜像 
--创建一个 nginx 容器：docker create nginx 
--指定容器的name为nginx：docker create --name=nginx nginx 
#创建并启动一个容器 docker run 镜像 --docker run --name nginx1 -d -p 8080:80 nginx
 --name 含义和上文一样，表示创建的容器的名字，
 -d 表示容器在后台运行，
 -p 表示将容器的 80 端口映射到宿主机的 8080 端口
 
 #删除容器 docker rm id|name 
 #删除所有停止运行的容器 
 docker ps -aq | xargs -r docker rm -f
 
 
 #启动容器 docker start id|name #重启容器 docker restart id|name #关闭容器 docker stop id|name
 
 
 #容器详情 docker inspect id|name #查看容器进程 docker top #查看你容器日记 docker logs id|name -f ：实时日记 -t : 显示时间 -tail： 控制输出行数
 
 #进入一个正在运行的容器 docker exec -it 容器名称 /bin/bash #退出容器 exit
 
 
 
 #导出一个已经创建的容器导到一个文件 docker export -o 文件名.tar 容器id #将文件导入为镜像 docker import 文件名.tar 镜像名:镜像标签 docker load -i 文件名.tar
 
 
 #容器保存为镜像 docker commit 1fc7ba65fa07 hhkj-huawei-gwxf:v8 #导出镜像 docker save -o hhkj-huawei-gwxf-v8.tar hhkj-huawei-gwxf:v8
 
 
 docker cp [OPTIONS] SRC_PATH|- CONTAINER:DEST_PATH
 
 
 
 docker run -itd -m 8192M --cpus 8 -u root --device=/dev/davinci0 \
--device=/dev/davinci_manager --device=/dev/devmm_svm --device=/dev/hisi_hdc \
-v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro \
-v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro \
--privileged=true  -p 30507:30507 \
 hhkj-ascend-infer:v1.0.2510.2 /bin/bash
 
 
 
 sudo -E python -m pip install pyav -i https://pypi.tuna.tsinghua.edu.cn/simple
 10.2版本启动命令
 docker run -itd -m 8192M --cpus 8 -u root \
--device=/dev/davinci0 \
--device=/dev/davinci_manager \
--device=/dev/devmm_svm \
--device=/dev/hisi_hdc \
-v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro \
-v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro \
--privileged=true \
-p 30507:30507 \
-p 30508:8080 \
hhkj-ascend-infer:v1.0.2510.2 /bin/bash
 
 
 
 10.2
 source /usr/local/Ascend/ascend-toolkit/set_env.sh
 export LD_PRELOAD="/usr/local/Ascend/driver/lib64/common/libc_sec.so:/usr/local/python3/lib/python3.9/site-packages/torch.libs/libgomp-6e1a1d1b.so.1.0.0"
 
 
 python setup.py build_ext  --inplace
 
 
 av  库编译环境设置
 cat > /tmp/ffmpeg-pc/libavutil.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libavutil Description: FFmpeg utility library Version: 56.31.100 Libs: -L${libdir} -lavutil Cflags: -I${includedir} EOF cat > /tmp/ffmpeg-pc/libavcodec.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libavcodec Description: FFmpeg codec library Version: 58.54.100 Libs: -L${libdir} -lavcodec -lavutil Cflags: -I${includedir} EOF cat > /tmp/ffmpeg-pc/libavformat.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libavformat Description: FFmpeg format library Version: 58.29.100 Libs: -L${libdir} -lavformat -lavcodec -lavutil Cflags: -I${includedir} EOF cat > /tmp/ffmpeg-pc/libavdevice.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libavdevice Description: FFmpeg device library Version: 58.8.100 Libs: -L${libdir} -lavdevice -lavformat -lavcodec -lavutil Cflags: -I${includedir} EOF cat > /tmp/ffmpeg-pc/libavfilter.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libavfilter Description: FFmpeg filter library Version: 7.57.100 Libs: -L${libdir} -lavfilter -lavformat -lavcodec -lavutil -lswscale Cflags: -I${includedir} EOF cat > /tmp/ffmpeg-pc/libswscale.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libswscale Description: FFmpeg scale library Version: 5.5.100 Libs: -L${libdir} -lswscale -lavutil Cflags: -I${includedir} EOF cat > /tmp/ffmpeg-pc/libswresample.pc <<'EOF' prefix=/usr/local/ffmpeg exec_prefix=${prefix} libdir=${prefix}/lib includedir=${prefix}/include Name: libswresample Description: FFmpeg resample library Version: 3.5.100 Libs: -L${libdir} -lswresample -lavutil Cflags: -I${includedir} EOF
 
 配置环境变量
 export PKG_CONFIG_PATH=/tmp/ffmpeg-pc pkg-config --modversion libavformat pkg-config --libs libavfilter
 
 切回源码 av8.1目录
 python setup.py build_ext --inplace
 python setup.py install
 
 echo "/usr/local/ffmpeg/lib" > /etc/ld.so.conf.d/ffmpeg.conf ldconfig
 
 python -c "import av; print('av ok')"
 
 
 
 
 
 
 260805-版本
 
 
 docker commit 93e59408ed47 hhkj-ascend-infer:v1.0.2608.5-avde
 启动命令
 docker run -itd -u root \ --device=/dev/davinci0 \ --device=/dev/davinci_manager \ --device=/dev/devmm_svm \ --device=/dev/hisi_hdc \ -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro \ -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro \ -p 30507:30507 -p 8080:8080 \ hhkj-ascend-infer:v1.0.2608.5-avde /bin/bash
 环境变量
 source /usr/local/Ascend/ascend-toolkit/set_env.sh
 export LD_PRELOAD="/usr/local/Ascend/driver/lib64/common/libc_sec.so:/usr/local/python3/lib/python3.9/site-packages/torch.libs/libgomp-6e1a1d1b.so.1.0.0"
 
 export LD_LIBRARY_PATH=/usr/local/ffmpeg/lib:$LD_LIBRARY_PATH
 
 
 
 
 
 docker run -itd -u root -e ASCEND_RT_VISIBLE_DEVICES=0  -m 8192M --cpus 8 --name mul_analysis --restart no --device=/dev/davinci0 --device=/dev/davinci_manager --device=/dev/devmm_svm --device=/dev/hisi_hdc -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro -p 30507:30507 -p 8080:8080 hhkj-ascend-infer:v1.0.2608.5-avde /bin/bash /home/HwHiAiUser/analysisServer_dvppbak/start.sh

 
 
 docker run -itd -u root -e ASCEND_RT_VISIBLE_DEVICES=1   -m 8192M --cpus 8 --device=/dev/davinci0 --device=/dev/davinci_manager --device=/dev/devmm_svm --device=/dev/hisi_hdc -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro -p 30507:30507 -p 8080:8080 hhkj-ascend-infer:v1.0.2608.5-avde /bin/bash
 
 
 
 
 
 docker run -itd -u root -e ASCEND_RT_VISIBLE_DEVICES=0  -m 8192M --cpus 8 --name mul_analysis --restart no --device=/dev/davinci0 --device=/dev/davinci_manager --device=/dev/devmm_svm --device=/dev/hisi_hdc -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro -p 30507:30507 -p 8080:8080 hhkj-ascend-infer:v1.0.2608.11-avde /bin/bash /home/HwHiAiUser/analysisServer_dvppbak/start.sh
 
 
 docker commit \ --message="容器运行环境固化 2026-08-11更新" \ mul_analysis \ hhkj-ascend-infer:v1.0.2608.11-avde
 
 
 
 
 
 
 
 
 
 关于docekr 开机自起的方案
 1.首先将最新的程序代码文件夹的所有内容 拷贝到Docker容器里面，/home/HwHiAiUser/analysisServer_dvppbak/对应的路径，然后
 执行以下命令进行固化镜像，具体参数按照日期进行调整版本号每次固化
 docker commit \ --message="容器运行环境固化 2026-08-11更新" \ mul_analysis \ hhkj-ascend-infer:v1.0.2608.11-avde
 2.执行完以后启动镜像容器（注意版本号）记得删除不要的Docker 容器避免资源占用
 docker run -itd -u root -e ASCEND_RT_VISIBLE_DEVICES=0  -m 8192M --cpus 8 --name mul_analysis --restart no --device=/dev/davinci0 --device=/dev/davinci_manager --device=/dev/devmm_svm --device=/dev/hisi_hdc -v /usr/local/bin/npu-smi:/usr/local/bin/npu-smi:ro -v /usr/local/Ascend/driver:/usr/local/Ascend/driver:ro -p 30507:30507 -p 8080:8080 hhkj-ascend-infer:v1.0.2608.11-avde /bin/bash /home/HwHiAiUser/analysisServer_dvppbak/start.sh
 3.docker ps查看，看是否启动成功，启动成功后就可以正常运行了，
 4. 将程序文件夹的下的analysisServer_docker.service文件拷贝到/etc/systemd/system,然后执行enable 设置开机自启可以拉起docker 容器（更换时记得先停止该服务，避免多次拉起）
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    
    0813 问题记录排查
    [ERROR] Copy host data to host memory failed, error: 107000
[ERROR] Copy image to host failed
[2026-08-13 10:10:44,580] [ERROR] >>>  分析错误: copy_to_host return None, copy failed
[2026-08-13 10:10:44,580] [ERROR] >>>  Traceback (most recent call last):
  File "/home/pdy/analysisServer_dvppbak_localbak/ImgAnalysis.py", line 102, in run
    filter_befor_results = self.hhdet.detect_smart_1_batch_detr(frame)
  File "/home/pdy/analysisServer_dvppbak_localbak/source/HHOD_box.py", line 747, in detect_smart_1_batch_detr
    img1=self.convertImagetoopencv(img)
  File "/home/pdy/analysisServer_dvppbak_localbak/source/HHOD_box.py", line 724, in convertImagetoopencv
    raise RuntimeError("copy_to_host return None, copy failed")
RuntimeError: copy_to_host return None, copy failed


