---
title: Arch Linux Note
subtitle:
date: 2024-10-20T20:13:36+08:00
slug: ArchLinuxNote
summary: Arch Linux 安装与配置备忘录，包括网络连接、磁盘分区、安装基本系统、配置新系统、安装引导、重启系统、安全启动、备份快照、定制优化等.
---

## 连接网络
```shell
iwctl # 进入交互式命令行
device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
station wlan0 scan # 扫描网络
station wlan0 get-networks # 列出所有 wifi 网络
station wlan0 connect wifi-name # 进行连接，注意这里无法输入中文。回车后输入密码即可
exit # 连接成功后退出
```
## 更换国内镜像源
```shell
pacman -Sy pacman-mirrorlist # 更新镜像清单
cp /etc/pacman.d/mirrorlist.pacnew /etc/pacman.d/mirrorlist # 使用新的镜像清单
vim /etc/pacman.d/mirrorlist # 编辑镜像清单，取消注释目标镜像源
pacman -Syy # 更新软件包数据库
```
## 磁盘分区
```shell
lsblk # 查看磁盘分区
cfdisk /dev/sdx # 分区
mkfs.fat -F32 /dev/sdxn # 格式化 EFI 分区
mkfs.btrfs -L 'Arch Linux' /dev/sdxn # 格式化根分区
mkswap /dev/sdxn # 格式化交换分区
mount -t btrfs -o compress=zstd /dev/sdxn /mnt # 挂载根分区
btrfs su cr /mnt/@ # 创建子卷
btrfs su cr /mnt/@home # 创建子卷
umount /mnt # 卸载根分区
mount -t btrfs -o compress=zstd,subvol=@ /dev/sdxn /mnt # 挂载根分区
mkdir /mnt/home # 创建家目录
mount -t btrfs -o compress=zstd,subvol=@home /dev/sdxn /mnt/home # 挂载家目录
mkdir -p /mnt/boot # 创建引导目录
mount /dev/sdxn /mnt/boot # 挂载引导目录
swapon /dev/sdxn # 启用交换分区
df -h # 查看挂载情况
free -h # 查看交换分区
```
## 安装基本系统
```shell
pacstrap /mnt base base-devel linux linux-firmware neovim networkmanager btrfs-progs efibootmgr grub os-prober fastfetch sudo git intel-ucode # 安装基本系统
genfstab -U /mnt >> /mnt/etc/fstab # 生成挂载表
cat /mnt/etc/fstab # 复查挂载表
```
## 配置新系统
```shell
arch-chroot /mnt # 进入新系统
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime # 设置时区
hwclock --systohc # 硬件时钟
nvim /etc/locale.gen # 编辑本地化
locale-gen # 生成本地化
echo 'LANG=en_US.UTF-8' > /etc/locale # 设置语言
nvim /etc/hostname # 编辑主机名
nvim /etc/hosts # 加入以下内容
127.0.0.1  localhost
::1        localhost
127.0.1.1  myhostname.localdomain  myhostname
passwd # 设置 root 密码
echo "export EDITOR='nvim'" >> ~/.bash_profile # 设置root用户默认编辑器
useradd -m -G wheel -s /bin/bash username # 添加用户
EDITOR=nvim visudo # 编辑 sudoers 文件，取消注释以下内容：
%wheel ALL=(ALL) ALL
passwd username # 设置用户密码
```
## 安装引导
[Secure Boot Refer](https://www.cnblogs.com/wswind/p/archlinux-secure-boot.html#配置方法)
```shell
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id='Arch Linux' --modules="tpm" --disable-shim-lock
# 若想要引导 Windows 系统，还需执行以下命令：
nvim /etc/default/grub # 取消注释以下内容
GRUB_DISABLE_OS_PROBER=false
grub-mkconfig -o /boot/grub/grub.cfg # 生成引导配置
```
## 重启系统
```shell
exit # 退出新系统
umount -R /mnt # 卸载分区
reboot # 重启系统
```
## 安全启动
```shell
# 进入 BIOS 设置，关闭 Secure Boot，切换至 Setup Mode，修改启动项进入 Arch Linux
sudo pacman -S sbctl # 安装 sbctl
sbctl status # 查看安全启动状态，输出应该如下：
# Installed:    Sbctl is not installed
# Setup Mode:   Enabled
# Secure Boot:  Disabled
sudo sbctl create-keys # 创建密钥
sudo sbctl enroll-keys -m # 导入微软密钥
sudo sbctl verify # 验证密钥，将列出需要签署的文件
sudo sbctl sign -s <file> # 签署文件
sudo sbctl list-files # 列出已签署文件
```
## 备份快照
```shell
sudo pacman -S timeshift cronie # 安装 timeshift 和 cronie
sudo systemctl enable cronie # 启用 cronie
sudo nvim /etc/timeshift/timeshift.json # 编辑配置文件
sudo sed -i -E 's/(subvolid=[0-9]+,)|(,subvolid=[0-9]+)//g' /etc/fstab # 删除 fstab 中的 subvolid
sudo timeshift --create --comments "Initial Backup" # 创建快照
```
## 定制优化（杂项）
### 桌面环境
```shell
sudo pacman -S plasma-meta dolphin konsole
sudo systemctl enable sddm
```
- SDDM登录界面缩放异常 [Scaling Issue On SDDM](https://forums.opensuse.org/t/ui-scaling-in-sddm-no-longer-works-after-kde-plasma-6-upgrade/173350/3)
### AUR
```shell
cd ~/Downloads
git clone https://aur.archlinux.org/paru
cd paru
makepkg -sic
```
### 输入法
```shell
paru -S fcitx5-im fcitx5-chinese-addons # 通过KDE的虚拟键盘启用输入法
```
### 显卡驱动
```shell
sudo pacman -S mesa lib32-mesa vulkan-intel lib32-vulkan-intel # 安装 Intel 集成显卡驱动
sudo pacman -S nvidia nvidia-settings lib32-nvidia-utils # 安装 Nvidia 独立显卡驱动
paru -S optimus-manager optimus-manager-qt
```
- 使用独显启动应用 [KDE Walyand Nvidia Only](https://arch.icekylin.online/guide/rookie/graphic-driver#双显卡-集显-独显)
### 休眠
- 见 [Arch Linux Hibernate](https://wiki.archlinuxcn.org/wiki/电源管理/挂起与休眠#休眠)
### 常用软件
```shell
nvim /etc/pacman.conf # 编辑软件源
# 取消注释以下内容以开启 multilib 软件源
[multilib]
Include = /etc/pacman.d/mirrorlist
# 添加 archlinuxcn 软件源
[archlinuxcn]
Server = https://repo.archlinuxcn.org/$arch
paru -S archlinuxcn-keyring # 安装 archlinuxcn 密钥
sudo pacman -Syy # 更新软件源
paru -S obsidian syncthing telegram discord steam visual-studio-code-bin clash-verge-rev-bin wechat-universal-bwrap linuxqq 
paru -S firefox-kde-opensuse # 与 KDE 集成的 Firefox
```
- Steam缩放问题 [KDE Steam Client UI scaling issue](https://steamcommunity.com/app/221410/discussions/0/3801651661442104568/)