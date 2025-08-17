Arch Linux reference

参考网站：

---

## 安装 Arch Linux

### 安装前的准备

#### 获取安装映像

archlinux-downlode

#### 验证哈希校验和

目的：防止下载过程中损坏或被篡改
（但不能证明文件是官方发布的）。

下载 sha256sums.txt 与安装映像放在同一目录下，
运行以下命令：

```bash
sha256sum -c sha256sums.txt --ignore-missing
```

输出：

```bash
archlinux-x86_64.iso: OK
```

#### 验证 PGP 签名

Arch Linux 官方下载 PGP signature 签名文件

目的：确认文件确实来自 Arch 官方，而不是被别人替换。

##### GunPG

```bash
gpg --keyserver-options auto-key-retrieve --verify
archlinux-版本（时间）-x86_64.iso.sig
archlinux-版本（时间）-x86_64.iso
```

输出：`Primary key fingerprint` 验证指纹。

##### Arch Linux

```bash
pacman-key -v archlinux-版本-x86_64.iso.sig archlinux-版本-x86_64.iso
```

#### 准备安装介质

---

### 正式安装

#### 禁用 reflector 服务

```bash
systemctl stop reflector.service
```

查看 reflector 状态

```
systemctl status reflector.service
```

#### 禁用蜂鸣器

```bash
rmmod pcspkr
```

#### 设置字体

```bash
setfont ter-122b 
```

#### 验证引导模式

```bash
cat /sys/firmware/efi/fw_platform_size
```

#### 确认是否为 UEFI 模式

```bash
ls /sys/firmware/efi/efivars
```

#### 连接网络

```bash
iwctl # 进入交互式命令行
device list # 列出无线网卡设备名，比如无线网卡看到叫 wlan0
station wlan0 scan # 扫描网络
station wlan0 get-networks # 列出所有 wifi 网络
station wlan0 connect wifi-name # 进行连接，注意这里无法输入中文。回车后输入密码即可
exit # 连接成功后退出
```

如果有问题，请参考以下

查看内核是否加载无线网卡驱动

```bash
lspci -k | grep Network
```

如果 BIOS 没有开启无线网卡的开关可以参考下列的命令来开启 WIFI

```bash
rfkill list #查看无线连接 是否被禁用(blocked: yes)
ip link set wlan0 up #比如无线网卡看到叫 wlan0
```

若看到类似Operation not possible due to RF-kill的报错，继续尝试rfkill unblock wifi来解锁无线网卡。

```bash
rfkill unblock wifi
```

测试网络

```bash
ping  www.xxx.com （测试网络）
```

#### 启用systemd-timesyncd

systemd-timesyncd 是一个用于跨网络同步系统时钟的守护服务。

启用并启动

```bash
timedatectl set-ntp true
```

检查服务

```bash
timedatectl status
```

查看详细信息

```bash
timedatectl timesync-status
```

#### 选择镜像站

```bash
vim /etc/pacman.d/mirrorlist
```

#### 磁盘分区

- `/`：根目录
- `/home`：用户主目录
- `/boot`：efi分区
- `Swap`

列出系统中所有块设备（block devices），以树状结构显示磁盘和分区

```bash
lsblk
```

```bash
cfdisk /dev/被分区磁盘
```

首先创建 EFI 分区（Type 选EFI System）；再Linux filesystem 分区;最后 Swap 分区（Type 选Linux swap）

复查磁盘情况，列出系统中所有磁盘和分区的详细信息，包括分区类型、大小、启动标记等

```bash
fdisk -l
```

#### 磁盘格式化(make file system)

格式化 EFI 分区（一个 EFI 内多个系统不要执行此项。）

```bash
mkfs.fat -F32 /dev/efi_system_partition
```

格式化 Btrfs 分区 

```bash
mkfs.btrfs /dev/root_partition
```

格式化 Swap 分区

```
mkswap /dev/swap_partition
```
lsblk -f （查看磁盘系统）

#### 创建子卷

将 `Btrfs` 分区挂载到 `/mnt` 下：

```bash
mount -t btrfs -o compress=zstd /dev/root_partition /mnt
```

复查挂载情况

```bash
df -h
```

创建 Btrfs 子卷

```bash
btrfs subvolume create /mnt/@ # 创建 / 目录子卷
btrfs subvolume create /mnt/@home # 创建 /home 目录子卷
btrfs subvolume list -p /mnt # 复查子卷
umount /mnt # 将 /mnt 卸载掉，以挂载子卷
```

