# 

Node.js 是基于 Chrome V8 引擎的 JavaScript 运行环境，让我们可以在服务端运行 JavaScript 代码，也是前端开发、后端项目搭建必不可少的工具。本篇教程手把手教你在**Windows、Mac、Linux**三大系统安装 Node.js，全程零基础可操作，看完就能顺利安装成功。

## 一、Node.js 版本选择

Node.js 分为两个版本，新手建议直接选**LTS 长期支持版**，更稳定、兼容性更好，适合开发和生产环境；Current 为最新尝鲜版，适合体验新特性，不建议新手使用。

- **LTS 版**：稳定、bug 少、长期维护，优先选择

- **Current 版**：功能最新，可能存在兼容性问题

## 二、Windows 系统安装 Node.js

### 1. 下载安装包

1. 打开 Node.js 官方下载页面：[https://nodejs.org/zh-cn/download/](https://nodejs.org/zh-cn/download/)

2. 在下载页面中，选择 **Windows 安装包（.msi 64位）**，自动匹配系统版本，点击下载。

### 2. 安装步骤

1. 双击下载好的 `.msi` 安装文件，点击【Next】进入安装界面

2. 勾选【I accept the terms in the License Agreement】同意协议，点击【Next】

3. 选择安装路径，**建议默认路径**，无需修改，避免环境变量配置出错，点击【Next】

4. 安装组件页面，**保持默认勾选**，包含 Node.js 核心程序、npm 包管理器、环境变量自动配置，直接点击【Next】

5. 无需勾选【Automatically install the necessary tools】（新手可跳过），直接点击【Install】开始安装

6. 等待安装完成，点击【Finish】结束安装

### 3. 验证安装是否成功

1. 按下 `Win + R`，输入 `cmd` 打开命令提示符

2. 分别输入以下两条命令，查看版本号：

```Bash

node -v
npm -v
```

1. 若能正常显示 Node.js 和 npm 版本号，说明安装成功！

## 三、Mac 系统安装 Node.js

### 方法1：图形化安装包安装（推荐新手）

1. 打开 Node.js 中文官网，下载 **Mac 安装包（.pkg）**

2. 双击下载的 `.pkg` 文件，按照提示点击【继续】【同意】

3. 选择安装磁盘，默认即可，点击【安装】，输入电脑密码授权安装

4. 安装完成后，关闭安装程序

### 方法2：Homebrew 安装（适合有Homebrew用户）

1. 打开终端，执行以下命令安装 Homebrew（已安装可跳过）：

```Bash

/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

1. 执行命令安装 Node.js：

```Bash

brew install node
```

### 验证安装

打开 Mac 终端，输入命令查看版本：

```Bash

node -v
npm -v
```

显示版本号即安装成功。

## 四、Linux 系统（Ubuntu/Debian）安装 Node.js

1. 打开终端，先更新系统软件源：

```Bash

sudo apt update
```

1. 安装 Node.js 和 npm：

```Bash

sudo apt install nodejs npm -y
```

1. 验证安装：

```Bash

node -v
npm -v
```

## 五、npm 镜像源切换（解决下载慢问题）

默认 npm 镜像源在国外，下载依赖包速度很慢，建议切换为**淘宝镜像源**：

1. 打开终端/命令提示符，执行以下命令：

```Bash

npm config set registry https://registry.npmmirror.com/
```

1. 验证镜像源是否切换成功：

```Bash

npm config get registry
```

显示 `https://registry.npmmirror.com/` 即切换完成。

## 六、常见问题解决

1. **输入命令提示“不是内部或外部命令”**

原因：环境变量未自动配置，重新运行安装包，勾选【Add to PATH】选项，重新安装即可。

1. **npm 下载依赖报错**

解决：切换国内镜像源，或检查网络，重启终端后重试。

1. **Mac/Linux 权限不足报错**

解决：命令前加 `sudo` 获取管理员权限，输入密码即可。

## 七、总结

至此，Node.js 已经成功安装到你的电脑中，同时自带的 npm 包管理器也可以直接使用，后续即可开始搭建前端项目、开发服务端程序啦！
