---
title: Firecracker调研-API规范及用处
date: 2019-03-27 19:27:56
tags:
---
## 作用
当Firecracker启动的时候，会通过unix socket暴露一个API,可用于：
- 配置microVM：
    - 设定CPU数量（默认为1）
    - 设定内存大小（默认128MB）
    - 选择CPU模板（目前由C3和T2可供选择）
- 添加一个或多个网络设备到microVM
- 添加多个可读写或只读硬盘到microVM， 每一个表现为 file-backed block device(不太懂)
- 触当用户在运行的时候触发 block device扫描。使得客户系统能够获取设备备份文件的大小（不太懂）
- 更改块设备的备份文件，在客户启动前或后（不太懂）
- 配置虚拟I/O设备的限流器，做到带宽限制，每秒操作数限制
- 配置日志及计量系统
- [BETA] Configure the data tree of the guest-facing metadata service. The service is only available to the guest if this resource is configured.(不懂)
- [EXPERIMENTAL] 添加一个或多个[vsock sockets]到microVM
- 制定microVM使用的内核镜像，根文件系统及启动参数
- 定制microVM


## 规范
   ###  ACTIONS
   #### BlockDeviceRescan
   这个操作用于触发附在microVM上的一个块设备的重扫描，当一个块设备的backing file改变的时候，Rescanning是必须的，guest需要刷新它的内部数据结构来拾取这种改变  
   这个操作只能在guest启动后执行。
   为了使重扫描正确运行，调用这个操作的时候块设备不能挂载在guest。否则这个调用会默默失败掉，guest的内部数据结构会处于一个不一致的状态
   #### BlockDeviceRescan Example
   ```
   # Create and set up a block device.
   touch ${ro_drive_path}
   
   curl --unix-socket ${socket} -i \
        -X PUT "http://localhost/drives/scratch" \
        -H "accept: application/json" \
        -H "Content-Type: application/json" \
        -d "{
               \"drive_id\": \"scratch\",
               \"path_on_host\": \"${ro_drive_path}\",
               \"is_root_device\": false,
               \"is_read_only\": true
            }"
   
   # Finish configuring and start the microVM. Wait for the guest to boot.
   
   # Resize the block device's backing file and create a filesystem in it.
   truncate --size 100M ${ro_drive_path}
   mkfs.ext4 ${ro_drive_path}
   
   curl --unix-socket ${socket} -i \
        -X PUT "http://localhost/actions" \
        -H "accept: application/json" \
        -H "Content-Type: application/json" \
        -d "{
               \"action_type\": \"BlockDeviceRescan\",
               \"payload\": \"scratch\"
            }"
   ```

## 后续
Firecracker API 遵循OPEN API，了解OPEN API