#### 挂载分区

挂载分区一定要遵循顺序，先挂载根（root）分区（到 /mnt），再挂载引导（boot）分区（到 /mnt/boot 或 /mnt/efi，如果单独分出来了的话），最后再挂载其他分区。否则您可能遇到安装完成后无法启动系统的问题

```bash
mount -t btrfs -o subvol=/@,compress=zstd /dev/nvmexn1pn /mnt # 挂载 / 目录
mkdir /mnt/home # 创建 /home 目录
mount -t btrfs -o subvol=/@home,compress=zstd /dev/nvmexn1pn /mnt/home # 挂载 /home 目录
mkdir -p /mnt/boot # 创建 /boot 目录
mount /dev/nvmexn1pn /mnt/boot # 挂载 /boot 目录
swapon /dev/nvmexn1pn # 挂载交换分区
```

复查挂载情况

```bash
df -h
```

复查 Swap 分区挂载

```bash
free -h
```

#### 安装系统

```bash
pacstrap /mnt linux(-zen/lts) linux(-zen/lst)-headers linux-firmware base base-devel btrfs-progs
```

如果提示 GPG 证书错误，可以通过更新 archlinux-keyring 解决此问题

```bash
pacman -S archlinux-keyring
```

```bash
pacstrap /mnt zsh zsh-autosuggestions zsh-syntax-highlighting zsh-completions sudo dhcpcd（base-devel疑似包含） nvim amd-ucode networkmanager
```

#### 生成 fstab(generate file system table UEFI) 文件}

生成 fstab 文件以使需要的文件系统
（如启动目录 /boot）在启动时被自动挂载

```bash
genfstab -U /mnt > /mnt/etc/fstab
# >：覆盖重定向；>>：追加重定向
```

检查fstab
```cat /mnt/etc/fstab 
```

#### 进入新系统

```bash
arch-chroot /mnt
```

```bash
setfont /usr/share/kbd/consolefonts/sun12x22.psfu.gz
```

#### 设置主机名

```bash
vim /etc/hostname # 加入你想为主机取的主机名,luck
```

```bash
vim /etc/hosts
```

加入以下内容：

```
127.0.0.1  localhost
::1        localhost
127.0.0.1  luck.localdomain  luck
```

#### 设置时区

```bash
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

#### 硬件时间设置，将当前的正确 UTC 时间写入硬件时间

```bash
hwclock --systohc
```

#### 设置 Locale 进行本地化

Locale 决定了软件使用的语言、书写习惯和字符集。

```bash
vim /etc/locale.gen
```

```
en_US.UTF-8 UTF-8 （去注释）
zh_CN.UTF-8 UTF-8
```

生成 locale

```bash
locale-gen
```

向 `/etc/locale.conf` 输入内容：

```bash
echo 'LANG=en_US.UTF-8'  > /etc/locale.conf
```

#### 设置 root 密码

```bash
passwd root
```

#### 安装引导程序

```bash
pacman -S grub os-prober efibootmgr efivar（依赖项）
```

```bash
uname -m （打印架构）
```

#### 安装 GRUB 到 EFI 分区：

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Arch
```

```bash
nvim /etc/default/grub
```

进行如下修改：

- 在 `GRUB_CMDLINE_LINUX_DEFAULT` 一行中的参数中：
  - 去掉最后的 `quiet` 参数；
  - 把 `loglevel` 的数值从 `3` 改成 `5`；
  - 加入 `nowatchdog modprobe.blacklist=sp5100_tco` 参数，这可以显著提高开关机速度。
- GRUB_TIMEOUT=2 （GRUB 引导菜单的等待时间）
- 为了引导 win10，则还需要添加新的一行 `GRUB_DISABLE_OS_PROBER=false`


生成 GRUB 所需的配置文件：

```bash
grub-mkconfig -o /boot/grub/grub.cfg
```



#### 更改shell

确保 /bin/zsh 存在并在 /etc/shells 中

ls /bin/zsh

cat /etc/shells （/etc/shells 是一个文本文件，列出了 系统允许用户设置为默认 shell 的所有“合法 shell” 路径）

\begin{verbatim}
	/bin/sh
	/bin/bash
	/bin/zsh （传统上用于系统启动的基础命令路径（必需））
	/bin/zsh （多数发行版实际安装 zsh 的地方）
\end{verbatim}

