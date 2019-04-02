---
title: Firecraker调研-初探
date: 2019-03-26 22:14:34
tags:
---

### 简介
Firecracker 是 AWS 开源的用于 Serverless 计算的安全且快速的微虚拟机（microVM)。
根据AWS官方网站介绍，在推出AWS Lambda之时， 为了达到理想的隔离状况，为每位客户使用了专用的EC2实例。  
后来因为效率原因，开发了Firecracker

### 特性
- 安全，使用多重隔离和保护，暴露的攻击面极小 。 
- 高性能，在125ms的时间内启动microVM（2019年将会进一步加快）。
- 经过广泛测试，已经为多种高容量AWS服务提供支持，包括AWS Lambda 和 AWS Fargate。
- 低开销，每个microVM仅占用5MiB内存


### 安全性  
以下列出Firecracker的一部分安全功能：
- 简单访客模型-Firecracker访客将获得非常简单的虚拟化设备模型，以最大限度地缩减攻击面：网络设备，块I/O设备，可编程的间隔定时器，KVM时钟，串型控制器和部分键盘
- 进程监禁- Firecracker进程使用cgroup和seccomp BPF进行监禁，而且可以访问一小部分收到严密控制的系统调用
- 静态链接- Firecracker进程以静态形式链接，可以通过jailer启动，以尽可能确保托管环境安全干净

 
### quick-start 操作
在本地电脑上操作，Firecracker目前支持 Linux x86_64 主机，内核版本在4.14+，同时需要开启KVM功能,且能够读写**/dev/kvm**  
首先需要三个文件（firecracker二进制文件，根文件系统和Linux内核）  
打开两个命令行窗口  
- 在第一个窗口：
    - 确保Firecracker能够创建其Unix socket：
    ```
    rm -f /tmp/firecracker.socket
    ```
    - 启动Firecracker：
    ```
    ./firecracker --api-sock /tmp/firecracker.socket
    ```
- 在第二个窗口：
    - 设置内核：
    ```
    curl --unix-socket /tmp/firecracker.socket -i \
        -X PUT 'http://localhost/boot-source'   \
        -H 'Accept: application/json'           \
        -H 'Content-Type: application/json'     \
        -d '{
            "kernel_image_path": "./hello-vmlinux.bin",
            "boot_args": "console=ttyS0 reboot=k panic=1 pci=off"
        }'
    ```
    - 设置根文件系统：
    ```
    curl --unix-socket /tmp/firecracker.socket -i \
        -X PUT 'http://localhost/drives/rootfs' \
        -H 'Accept: application/json'           \
        -H 'Content-Type: application/json'     \
        -d '{
            "drive_id": "rootfs",
            "path_on_host": "./hello-rootfs.ext4",
            "is_root_device": true,
            "is_read_only": false
        }'
    ```
    - 启动机器
    ```
    curl --unix-socket /tmp/firecracker.socket -i \
        -X PUT 'http://localhost/actions'       \
        -H  'Accept: application/json'          \
        -H  'Content-Type: application/json'    \
        -d '{
            "action_type": "InstanceStart"
         }'
    ```
### 后续
详解Firecracker启动之后暴露的API，由[OpenAPI](https://github.com/firecracker-microvm/firecracker/blob/master/api_server/swagger/firecracker.yaml)格式制定  
参阅[API文档](https://github.com/firecracker-microvm/firecracker/blob/master/docs/api_requests)  
参阅[生产环境配置文档](https://github.com/firecracker-microvm/firecracker/blob/master/docs/prod-host-setup.md)
