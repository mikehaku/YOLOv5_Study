# YOLOv5_Study
此文档主要用于记录YOLOv5的学习过程。目前主要还是从TensorRT加速网络的角度入手。但目前发现，不把网络内部的结构吃透，根本没法搞清楚如何修改和优化。因此，考虑作为一个长期的课题，进行探讨和分析。

## 1. 关于历史版本的问题。

YOLOv5是YOLO系列目标检测网络的经典之作。有必要将该项目完全吃透，以提升后续修改和使用其他类似目标检测网络的能力。

目前找到的网上教程是2022年以及2023年的版本（主要是恩培教程，以C++和TensorRT为主的一个教程）。其中的一些export.patch文件已经不是当时的版本。需要通过github的历史版本寻找当时的版本。虽然当时的版本有很多bug在后续时间修复了，但目前最合适的思路仍然是先将当时的版本在本地复现，然后再探索目前版本与当时版本的区别。

当前版本的地址是：https://github.com/ultralytics/yolov5

历史版本的地址是：https://github.com/ultralytics/yolov5/tree/7cef03dddd6fba26fff6748ed1cfdd18208c193e

## 2. YOLO前处理的CUDA手写中遇到的一些问题汇总

### 1. 安装C++版本的OpenCV4遇到的问题

C++的OpenCV4安装，基本流程大致如下：
1. 使用Ubuntu自带的apt工具，安装build-essential等支撑OpenCV的基本lib。
2. 下载OpenCV源码和contrib源码，contrib是一些非官方的代码，一些特殊的库可能会用到，最好安装。
3. 设计CMake的语句，设置需要安装和不需要安装的一些包，例如CUDA/CUDNN等需要显式规定出来。
4. 创建build文件夹，执行cmake指令。
5. 执行make指令，可以使用nproc观看CPU内核数量，再使用-j20等参数加快编译速度。
6. sudo make install，可以将OpenCV安装到系统的目录中，使得后续的CMake工具可以发现其目录。

过程中，遇到过的最头疼的问题是，安装后使用过程中，对程序进行make操作，出现了下面的报错：
/lib/x86_64-linux-gnu/libgio-2.0.so.0: undefined symbol: g_log_set_debug_enabled
这是由于安装C++版本的OpenCV时，没有关闭Conda环境导致的。切换到base环境，安装后依然报错。之后我直接卸载Conda，再安装OpenCV，亲测确实解决了问题，但也很伤，之前费了很大力气安装的环境都没了。
正确的操作，似乎应该是把conda关掉，命令行界面连(base)都不显示的那种状态，可惜没有去试。下次可以用公司服务器试试。

### 2. CUDA核函数执行时间非常长

使用CUDA和C++编写YOLO的前处理程序，做HWC->CHW+normalize+BGR->RGB的操作，自编函数耗时长达100ms以上，而教程仅需要3ms。
经过各方面排查，包括访存方式等，最终发现，问题出在访问GPU内部数组越界上。正常的线程会一直等待越界的线程，直到超时退出。但此类问题，在GPU编程中并不会有明显的报错。
因此，这种情况需要仔细手动校验。这个也算是一个经验。
