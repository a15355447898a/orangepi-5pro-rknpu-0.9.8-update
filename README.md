# 在 Orange Pi 5PRO 上更新 RKNPU 驱动程序 0.9.8

本指南提供在 Orange Pi 5PRO 上将 RKNPU 驱动程序更新至 0.9.8 版本的详细说明，该设备运行 Linux 和 Linux 6.1.43-rockchip-rk3588 内核。升级 RKNPU 驱动程序对于成功运行 RKLLM 多模态模型至关重要。

**您可以直接克隆此存储库并直接跳转到步骤 9**，或者继续在您的板上构建内核。

> 更新日志: 
>
> * 2025/11/1: fork了原仓库,构建了Orange Pi 5PRO的内核,并且开启了`CONFIG_SECURITY_LANDLOCK`(codex需要这个)

# 构建带 RKNPU 驱动程序 0.9.8 的 Linux 6.1.43 rk3588 内核

## 先决条件

- **操作系统**：debian系的Linux
- **内核版本**：Linux 6.1.43-rockchip-rk3588
- **磁盘空间**：至少 20 GB 可用空间
- **网络访问**：确保开发板具有互联网访问权限，以克隆存储库和下载必要的软件包。

## 步骤 1：验证当前的 NPU 驱动程序版本

在继续之前，请检查 RKNPU 驱动程序的当前版本：

```bash
sudo cat /sys/kernel/debug/rknpu/version
```

如果输出显示的版本低于 0.9.8，请继续执行以下步骤升级驱动程序。

## 步骤 2：安装所需的依赖项

确保您的系统已安装必要的软件包：

```bash
sudo apt-get update
sudo apt-get install -y git cmake
```

## 步骤 3：克隆 Orange Pi 构建存储库

Orange Pi 构建存储库基于 Armbian 构建框架，用于为 Orange Pi 板编译 Linux 内核。克隆存储库：

```bash
cd ~
git clone https://github.com/orangepi-xunlong/orangepi-build.git -b next
```

## 步骤 4：下载 Linux 6.1 内核源代码

创建一个目录来存储内核源代码并进入该目录：

```bash
cd orangepi-build
mkdir kernel && cd kernel
```

克隆内核源代码：

```bash
git clone https://github.com/orangepi-xunlong/linux-orangepi.git -b orange-pi-6.1-rk35xx
```

为保持一致性重命名目录：

```bash
mv linux-orangepi/ orange-pi-6.1-rk35xx
```

## 步骤 5：下载并解压 RKNPU 驱动程序

从官方存储库获取 RKNPU 驱动程序 0.9.8 版本：

```bash
cd ~/orangepi-build
git clone https://github.com/airockchip/rknn-llm.git
```

解压驱动程序：

```bash
tar -xvf /rknn-llm/rknpu-driver/rknpu_driver_0.9.8_20241009.tar.bz2
```

将解压后的驱动程序文件复制到内核源代码中：

```bash
cp -r drivers/ kernel/orange-pi-6.1-rk35xx/
```

## 步骤 6：修改内核源代码文件

为确保兼容性并避免编译错误，请进行以下修改：

1. **修改 kernel/include/linux/mm.h 文件**：

   编辑文件：

   ```bash
   sudo nano kernel/orange-pi-6.1-rk35xx/include/linux/mm.h
   ```

   在适当的位置添加以下代码：

   ```c
   static inline void vm_flags_set(struct vm_area_struct *vma, vm_flags_t flags)
   {
       vma->vm_flags |= flags;
   }
   static inline void vm_flags_clear(struct vm_area_struct *vma, vm_flags_t flags)
   {
       vma->vm_flags &= ~flags;
   }
   ```
![图片](https://github.com/user-attachments/assets/adcb44bc-15b9-41bd-bd53-72273c06d021)

2. **修改 rknpu_devfreq.c 文件：**

   编辑文件：

   ```bash
   sudo nano kernel/orange-pi-6.1-rk35xx/drivers/rknpu/rknpu_devfreq.c
   ```

   注释掉第 242 行的 .set_soc_info = rockchip_opp_set_low_length,

   ```c
   //.set_soc_info = rockchip_opp_set_low_length,
   ```
![图片](https://github.com/user-attachments/assets/26e01e59-d2b1-4f29-b997-f171b998ec8f)

## 步骤 7：禁用源代码同步

因为我们之前手动将驱动程序覆盖到 kernel/orange-pi-6.10-rk35xx 目录，如果现在直接运行编译，脚本会检查与云端源代码的不一致，导致重新拉取代码覆盖的问题。因此，应在配置文件中禁用源代码同步功能。

首先，运行一次 build.sh 脚本进行初始化：

1. 运行构建脚本进行初始化：
   ```bash
   sudo ./build.sh
   ```
   稍等片刻，当看到要求我们进行选择的界面时，使用键盘上的 → 箭头键和 Enter 键退出菜单。

   再次检查当前目录，发现多了一个 userpatches 文件夹，其中包含配置文件。
2. 更新 config-default.conf 文件：
   ```bash
   sudo nano userpatches/config-default.conf
   ```
3. 找到并将 IGNORE_UPDATES 设置为 yes：
   ```bash
   IGNORE_UPDATES="yes"
   ```

## 步骤 8：编译 Linux 内核

启动构建过程：

```bash
sudo ./build.sh docker
./build.sh
```

在提示时为您的板和内核版本选择适当的选项。

![图片](https://github.com/user-attachments/assets/c7730fc3-59ce-404b-ad18-e95c2e2812e7)

![图片](https://github.com/user-attachments/assets/87a5024e-0561-41df-9f16-231dda9f56db)

![图片](https://github.com/user-attachments/assets/fb142587-3888-4964-bdd3-e5a3b051e725)

![图片](https://github.com/user-attachments/assets/f078d22f-886a-40b0-9bc6-8b8773c8ac00)

成功构建后，您将看到类似以下的消息（注意：首次构建可能需要近 40 分钟！）
![图片](https://github.com/user-attachments/assets/addcd887-8dda-4b7c-a2fb-f4ddd6c011f5)

检查文件
![图片](https://github.com/user-attachments/assets/f32ef375-33cd-4c30-8af0-e86cdefe0e13)

## 步骤 9：安装新内核

仅需安装 **linux-image-current-rockchip-rk3588_1.0.6_arm64.deb** 软件包：

```bash
sudo apt purge -y linux-image-current-rockchip-rk3588
sudo dpkg -i output/debs/linux-image-current-rockchip-rk3588_1.0.6_arm64.deb
```

## 步骤 10：验证更新后的 NPU 驱动程序版本

重新启动系统：

```bash
sudo reboot
```

重新启动后，验证 NPU 驱动程序版本：

```bash
sudo cat /sys/kernel/debug/rknpu/version
```
输出现在应显示版本 0.9.8。
![图片](https://github.com/user-attachments/assets/80a35e0a-8389-4800-bf2d-c27547155f0c)

通过执行这些步骤，您应该已成功将 RKNPU 驱动程序更新到 0.9.8 版本，从而可以在您的 Orange Pi 5PRO 上部署 RKLLM 多模态模型。
