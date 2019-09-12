---
layout: post
title: "Travis CI Push 权限配置"
date: 2019-01-13 22:46:38
categories: Travis
---
<!-- more -->

要 Travis 向 Github 仓库 Push 的权限，需要由 SSH 协议机制来保证安全性。现在 Github 开放免费私有库使用，Travis Push 私有库通过 SSH 的方式比较方便安全。下面简单讲解一下如何配置。

## 基本原理

首先，了解一下 git 使用 SSH 协议向 Github 仓库 Push 时采用的 **公钥加密** 机制。

1. Github 保存本机生成的 **SSH公钥**
2. git 客户端发起 Push 请求，Github 接收到后向客户端发送一个随机字符串
3. git 客户端接收到后，使用本地的 **SSH私钥** 对字符串进行加密，然后发送回 Github
4. Github 使用存储的公钥对字符串进行解密，若和之前发送的字符串匹配，则验证通过。

在 Github 中，可以添加用户全局的公钥，对用户的所有仓库具有权限；也可以对每一个仓库单独设置公钥，使得该公钥只能在该仓库的验证步骤中生效，即后文我们会用到的 Deploy Key。

要使得 Travis 能够向 Github 推送，用户需要以某种形式把本地的 **SSH私钥** 发送给 Travis，直接发送显然是不安全的，于是 Travis 把用户的 SSH 私钥再次加密，然后每次在 CI 容器中解密私钥，再用私钥来完成推送前的验证步骤，加密私钥的过程可以通过其客户端 `travis` 来完成。

## 步骤概览

1. 在 Github 上创建一个仓库；
2. 为该仓库开启 [Travis CI](https://travis-ci.com/)；
3. 本地生成公钥并添加到仓库的 Deploy Key；
4. 克隆仓库到本地，并进入仓库目录；
5. 使用 `travis` 客户端加密 SSH 私钥；
6. 创建 CI 相关文件；
7. 推送仓库，触发 CI 并完成配置。

下面对钥匙生成和添加、仓库克隆、加密私钥和配置文件进行详细说明。

### 钥匙生成

生成 RSA Key Pair（使用你的邮箱替换命令中的邮箱）

```
$ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

```

连续回车三次，会在 `~/.ssh/` 目录中生成密钥对，其中 `~/.ssh/id_rsa.pub` 即是需要添加到仓库的 Deploy Key 的公钥。`cat ~/.ssh/id_rsa.pub`，然后选择并复制公钥内容，添加到仓库的 Deploy Key。

设置 Deploy Key 的页面为 `https://github.com/<your name>/<your repo>/settings/keys`，将 `your name` 和 `your name` 分别替换为你的用户名和仓库名。注意需要选定 `Allow Write Access`，开启该公钥对仓库的写权限。

### 克隆仓库

```
$ git clone git@github.com:<username>/<reponame>.git

```

分别替换 `<username>` 和 `<reponame>` 为用户名和仓库名。

克隆完成后 `cd` 进入仓库目录。

### 加密私钥

首先安装 `travis` 客户端（假设系统已安装 Ruby）。

```
$ gem install travis

```

然后使用 Github 账户信息登录

```
$ travis login --com

```

创建 Travis CI 配置文件（空）

```
$ touch .travis.yml

```

登录成功以后使用

```
$ travis encrypt-file ~/.ssh/id_rsa --add

```

加密私钥并将加密信息添加到 `.travis.yml`。加密成功后 `.travis.yml` 文件内在`before_install`段应该有解密私钥的指令，其中解密过程用到的钥匙以环境变量的形式指定，该环境变量的值对其他用户是透明的，从而保证了安全性。

要注意这里，生成的命令的-out选项的值多了一个\，类似 `-in ./id_rsa.enc -out ~\/.ssh/id_rsa -d`，这会导致报错，要手工去掉。


### 配置文件

一个通常的实践是，创建多个 git 分支，在某些分支上修改，构建成功后集成到 `master` 分支。于是假设以下的示例流程：

1. 创建一个 `dev` 分支，在 `dev` 分支上修改代码并推送；
2. 触发 Travis 的构建程序；
3. 若构建成功，将代码（或构建结果）推送到 `master` 分支。

创建 `dev` 分支

```
$ git checkout -b dev

```

在 `.travis.yml` 文件中 `travis` 自动添加的信息（`before_install`段）下面添加

```
- chmod 600 ~/.ssh/id_rsa
- eval "$(ssh-agent -s)"
- ssh-add ~/.ssh/id_rsa
- git config --global user.name "Travis Bot"
- git config --global user.email <your_email>@example.com
- cd ~
- git clone --depth=50 --branch=dev git@github.com:<username>/<reponame>.git build_dir
- cd build_dir
- git checkout -b master

script:
- bash -x ./compile.sh

after_success:
- git add --all
- git commit -m":black_nib:Updated By Travis"
- git push -u origin HEAD:master --force

branches:
  except:
    - master

branches:
  only:
    - dev

```

注意需要替换 `user.email`, `<username>`, `<reponame>` 等配置信息。

注意到 `script` 段中调用 `compile.sh` 脚本完成构建，因此需要创建一个 `compile.sh` 文件，在里面添加项目相关的构建代码。

完成之后初始推送，即可触发 Travis 的第一次构建。

```
$ git add --all
$ git commit -m"init"
$ git push origin dev

```

到 Travis 上看看，Travis 构建完成后 `master` 分支会在构建成功后由Travis的Push更新。