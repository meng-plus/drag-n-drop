# drag-n-drop

DAPLink 拖拽升级组件，用于实现通过 USB 虚拟磁盘进行固件升级的功能。

## 概述

本组件从 DAPLink 项目中剥离出来，提供了完整的拖拽升级功能实现。用户可以通过将固件文件拖拽到虚拟磁盘中，实现固件的自动烧录。

## 功能特性

- **虚拟文件系统 (VFS)**: 实现 FAT12/16 文件系统，通过 USB 模拟可移动磁盘
- **多格式支持**: 支持 BIN 和 Intel HEX 格式的固件文件
- **Flash 管理**: 提供完整的 Flash 编程接口，支持页擦除、扇区擦除、整片擦除
- **文件流解析**: 自动识别文件格式并解析
- **Flash 接口抽象**: 支持多种 Flash 接口实现（IAP、目标 Flash 等）

## 组件结构

```
drag-n-drop/
├── virtual_fs.c/h          # 虚拟文件系统核心实现
├── vfs_manager.c/h          # VFS 管理器，处理文件系统状态机
├── vfs_user.c/h             # VFS 用户接口，需要用户实现
├── file_stream.c/h          # 文件流解析器（BIN/HEX）
├── intelhex.c/h             # Intel HEX 格式解析器
├── flash_manager.c/h         # Flash 管理器，处理数据写入
├── flash_intf.h             # Flash 接口抽象定义
├── flash_decoder.c/h        # Flash 解码器，选择 Flash 接口
└── iap_flash_intf.c         # IAP Flash 接口实现示例
```

## 核心模块

### 虚拟文件系统 (VFS)

提供 FAT12/16 文件系统实现，支持：
- 文件创建、删除、读写
- 文件属性管理
- 文件变更回调

### Flash 管理

提供 Flash 编程接口抽象：
- `flash_intf_t`: Flash 接口结构体，包含初始化、擦除、编程等回调函数
- `flash_manager`: 管理 Flash 数据写入，处理地址对齐、扇区擦除等

### 文件流处理

支持两种文件格式：
- **BIN**: 二进制格式，直接写入 Flash
- **HEX**: Intel HEX 格式，自动解析地址和数据

## 使用方法

### 1. 实现 Flash 接口

实现 `flash_intf_t` 结构体中的回调函数：

```c
const flash_intf_t my_flash_intf = {
    .init = my_flash_init,
    .uninit = my_flash_uninit,
    .program_page = my_flash_program_page,
    .erase_sector = my_flash_erase_sector,
    .erase_chip = my_flash_erase_chip,
    .program_page_min_size = my_flash_program_page_min_size,
    .erase_sector_size = my_flash_erase_sector_size,
    .flash_busy = my_flash_busy,
    .flash_algo_set = my_flash_algo_set,
};
```

### 2. 初始化 VFS

在 USB 初始化后调用：

```c
vfs_mngr_init(true);  // 启用虚拟文件系统
```

### 3. 构建文件系统

实现 `vfs_user_build_filesystem()` 函数，创建虚拟文件：

```c
void vfs_user_build_filesystem(void)
{
    vfs_init("DAPLINK", disk_size);

    // 创建固件文件
    vfs_file_t file = vfs_create_file("FIRMWARE.BIN",
                                       read_callback,
                                       write_callback,
                                       file_size);
}
```

### 4. 处理文件变更

实现 `vfs_user_file_change_handler()` 函数，处理文件拖拽事件：

```c
void vfs_user_file_change_handler(const vfs_filename_t filename,
                                   vfs_file_change_t change,
                                   vfs_file_t file,
                                   vfs_file_t new_file_data)
{
    if (change == VFS_FILE_CHANGED) {
        // 文件被拖拽，开始解析和烧录
        stream_type_t type = stream_type_from_name(filename);
        stream_open(type);
        flash_manager_init(&my_flash_intf);
        // ...
    }
}
```

### 5. 周期性调用

在 USB 线程中周期性调用：

```c
void usb_thread(void)
{
    while (1) {
        vfs_mngr_periodic(elapsed_ms);
        // ...
    }
}
```

## 依赖项

本组件需要以下依赖：

- `error.h`: 错误码定义（需要从 DAPLink 主项目获取或自行实现）
- USB 设备栈（用于虚拟磁盘功能）
- Flash 驱动（需要用户实现）

## 许可证

Apache License 2.0

Copyright (c) 2009-2019, ARM Limited, All Rights Reserved

## 来源

本组件来自 [DAPLink](https://github.com/ARMmbed/DAPLink) 主仓库提交点：`8f4e9e40cf1a0902b88088a5a55a8c0559f709a5`

## 注意事项

1. **线程安全**: VFS 管理器相关函数必须在 USB 线程中调用
2. **错误处理**: 需要实现 `error.h` 中定义的错误码类型
3. **Flash 接口**: 必须实现完整的 Flash 接口回调函数
4. **文件系统**: 文件系统必须在 USB 初始化后创建

## 集成示例

参考 `iap_flash_intf.c` 中的 IAP Flash 接口实现示例，了解如何实现 Flash 接口。