然后运行：

which zsh （输出：/usr/bin/zsh）



chsh -s /usr/bin/zsh
echo \$SHELL

#### 完成安装

```bash
exit
umount -R /mnt # 递归卸载挂载点
poweroff
```

---

## 配置 Arch Linux

### 永久禁用蜂鸣器

创建并编辑：`/etc/modprobe.d/blacklist.conf`

加入以下内容：

```conf
blacklist pcspkr
```

### 启动 networkmanager 服务

```bash
systemctl enable --now NetworkManager # 设置开机自启并立即启动 NetworkManager 服务
ping www.bilibili.com # 测试网络连接
```

若为无线连接，则需要在启动 `networkmanager` 后使用 `nmcli` 连接网络：

```bash
nmcli dev wifi list # 显示附近的 Wi-Fi 网络
nmcli dev wifi connect "Wi-Fi名（SSID）" password "网络密码" # 连接指定的无线网络
```



### 确保系统为最新

```bash
pacman -Syyu
```

vim ~/.bash_profile （配置 root 账户的默认编辑器）

```
export EDITOR='vim'
```

### 增加非 root 用户

添加 `luck` 用户（luck：用户名，自定义）

```bash
useradd -m -G wheel -s /bin/zsh luck
```

设置 `luck` 用户密码

```bash
passwd luck
```

编辑 `sudoers` 配置文件

```bash
EDITOR=nvim visudo

```

找到如下这样的一行，把前面的注释符号 # 去掉：

```
#%wheel ALL=(ALL:ALL) ALL
```

### 开启32位支持库与 Arch Linux 中文社区仓库

```bash
nvim /etc/pacman.conf
```

去掉 [multilib] 一节中两行的注释，来开启 32 位库支持

在文档结尾处加入下面的文字，来添加 archlinuxcn 源

```conf
[archlinuxcn]
Server = https://mirrors.ustc.edu.cn/archlinuxcn/$arch # 中国科学技术大学开源镜像站
Server = https://mirrors.tuna.tsinghua.edu.cn/archlinuxcn/$arch # 清华大学开源软件镜像站
Server = https://mirrors.hit.edu.cn/archlinuxcn/$arch # 哈尔滨工业大学开源镜像站
Server = https://repo.huaweicloud.com/archlinuxcn/$arch # 华为开源镜像站
```

之后执行以下命令添加farseerfc的key到系统的keyring中并更新软件源：

```bash
sudo pacman -Sy archlinuxcn-keyring
sudo pacman -S paru
```

遇到 `error: archlinuxcn-keyring: Signature from "Jiachen YANG (Arch Linux Packager Signing Key) <farseerfc@archlinux.org>" is marginal trust` 报错，执行此项：

```bash
sudo pacman-key --lsign-key "farseerfc@archlinux.org"
```

### 休眠（hibernate）设置

通过以下命令确认 Swap 分区的 UUID：

```bash
lsblk -o name,mountpoint,size,uuid
```

使用 `nvim` 编辑 `/etc/default/grub` 文件：

```bash
sudo nvim /etc/default/grub
```

将相关参数加入内核启动参数中 —— 找到 `GRUB_CMDLINE_LINUX_DEFAULT` 一行，在其值后添加类似如下数据（根据你自身的 UUID 确定，参数之间以空格分隔）：

```
resume=UUID=13ec7b86-eb9c-45a9-ae50-9606279b506a
```

通过以下命令更新 GRUB 配置：

