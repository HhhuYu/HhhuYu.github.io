---
title: wsl2环境配置+docker+wt+vscode
categories:
  - tool
tags:
  - docker
  - wsl2
  - zsh
  - wt
mathjax: false
abbrlink: 4258143485
date: 2020-07-19 15:28:28
---


## 序言

最近新换的电脑又坏了，拿回来还得重新配置环境，还得挨个找教程进行配置。这次就简单地全写一起吧。

作为`wsl`的忠实爱好者，我已经使用了`wsl`两年了，相比虚拟机来说，实在是不要方便太多了，而且配置要求也没这么高。

之前`windows` 商店也发布了`wt`(`windows terminal`)，相比`cmd`和`powershell`以及原本`bash`漂亮太多了，而且集成了多个窗口，切换很方便。

这是我现在的界面，大家可以自己定义其他的外观等等。

![overview](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-overview.png)


最近看了一下，居然`wsl2`已经出好久了，还支持`docker`。我居然才知道，我想了想那时我应该在准备考研，emmm 。好了，废话不多说我们开始配置。

另外说一句，`wsl2`还没有推送到正式版的`windows 10`中，目前还是在`windows`的测试版中。

<!--more-->

### 对比

[比较 WSL 2 和 WSL 1](https://docs.microsoft.com/zh-cn/windows/wsl/compare-versions)

`wsl1`和`wsl2`底层架构不同，所以`wsl2`能相比`wsl1`更多的实现一些`linux`内的功能。包括这次的`docker`，同时在`windows 10`最新的测试版中已经提供了GPU计算，有兴趣的可以自己升到最新的版本使用。


## 开始配置

### 更新windows 10

`wsl2`只有在`windows 10`版本2004的内部版本19041或者更高的版本中有提供 

- 你可以`win+R`中输入`winver`中查看windows的版本
  - ![winver](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-winver.png)
- 如果你的版本没有达到要求的话，你可以先试图通过windows 10的系统更新。
- 如果依旧没达到版本要求的话：
  - 加入微软的`windows 预览体院计划`，并选择Beta渠道
  - ![preview](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-preview.png)
  - 进行windows 10的系统更新


### 系统设置


#### 如果你没有安装过wsl
那么，你必须先启用`适用于 Linux 的 Windows 子系统`功能。

以管理员身份打开 PowerShell 并运行：

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
```
并重启电脑

#### 如果你安装过wsl

安装 WSL 2 之前，必须启用`虚拟机平台`功能。

以管理员身份打开 PowerShell 并运行：

```powershell
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```

重新启动计算机，以完成 WSL 安装并更新到 WSL 2。


#### 将wsl2设置为默认版本

在powershell中输入：
```powershell
wsl --set-default-version 2
```

你可能会遇到错误信息：`WSL 2 requires an update to its kernel component. For information please visit https://aka.ms/wsl2kernel`
请安装[wsl2kernel](https://aka.ms/wsl2kernel)。

#### wsl1 to wsl2

先查看wsl的版本信息
```powershell
wsl --list --verbose
# or
wsl -l -v
```


如果你是版本1的话，切换至版本2
```powershell
wsl --set-version <distro-name> 2
# in my case
wsl --set-version Ubuntu 2
```

从wsl1切换到wsl2可能会有错误，可能这能够解决[卸载并删除任何旧分发](https://docs.microsoft.com/zh-cn/windows/wsl/install-legacy#uninstallingremoving-the-legacy-distro)

### 安装 linux 发行版与 windows terminal

在`Microsoft Store`搜索`windows terminal`与想要的`linux` 发行版


![windows-ternimal](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-windows-terminal.png)


你可以搜索你想要的 linux 发行版，基本上都有
![disto](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-disto.png)

- Ubuntu 16.04 LTS
- Ubuntu 18.04 LTS
- Ubuntu 20.04 LTS
- openSUSE Leap 15.1
- SUSE Linux Enterprise Server 12 SP5
- SUSE Linux Enterprise Server 15 SP1
- Kali Linux
- Debian GNU/Linux
- Fedora Remix for WSL
- Pengwin
- Pengwin Enterprise
- Alpine WSL


### linux环境配置

我安装的是`ubuntu18.04`，所以接下来的流程都在该发行版下进行。

在使用`wt`中使用发行版时，先启动该发行版完成预安装和预配置。完成安装之后便可以通过`win+r`输入`wt`，打开`windows ternimal`点击`+`号选择你安装的发行版，就可以了，`wt`的配置我会在后面贴出。

#### 换源

```shell
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo vim /etc/apt/sources.list
```

清空里面所有内容，换入国内源。

例如阿里源：

```
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
```

更新source包，安装更新，开发基础内容包
```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install build-essential
```

#### zsh

将zsh替代bash
```bash
sudo apt install zsh # 安装zsh
```

更改默认的shell
```bash
chsh -s $(which zsh)
```

下次再打开分发版时默认就是zsh


#### Oh My ZSH
`oh my zsh`是一个开源的社区驱动的框架，用于管理zsh配置。

下载安装：
```
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

zsh的配置文件在
```
vim ~/.zshrc
```

可以更改主题

```zsh
# 默认主题
ZSH_THEME="robbyrussell"
# 新主题
ZSH_THEME="agnoster"
```

可以安装插件`Syntax Highlighting`, `Auto Suggestions`
分别是语法高亮和自动补齐

```bash
mkdir ~/.zsh
cd ~/.zsh
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
git clone https://github.com/zsh-users/zsh-autosuggestions.git
```

同时`wt`里面文件夹名比较丑，下载`dircolors`

```bahs
curl https://raw.githubusercontent.com/seebi/dircolors-solarized/master/dircolors.ansi-dark --output ~/.dircolors
```


然后
```bash
vim ~/.zshrc
```

在末尾加上
```bash
source ~/.zsh/zsh-autosuggestions/zsh-autosuggestions.zsh
source ~/.zsh/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh 
eval `dircolors ~/.dircolors`
```
这样的话在每一次开启终端，都会启用插件


#### proxychains

在wsl2中的网络有了新的变化，系统会生成一个虚拟网卡，也就是说，在windows上会多一个网卡，且Ip是变化的。

所以在linux中它的`hosts`是自动生成的、沿袭原本windows内容的`hosts`，`nameserver`也是一样的。

所以window上的小飞机是无法直接给`wsl2`使用的，但是小飞机可以走本地端口代理。

而proxychains就是这样以一个工具可以利用http协议，走1080端口本地代理。

```zsh
sudo apt-get install proxychains
```
它的配置文件在
```zsh
cat /etc/proxychains.conf
```

因为wsl2的ip是变化的，host的ip也是变化的
所以要让`proxychains.conf`的代理信息自动变化


```zsh
cd ~/.zsh
mkdir proxy
cd proxy
vim proxy.sh
```
输入以下内容

```bash
#!/bin/sh
export HOSTIP=$(cat /etc/resolv.conf | grep 'nameserver' | cut -f 2 -d ' ')
export WSLIP=$(ip addr show eth0 | grep 'inet ' | cut -f 6 -d ' ' | cut -f 1 -d '/')
echo HOSTIP $HOSTIP
echo WSLIP $WSLIP

echo Your password | sudo -S sed -i "/http/c http\t$HOSTIP 1080" /etc/proxychains.conf
```
保存退出后

```bash
vim ~/.zshrc
```
输入
```bash
source ~/.zsh/proxy/proxy.sh
```

以后只要想要下载某些外网内容就可以直接在前面加`proxychains`

```bash
# 例如
sudo proxychains apt-get update
sudo proxychains apt-get upgrade 
proxychian git clone ......
```


#### Nodejs

方法一：利用n安装nodejs
```bash
sudo apt-get install nodejs
sudo apt-get install npm
sudo npm install -g npm
sudo npm install -g n
sudo n stable //稳定版
```

方法二：利用nvm管理nodejs

```
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.34.0/install.sh | bash
```

之后重启`wt`

输入
```bash
command -v nvm
# return nvm

# 安装node
nvm install node # 最新版本
nvm use node
```

#### Docker

```bash
# 更新索引包依赖
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common

# 添加官方的GPG-key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -


# 验证
sudo apt-key fingerprint 0EBFCD88
# 答案 9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88

# 添加 Docker repository 
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

# 安装
sudo apt update
sudo apt install docker-ce

# 启动docker
sudo service docker start


# 测试docker
sudo docker run hello-world

```

#### Docker Compose

docker容器的管理工具。

```bash
# 下载当前的稳定发行版并将其放在/usr/local/bin下。
sudo curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# 修改权限
sudo chmod +x /usr/local/bin/docker-compose

# 测试
docker-compose --version
```

#### Docker Desktop

在windows上下载
[Docker Desktop](https://www.docker.com/products/docker-desktop)

提供了docker的gui

![docker-desktop](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-docker-desktop.png)



### 连接vscode

请先下载vscode安装插件：`remote-wsl`和`terminal`。

![remote-wsl](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-remote-wsl.png)
![terminal-plugin](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-terminal-plugin.png)

这样你就可以通过点击![](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-connection.png)连接wsl

通过`ctrl+p`输入terminal，选择打开一个内部的teriminal。就可以了。
![](https://cdn.jsdelivr.net/gh/charstal/images/hexo/wsl2-wt-zsh-docker-vscode-shell.png)

修改默认的shell为zsh。


#### 字体问题

你可能会遇到某些字无法正常显示。

下载[DejaVuSansMono](https://github.com/powerline/fonts/blob/master/DejaVuSansMono/DejaVu%20Sans%20Mono%20for%20Powerline.ttf)，并安装。

同时在vscode的设置中，将terminal的字体选择为这个名字。


### windows terminal

这个是我的配置，不做过多介绍了
```json
// This file was initially generated by Windows Terminal 1.0.1401.0
// It should still be usable in newer versions, but newer versions might have additional
// settings, help text, or changes that you will not see unless you clear this file
// and let us generate a new one for you.

// To view the default settings, hold "alt" while clicking on the "Settings" button.
// For documentation on these settings, see: https://aka.ms/terminal-documentation
{
    "$schema": "https://aka.ms/terminal-profiles-schema",
    
    "defaultProfile": "{c6eaf9f4-32a7-5fdc-b5cf-066e8a4b1e40}",

    // You can add more global application settings here.
    // To learn more about global settings, visit https://aka.ms/terminal-global-settings

    // If enabled, selections are automatically copied to your clipboard.
    "copyOnSelect": false,

    // If enabled, formatted data is also copied to your clipboard
    "copyFormatting": false,
    
    // A profile specifies a command to execute paired with information about how it should look and feel.
    // Each one of them will appear in the 'New Tab' dropdown,
    //   and can be invoked from the commandline with `wt.exe -p xxx`
    // To learn more about profiles, visit https://aka.ms/terminal-profile-settings
    "profiles":
    {
        "defaults":
        {
            "fontFace": "DejaVu Sans Mono for Powerline",
            "acrylicOpacity": 0.8, //背景透明度
            "useAcrylic": true, // 启用毛玻璃
            // "backgroundImage": "D:\\OneDrive\\图片\\stack.jpg", //背景图片
            // "backgroundImageOpacity": 0.5, //图片透明度
            // "backgroundImageStretchMode": "fill", //填充模式
            // "icon": "ms-appx:///ProfileIcons/{9acb9455-ca41-5af7-950f-6bca1bc9722f}.png", //图标
            "fontSize": 10, //文字大小
            "colorScheme": "Solarized Dark", //主题
            "cursorColor": "#FFFFFF", //光标颜色
            "cursorShape": "bar" //光标形状
            // "startingDirectory":"D://Projects//" //起始目录
            // Put settings here that you want to apply to all profiles.
        },
        "list":
        [
            {
                // Make changes here to the powershell.exe profile.
                "guid": "{61c54bbd-c2c6-5271-96e7-009a87ff44bf}",
                "name": "Windows PowerShell",
                "commandline": "powershell.exe",
                "hidden": false
            },
            {
                // Make changes here to the cmd.exe profile.
                "guid": "{0caa0dad-35be-5f56-a8ff-afceeeaa6101}",
                "name": "命令提示符",
                "commandline": "cmd.exe",
                "hidden": false
            },
            {
                "guid": "{c6eaf9f4-32a7-5fdc-b5cf-066e8a4b1e40}",
                "hidden": false,
                "name": "Ubuntu-18.04",
                "source": "Windows.Terminal.Wsl"
            },
            {
                "guid": "{b453ae62-4e3d-5e58-b989-0a998ec441b8}",
                "hidden": false,
                "name": "Azure Cloud Shell",
                "source": "Windows.Terminal.Azure"
            },
            {
                "guid": "{07b52e3e-de2c-5db4-bd2d-ba144ed6c273}",
                "hidden": false,
                "name": "Ubuntu-20.04",
                "source": "Windows.Terminal.Wsl"
            }
        ]
    },

    // Add custom color schemes to this array.
    // To learn more about color schemes, visit https://aka.ms/terminal-color-schemes
    "schemes": [
        {
            "name": "Solarized Dark",
            "black": "#002831",
            "red": "#d11c24",
            "green": "#738a05",
            "yellow": "#a57706",
            "blue": "#2176c7",
            "purple": "#c61c6f",
            "cyan": "#259286",
            "white": "#eae3cb",
            "brightBlack": "#475b62",
            "brightRed": "#bd3613",
            "brightGreen": "#475b62",
            "brightYellow": "#536870",
            "brightBlue": "#708284",
            "brightPurple": "#5956ba",
            "brightCyan": "#819090",
            "brightWhite": "#fcf4dc",
            "background": "#001e27",
            "foreground": "#708284"
          },
          {
            "name": "Solarized Darcula",
            "black": "#25292a",
            "red": "#f24840",
            "green": "#629655",
            "yellow": "#b68800",
            "blue": "#2075c7",
            "purple": "#797fd4",
            "cyan": "#15968d",
            "white": "#d2d8d9",
            "brightBlack": "#25292a",
            "brightRed": "#f24840",
            "brightGreen": "#629655",
            "brightYellow": "#b68800",
            "brightBlue": "#2075c7",
            "brightPurple": "#797fd4",
            "brightCyan": "#15968d",
            "brightWhite": "#d2d8d9",
            "background": "#3d3f41",
            "foreground": "#d2d8d9"
          }        
    ],

    // Add custom keybindings to this array.
    // To unbind a key combination from your defaults.json, set the command to "unbound".
    // To learn more about keybindings, visit https://aka.ms/terminal-keybindings
    "keybindings":
    [
        // Copy and paste are bound to Ctrl+Shift+C and Ctrl+Shift+V in your defaults.json.
        // These two lines additionally bind them to Ctrl+C and Ctrl+V.
        // To learn more about selection, visit https://aka.ms/terminal-selection
        { "command": {"action": "copy", "singleLine": false }, "keys": "ctrl+c" },
        { "command": "paste", "keys": "ctrl+v" },

        // Press Ctrl+Shift+F to open the search box
        { "command": "find", "keys": "ctrl+shift+f" },

        // Press Alt+Shift+D to open a new pane.
        // - "split": "auto" makes this pane open in the direction that provides the most surface area.
        // - "splitMode": "duplicate" makes the new pane use the focused pane's profile.
        // To learn more about panes, visit https://aka.ms/terminal-panes
        { "command": { "action": "splitPane", "split": "auto", "splitMode": "duplicate" }, "keys": "alt+shift+d" }
    ]
}

```


## 参考

[WSL2, zsh, and docker. Linux through Windows.](https://nickymeuleman.netlify.app/blog/linux-on-windows-wsl2-zsh-docker#powerline-fonts)
[WSL2中自动配置Windows IP地址](https://www.oyohyee.com/post/note_wsl2_net)
[适用于 Linux 的 Windows 子系统安装指南 (Windows 10)](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10#update-to-wsl-2)