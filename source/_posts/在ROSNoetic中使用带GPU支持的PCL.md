---
title: 在 ROS Noetic 中使用带 GPU 支持的 PCL
date: 2024-03-05T17:09:40+08:00
toc: true
cover:
thumbnail:
categories:
    - ROS
tags:
    - ROS
    - Linux
    - 自动驾驶
---

# 前言
ROS Noetic 中自带的 PCL 1.10 并没有 GPU 支持。如果想要启用 GPU/CUDA 支持的话，只能自己编译。    
然而与 OpenCV 类似，ROS 使用了 pcl_ros, pcl_conversions 等包作为 PCL 与 在 ROS 的接口，使用自编译版本的 PCL 需要修改这些包的设置，较为麻烦。     
以下是在 ROS 中使用自编译（带 GPU 支持）版本 PCL 的完整步骤。

# 编译 PCL
最新版 PCL (PCL 1.14.0) 若启用 gpu 支持，需要 CUDA Toolkit v9.2+   
同时，因为使用了 `ccmake`，需要安装 `cmake-curses-gui`: 

    sudo apt install cmake-curses-gui
在项目[官方仓库](https://github.com/PointCloudLibrary/pcl)下载 PCL，然后签出到自己需要的版本。
> 一般保持在 master 分支即可。若想使用特定版本，可以使用 `git tag` 查看版本并签出。
签出完成后，至项目目录下：

```
mkdir build && cd build
ccmake ..
```

此时，进入 cmake 配置界面。    
按 `c` 进入编译选项配置。
![ccmake](ccmake-1.png)
这里把选项 `BUILD_CUDA` 和 `BUILD_GPU` 打开。   
多按几次 `c`，会根据之前的选项生成新的选项。然后根据需要打开选项。   
按 `c`，直到下方出现按键 `g` 的提示后，按 `g`。此时配置完成。
之后编译安装：
```
make
sudo make install
```
默认安装路径下，PCL 会被安装到 `/usr/local`。

# 修改 pcl_conversions 和 pcl_ros
对于 pcl_conversion，需要修改的文件是 `/opt/ros/noetic/share/pcl_ros/cmake/pcl_rosConfig.cmake`    
首先是修改头文件的目录。找到含有 `include` 目录的行，将 `/usr/include/pcl-1.10` 改为 `/usr/local/include/pcl-1.14`    
再往下到 `set(libraries` 开头的行，将所有 pcl 的库路径修改至 `/usr/local/lib` 下。      
对于 pcl_ros，也是同样操作。但 pcl_ros 可能缺少部分 pcl 库的路径，如 `libpcl_registration.so`。需要手动加上。

> 实际路径根据安装位置确定，这里是默认安装位置。


至此，安装已全部完成。所有依赖 pcl，pcl_ros，pcl_conversion 的包应该都能正常编译。    
如果发现链接器报错 `undefined reference`，可能是 pcl_ros 的部分库目录没有加上，找到没有加上的路径就好。