```bash
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

除此之外，还需配置 `initranfs` 的 `resume` 钩子。使用 `vim` 编辑 `/etc/mkinitcpio.conf`：

```bash
sudo vim /etc/mkinitcpio.conf
```

在 HOOKS 行添加 `resume` 值。注意，`resume` 需要加入在 `udev` 后。若使用了 LVM 分区，`resume` 需要加入在 `lvm2` 后。

最后通过以下命令重新生成 `initramfs` 镜像：

```bash
sudo mkinitcpio -P
```

---

## 设置 DNS

```bash
nvim /etc/resolv.conf
```

删除已有条目，并将如下内容加入其中

```conf
nameserver 8.8.8.8
nameserver 2001:4860:4860::8888
nameserver 8.8.4.4
nameserver 2001:4860:4860::8844
```

\section{配置环境变量}
vim .bashrc （环境变量配置文件）
[ 
	







：

]



### 显卡驱动

#### AMD 核芯显卡

```bash
sudo pacman -S mesa lib32-mesa xf86-video-amdgpu vulkan-radeon lib32-vulkan-radeon
```

### 输入法

[Arch Wiki](https://wiki.archlinuxcn.org/wiki/Fcitx5?rdfrom=https%3A%2F%2Fwiki.archlinux.org%2Findex.php%3Ftitle%3DFcitx5_%28%25E7%25AE%2580%25E4%25BD%2593%25E4%25B8%25AD%25E6%2596%2587%29%26redirect%3Dno)

```bash
sudo pacman -S fcitx5-im # （包含：fcitx5 fcitx5-configtool fcitx5-gtk fcitx5-qt）
sudo pacman -S fcitx5-rime # 官方中文输入引擎
sudo pacman -S fcitx5-nord # 主题
paru pacman -S fcitx5-qt4-git #（AUR 通常不需要）
```
#### X11

为了告诉程序使用 fcitx5 输入法，需要设置相应的环境变量。

```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
GLFW_IM_MODULE=ibus
```



## 安装字体


---


### 功耗控制


















\section{文件系统}
\subsection{总体结构}
\begin{description}
		\item[/] 文件系统根目录
		\item[/boot] 用于启动系统的启动分区。在 EFI 系统上，这可能是 EFI 系统分区 (ESP)
		\item[/etc] 系统配置
		\item[/home] 普通用户主目录的位置
		\item[/root] root 用户的主目录。root 用户的主目录位于 /home 之外，以确保即使 /home 不可用且未挂载，root 用户也可以登录
		\item[/tmp] 存放小型临时文件的地方
\end{description}
\begin{description}
		\item[/usr] 供应商提供的操作系统资源
		\item[/usr/bin] 用户命令的二进制文件和可执行文件
		\item[/usr/share] 多个软件包之间共享的资源，例如文档、手册页、时区信息、字体和其他资源。通常，存储在此目录下的文件的确切位置和格式受确保互操作性的规范约束
		\item[/usr/share/doc] 操作系统或系统包的文档
		\item[/usr/share/factory/etc] 供应商提供的默认配置文件的存储库。此目录应填充所有可能位于 /etc/ 中的配置文件的原始供应商版本。这有助于将系统的本地配置与供应商默认配置进行比较，并使用默认配置填充本地配置
\end{description}
\section{主目录}
\begin{description}
		\item[〜/.config ] 应用程序配置。创建新用户时，此目录将为空或根本不存在。如果此目录中的配置缺失，应用程序应恢复为默认配置。如果应用程序发现\$XDG_CONFIG_HOME已设置，则应使用 \$XDG_CONFIG_HOME 中指定的目录，而不是此目录
		\item[〜/.local/bin] 应出现在用户\$PATH 搜索路径中的可执行文件
\end{description}


\section{XDG Base Directory Specification}



\section{VPN}


\section{使用}
\subsection{i3}
sudo pacman -S i3-wm i3status i3lock dmenu

sudo pacman -S xorg-server xorg-xinit xorg-xrandr xorg-xsetroot
\subsection{字体}
sudo pacman -S ttf-nerd-fonts-symbols

系统字体（所有用户） /usr/share/fonts/

用户字体（当前用户） ~/.local/share/fonts

更新字体缓存 fc-cache -fv
\subsection{tree-sitter}
sudo pacman -S tree-sitter








\subsection{挂载}
\begin{paradesc}
	\item[lsblk] 查看设备名
	\item[sudo mount /dev/sda1 /mnt/usb] 挂在u盘
	\item[sudo umount /mnt/usb] 卸载挂载
\end{paradesc}
\subsection{压缩}
\textbf{常用压缩格式与对应命令}
\begin{tabularx}{\textwidth}[t]{|l|X|l|}
\hline
格式&解压命令&安装包\\\hline
.tar&tar -xvf file.tar&已内置\\\hline
.tar.gz&tar-xvzf file.tar.gz&已内置\\\hline
.tar.xz&tar -xvJf file.tar.xz&已内置\\\hline
.gz&gunzip file.gz&gzip\\\hline
.xz&unxz file.xz&xz\\\hline
.zip&unzip file.zip&unzip\\\hline
.7z`&7z x file.7z&p7zip\\\hline
.rar&unrar x file.rar 或 unar file.rar&unrar 或 unar\\\hline
.zst&unzstd file.zst& zstd\\\hline

\end{tabularx}


## EFI 系统分区

是一个与操作系统无关的分区，其中存储了由 UEFI 固件启动的 UEFI 引导加载器、应用程序和驱动，是 UEFI 启动所必须的。
luck@lu ~ % pacman -Q  | grep fcitx                                                                         [1]
fcitx5 5.1.14-1
fcitx5-configtool 5.1.10-1
fcitx5-gtk 5.1.4-1
fcitx5-nord 0.0.0.20210727-2
fcitx5-qt 5.1.10-1
fcitx5-rime 5.1.11-1
luck@lu ~ % pacman -Q  | grep font                                                                          [0]
adobe-source-code-pro-fonts 2.042u+1.062i+1.026vf-2
adobe-source-han-sans-cn-fonts 2.005-1
adobe-source-han-serif-cn-fonts 2.003-1
adobe-source-sans-fonts 3.052-2
adobe-source-serif-fonts 4.005-2
adwaita-fonts 49.0-2
fontconfig 2:2.17.1-1
gnu-free-fonts 20120503-8
libfontenc 1.1.8-1
libxfont2 2.0.7-1
noto-fonts-emoji 1:2.048-1
texlive-fontsextra 2025.2-1
texlive-fontsrecommended 2025.2-1
texlive-fontutils 2025.2-1
ttf-nerd-fonts-symbols 3.4.0-1
ttf-nerd-fonts-symbols-common 3.4.0-1
ttf-nerd-fonts-symbols-mono 3.4.0-1
xorg-fonts-encodings 1.1.0-1
luck@lu ~ % pacman -Q  | grep xorg                                                                          [0]
xorg-fonts-encodings 1.1.0-1
xorg-server 21.1.18-2
xorg-server-common 21.1.18-2
xorg-setxkbmap 1.3.4-2
xorg-xauth 1.1.4-1
xorg-xev 1.2.6-1
xorg-xinit 1.4.4-1
xorg-xinput 1.6.4-2
xorg-xkbcomp 1.4.7-1
xorg-xmodmap 1.0.11-2
xorg-xprop 1.2.8-1
xorg-xrandr 1.5.3-1
xorg-xrdb 1.2.2-2
xorg-xset 1.2.5-2
xorg-xsetroot 1.1.3-2
xorgproto 2024.1-2
luck@lu ~ % pacman -Q  | grep i3                                                                            [0]

i3-wm 4.24-1



aHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQDFzZzAwMS40NzY1NzkueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPTFzZzAwMS40NzY1NzkueHl6IyVFNSU4OSVBOSVFNCVCRCU5OSVFNiVCNSU4MSVFOSU4NyU4RiVFRiVCQyU5QTQwLjc1JTIwR0INCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUAxc2cwMDEuNDc2NTc5Lnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT0xc2cwMDEuNDc2NTc5Lnh5eiMlRTglQjclOUQlRTclQTYlQkIlRTQlQjglOEIlRTYlQUMlQTElRTklODclOEQlRTclQkQlQUUlRTUlODklQTklRTQlQkQlOTklRUYlQkMlOUExNSUyMCVFNSVBNCVBOQ0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQDFzZzAwMS40NzY1NzkueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPTFzZzAwMS40NzY1NzkueHl6IyVFNSVBNSU5NyVFOSVBNCU5MCVFNSU4OCVCMCVFNiU5QyU5RiVFRiVCQyU5QTIwMjYtMDYtMDQNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUAxc2cwMDEuNDc2NTc5Lnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT0xc2cwMDEuNDc2NTc5Lnh5eiMlRTYlOTYlQjAlRTUlOEElQTAlRTUlOUQlQTEwMWF3cw0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQGhrbWF4MDIuMzUyMzQzLmNjOjgwODAvP2luc2VjdXJlPTAmc25pPWhrbWF4MDIuMzUyMzQzLmNjIyVFNiU5NiVCMCVFNSU4QSVBMCVFNSU5RCVBMTAyDQp2bGVzczovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUB0dzEuMzQ0MjMzLmNjOjg0NDM/dHlwZT10Y3AmZW5jcnlwdGlvbj1ub25lJmhvc3Q9JnBhdGg9JmhlYWRlclR5cGU9bm9uZSZxdWljU2VjdXJpdHk9bm9uZSZzZXJ2aWNlTmFtZT0mbW9kZT1ndW4mc2VjdXJpdHk9cmVhbGl0eSZmbG93PXh0bHMtcnByeC12aXNpb24mZnA9Y2hyb21lJnNuaT13d3cucHl0aG9uLm9yZyZwYms9MmEwT05MUmlCZUhKZHI5cUNydXE1dFBWZjhfM2M0Zm1ac2c3WVFvckZTRSZzaWQ9MDRkNTkzNDAjdmxlc3MlRTUlOEYlQjAlRTYlQjklQkUwMQ0Kdmxlc3M6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAdHcwNi4zNTIzNDMuY2M6ODQ0Mz90eXBlPXRjcCZlbmNyeXB0aW9uPW5vbmUmaG9zdD0mcGF0aD0maGVhZGVyVHlwZT1ub25lJnF1aWNTZWN1cml0eT1ub25lJnNlcnZpY2VOYW1lPSZtb2RlPWd1biZzZWN1cml0eT1yZWFsaXR5JmZsb3c9eHRscy1ycHJ4LXZpc2lvbiZmcD1jaHJvbWUmc25pPXd3dy5weXRob24ub3JnJnBiaz0yYTBPTkxSaUJlSEpkcjlxQ3J1cTV0UFZmOF8zYzRmbVpzZzdZUW9yRlNFJnNpZD0wNGQ1OTM0MCN2bGVzcyVFNSU4RiVCMCVFNiVCOSVCRTAyDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFANWd6ZHguMjMzMjM1Lnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT01Z3pkeC4yMzMyMzUueHl6I2h5MiVFNSU4RiVCMCVFNiVCOSVCRTAxDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAdHc2MC4yMzMyMzUueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPXR3NjAuMjMzMjM1Lnh5eiNoeTIlRTUlOEYlQjAlRTYlQjklQkUwMg0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQHR3My43NTc4NjYueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPXR3My43NTc4NjYueHl6I2h5MiVFNSU4RiVCMCVFNiVCOSVCRTAzDQp0cm9qYW46Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAdHc3LjM0NDIzMy5jYzozNjYzP2FsbG93SW5zZWN1cmU9MCZwZWVyPXR3Ny4zNDQyMzMuY2Mmc25pPXR3Ny4zNDQyMzMuY2MmdHlwZT10Y3AjdHJvamFuJUU1JThGJUIwJUU2JUI5JUJFMDENCnRyb2phbjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUB0dzUuNDUzNTIxLnh5ejozNjYzP2FsbG93SW5zZWN1cmU9MCZwZWVyPXR3NS40NTM1MjEueHl6JnNuaT10dzUuNDUzNTIxLnh5eiZ0eXBlPXRjcCN0cm9qYW4lRTUlOEYlQjAlRTYlQjklQkUwMg0KdHJvamFuOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQHR3NC40NTM1MjEueHl6OjM2NjM/YWxsb3dJbnNlY3VyZT0wJnBlZXI9dHc0LjQ1MzUyMS54eXomc25pPXR3NC40NTM1MjEueHl6JnR5cGU9dGNwI3Ryb2phbiVFNSU4RiVCMCVFNiVCOSVCRTAzDQp0cm9qYW46Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAdHcwNy4zNTIzNDMuY2M6MzY2Mz9hbGxvd0luc2VjdXJlPTAmcGVlcj10dzA3LjM1MjM0My5jYyZzbmk9dHcwNy4zNTIzNDMuY2MmdHlwZT10Y3AjdHJvamFuJUU1JThGJUIwJUU2JUI5JUJFMDQNCnRyb2phbjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUB0dzExLjM1MjM0My5jYzozNjYzP2FsbG93SW5zZWN1cmU9MCZwZWVyPXR3MTEuMzUyMzQzLmNjJnNuaT10dzExLjM1MjM0My5jYyZ0eXBlPXRjcCN0cm9qYW4lRTUlOEYlQjAlRTYlQjklQkUwNQ0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQHR3NDAuNDUzNTIxLnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT10dzQwLjQ1MzUyMS54eXojaHkyJUU1JThGJUIwJUU2JUI5JUJFMDcNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUB2c2cwNS4zNDQyMzMuY2M6ODA4MC8/aW5zZWN1cmU9MCZzbmk9dnNnMDUuMzQ0MjMzLmNjIyVFNiU5NiVCMCVFNSU4QSVBMCVFNSU5RCVBMTAyYXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAc2cwMy4zNDQyMzMuY2M6ODA4MC8/aW5zZWN1cmU9MCZzbmk9c2cwMy4zNDQyMzMuY2MjJUU2JTk2JUIwJUU1JThBJUEwJUU1JTlEJUExMDNhd3MNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBzZzA0LjM1MjM0My5jYzo4MDgwLz9pbnNlY3VyZT0wJnNuaT1zZzA0LjM1MjM0My5jYyMlRTYlOTYlQjAlRTUlOEElQTAlRTUlOUQlQTEwNGF3cw0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQHNnMDUuNDUzNTIxLnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT1zZzA1LjQ1MzUyMS54eXojJUU2JTk2JUIwJUU1JThBJUEwJUU1JTlEJUExMDVhd3MNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBzZzA2LjQ1MzUyMS54eXo6ODA4MC8/aW5zZWN1cmU9MCZzbmk9c2cwNi40NTM1MjEueHl6IyVFNiU5NiVCMCVFNSU4QSVBMCVFNSU5RCVBMTA2YXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAc2cwNy4zNDQyMzMuY2M6ODA4MC8/aW5zZWN1cmU9MCZzbmk9c2cwNy4zNDQyMzMuY2MjJUU2JTk2JUIwJUU1JThBJUEwJUU1JTlEJUExMDdhd3MNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUB2c2cwNC4zNDQyMzMuY2M6ODA4MC8/aW5zZWN1cmU9MCZzbmk9dnNnMDQuMzQ0MjMzLmNjIyVFNiU5NiVCMCVFNSU4QSVBMCVFNSU5RCVBMTA4YXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAc2cwOS40NTM1MjEueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPXNnMDkuNDUzNTIxLnh5eiMlRTYlOTYlQjAlRTUlOEElQTAlRTUlOUQlQTEwOWF3cw0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQHNnMTAuMzUyMzQzLmNjOjgwODAvP2luc2VjdXJlPTAmc25pPXNnMTAuMzUyMzQzLmNjIyVFNiU5NiVCMCVFNSU4QSVBMCVFNSU5RCVBMTEwYXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAc2cxMS4zNTIzNDMuY2M6ODA4MC8/aW5zZWN1cmU9MCZzbmk9c2cxMS4zNTIzNDMuY2MjJUU2JTk2JUIwJUU1JThBJUEwJUU1JTlEJUExMTFhd3MNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBzZzEzLjM1MjM0My5jYzo4MDgwLz9pbnNlY3VyZT0wJnNuaT1zZzEzLjM1MjM0My5jYyMlRTYlOTYlQjAlRTUlOEElQTAlRTUlOUQlQTExM2F3cw0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQHNnMTguMzUyMzQzLmNjOjgwODAvP2luc2VjdXJlPTAmc25pPXNnMTguMzUyMzQzLmNjIyVFNiU5NiVCMCVFNSU4QSVBMCVFNSU5RCVBMTE4YXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAaGttYXgwMS4zNzY4OTcueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPWhrbWF4MDEuMzc2ODk3Lnh5eiMlRTklQTYlOTklRTYlQjglQUYwMQ0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQGhrbWF4MDIuMzQ0MjMzLmNjOjgwODAvP2luc2VjdXJlPTAmc25pPWhrbWF4MDIuMzQ0MjMzLmNjIyVFOSVBNiU5OSVFNiVCOCVBRjAzYXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAMWpwMDAzLjE1Njc4Ni54eXo6ODA4MS8/aW5zZWN1cmU9MCZzbmk9MWpwMDAzLjE1Njc4Ni54eXojJUU2JTk3JUE1JUU2JTlDJUFDMDENCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBoeWZuanAwMS4xNTY3ODYueHl6OjgwODEvP2luc2VjdXJlPTAmc25pPWh5Zm5qcDAxLjE1Njc4Ni54eXojJUU2JTk3JUE1JUU2JTlDJUFDMDINCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBqcDAzLjM0NDIzMy5jYzo4MDgwLz9pbnNlY3VyZT0wJnNuaT1qcDAzLjM0NDIzMy5jYyMlRTYlOTclQTUlRTYlOUMlQUMwM0wNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBoeWZuanAwMy4zNTIzNDMuY2M6ODA4MC8/aW5zZWN1cmU9MCZzbmk9aHlmbmpwMDMuMzUyMzQzLmNjIyVFNiU5NyVBNSVFNiU5QyVBQzA0TA0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQGh5Zm5qcDA0LjM0NDIzMy5jYzo4MDgwLz9pbnNlY3VyZT0wJnNuaT1oeWZuanAwNC4zNDQyMzMuY2MjJUU2JTk3JUE1JUU2JTlDJUFDMDFMDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAaHlmbmpwMDcuMTU2Nzg2Lnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT1oeWZuanAwNy4xNTY3ODYueHl6IyVFNiU5NyVBNSVFNiU5QyVBQzAyTA0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQDFqcDAwMS40NzY1NzkueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPTFqcDAwMS40NzY1NzkueHl6IyVFNiU5NyVBNSVFNiU5QyVBQzAzYXdzDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAaHlmbmpwMDYuMzc2ODk3Lnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT1oeWZuanAwNi4zNzY4OTcueHl6IyVFNiU5NyVBNSVFNiU5QyVBQzA1TA0KaHlzdGVyaWEyOi8vZDc0MWIzNGYtNWIwNy00MDEwLTlmMjYtMGVjMjhjN2I2ZmYxQGh5Zm51czAxLjU5NTc4MC54eXo6ODA4MC8/aW5zZWN1cmU9MCZzbmk9aHlmbnVzMDEuNTk1NzgwLnh5eiMlRTclQkUlOEUlRTUlOUIlQkQwMUwNCmh5c3RlcmlhMjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUBoeWZudXMwMi43Njc4ODcueHl6OjgwODAvP2luc2VjdXJlPTAmc25pPWh5Zm51czAyLjc2Nzg4Ny54eXojJUU3JUJFJThFJUU1JTlCJUJEMDJMDQpoeXN0ZXJpYTI6Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAaHlmbnVzMDQuNzY3ODg3Lnh5ejo4MDgwLz9pbnNlY3VyZT0wJnNuaT1oeWZudXMwNC43Njc4ODcueHl6IyVFNyVCRSU4RSVFNSU5QiVCRDAzDQp0cm9qYW46Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAMS4xLjEuMTo4MDgwP2FsbG93SW5zZWN1cmU9MCZwZWVyPTEuMS4xLjEmc25pPTEuMS4xLjEmdHlwZT10Y3AjJUU2JTlDJTg5NDAlRTUlQTQlOUElRTQlQjglQUElRTglOEElODIlRTclODIlQjklRUYlQkMlOEMlRTQlQjglOEQlRTUlQTQlOUYlRTglQUYlQjclRTUlODglQjAlRTUlQUUlOTglRTclQkQlOTElRTQlQkQlQkYlRTclOTQlQTglRTYlOTYlODclRTYlQTElQTMlRUYlQkMlOEMlRTQlQjglOEIlRTglQkQlQkQlRTYlOUMlODAlRTYlOTYlQjAlRTclOUElODQlRTUlQUUlQTIlRTYlODglQjclRTclQUIlQUYNCnRyb2phbjovL2Q3NDFiMzRmLTViMDctNDAxMC05ZjI2LTBlYzI4YzdiNmZmMUAyc2cxM2NiMmI2YjRkLTliMzItMDFhOC03NTc0LTM3NmExNzFiYjgzMi4xNzAyMDMueHl6OjgyMD9hbGxvd0luc2VjdXJlPTAmcGVlcj0yc2cxM2NiMmI2YjRkLTliMzItMDFhOC03NTc0LTM3NmExNzFiYjgzMi4xNzAyMDMueHl6JnNuaT0yc2cxM2NiMmI2YjRkLTliMzItMDFhOC03NTc0LTM3NmExNzFiYjgzMi4xNzAyMDMueHl6JnR5cGU9dGNwIyVFNyU5NCVCNSVFNiU4QSVBNSVFNyVCRSVBNGh0dHBzJTNBJTJGJTJGdC5tZSUyRmZlaW5pYW95dW5qaWNoYW5nDQp0cm9qYW46Ly9kNzQxYjM0Zi01YjA3LTQwMTAtOWYyNi0wZWMyOGM3YjZmZjFAMi4yLjIuMjo4MjA/YWxsb3dJbnNlY3VyZT0wJnBlZXI9Mi4yLjIuMiZzbmk9Mi4yLjIuMiZ0eXBlPXRjcCMlRTklOTglQjIlRTUlQTQlQjElRTglODElOTQlRTklQTElQjVodHRwcyUzQSUyRiUyRmdpdGh1Yi5jb20lMkZmZWluaWFveXVuDQo=
