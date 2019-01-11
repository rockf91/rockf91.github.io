---
title: 使用Travis CI给你的Github项目自动生成Doxygen文档
date: 2018-01-25 22:21:38
tags: travis-ci
---

 [Travis CI](https://travis-ci.org) 是一个基于云的持续集成项目，提供免费的云主机替你完成自动编译测试等功能。可以利用它做很多事情，一键发布什么的，岂不美哉？

 [Doxygen](http://doxygen.org) 是一个生成参考文档的工具，它直接抽取代码中一定格式的注释生成html站点或者pdf文档。因为文档是通过注释生成的，所以不用刻意维护，很方便。

本文介绍如何利用[Travis CI](https://travis-ci.org) 每次push代码后，自动在 `gh-pages` 分支生成新的 [Doxygen](http://doxygen.org) 文档网站。

<!-- more -->

## 首先注册Travis CI然后同步你的工程
直接用你的github账号登陆，然后Sync account就能看到你的项目列表了

## 创建个干净的 `gh-pages` 分支先
GitHub的gh-pages分支可以自动建站，参考[Github Pages](https://pages.github.com/)

```PowerShell
cd /你的项目目录
git checkout --orphan gh-pages
git rm -rf .
echo "My gh-pages branch" > README.md
git add .
git commit -a -m "Clean gh-pages branch"
git push origin gh-pages
```

## 创建自动发布脚本
这个脚本将会调用[DOXYFILE](http://www.stack.nl/~dimitri/doxygen/manual/starting.html#step1) 配置生成文档然后提交到 `gh-pages` 分支

生成`DOXYFILE`配置文件，放在 `项目根目录`

`DOXYFILE`的输出目录 `OUTPUT_DIRECTORY` 设空，源码目录 `INPUT` 填 `$(TRAVIS_BUILD_DIR)`

```
OUTPUT_DIRECTORY       = 
INPUT                  =  $(TRAVIS_BUILD_DIR)
```

创建脚本 `generateDocumentationAndDeploy.sh` 放在 `项目根目录`
```Shell
#!/bin/sh
################################################################################
# Title         : generateDocumentationAndDeploy.sh
# Date created  : 2016/02/22
# Notes         :
__AUTHOR__="Jeroen de Bruijn"
# Preconditions:
# - Packages doxygen doxygen-doc doxygen-latex doxygen-gui graphviz
#   must be installed.
# - Doxygen configuration file must have the destination directory empty and
#   source code directory with a $(TRAVIS_BUILD_DIR) prefix.
# - An gh-pages branch should already exist. See below for mor info on hoe to
#   create a gh-pages branch.
#
# Required global variables:
# - TRAVIS_BUILD_NUMBER : The number of the current build.
# - TRAVIS_COMMIT       : The commit that the current build is testing.
# - DOXYFILE            : The Doxygen configuration file.
# - GH_REPO_NAME        : The name of the repository.
# - GH_REPO_REF         : The GitHub reference to the repository.
# - GH_REPO_TOKEN       : Secure token to the github repository.
#
# For information on how to encrypt variables for Travis CI please go to
# https://docs.travis-ci.com/user/environment-variables/#Encrypted-Variables
# or https://gist.github.com/vidavidorra/7ed6166a46c537d3cbd2
# For information on how to create a clean gh-pages branch from the master
# branch, please go to https://gist.github.com/vidavidorra/846a2fc7dd51f4fe56a0
#
# This script will generate Doxygen documentation and push the documentation to
# the gh-pages branch of a repository specified by GH_REPO_REF.
# Before this script is used there should already be a gh-pages branch in the
# repository.
# 
################################################################################

################################################################################
##### Setup this script and get the current gh-pages branch.               #####
echo 'Setting up the script...'
# Exit with nonzero exit code if anything fails
set -e

# Create a clean working directory for this script.
mkdir code_docs
cd code_docs

# Get the current gh-pages branch
git clone -b gh-pages https://git@$GH_REPO_REF
cd $GH_REPO_NAME

##### Configure git.
# Set the push default to simple i.e. push only the current branch.
git config --global push.default simple
# Pretend to be an user called Travis CI.
git config user.name "Travis CI"
git config user.email "travis@travis-ci.org"

# Remove everything currently in the gh-pages branch.
# GitHub is smart enough to know which files have changed and which files have
# stayed the same and will only update the changed files. So the gh-pages branch
# can be safely cleaned, and it is sure that everything pushed later is the new
# documentation.
rm -rf *

################################################################################
##### Generate the Doxygen code documentation and log the output.          #####
echo 'Generating Doxygen code documentation...'
# Redirect both stderr and stdout to the log file AND the console.
doxygen $DOXYFILE 2>&1 | tee doxygen.log

################################################################################
##### Upload the documentation to the gh-pages branch of the repository.   #####
# Only upload if Doxygen successfully created the documentation.
# Check this by verifying that the html directory and the file html/index.html
# both exist. This is a good indication that Doxygen did it's work.
if [ -d "html" ] && [ -f "html/index.html" ]; then

    echo 'Uploading documentation to the gh-pages branch...'
    # Add everything in this directory (the Doxygen code documentation) to the
    # gh-pages branch.
    # GitHub is smart enough to know which files have changed and which files have
    # stayed the same and will only update the changed files.
    git add --all

    # Commit the added files with a title and description containing the Travis CI
    # build number and the GitHub commit reference that issued this build.
    git commit -m "Deploy code docs to GitHub Pages Travis build: ${TRAVIS_BUILD_NUMBER}" -m "Commit: ${TRAVIS_COMMIT}"

    # Force push to the remote gh-pages branch.
    # The ouput is redirected to /dev/null to hide any sensitive credential data
    # that might otherwise be exposed.
    git push --force "https://${GH_REPO_TOKEN}@${GH_REPO_REF}" > /dev/null 2>&1
else
    echo '' >&2
    echo 'Warning: No documentation (html) files have been found!' >&2
    echo 'Warning: Not going to push the documentation to GitHub!' >&2
    exit 1
fi
```

## 创建Travis CI的配置脚本
这个脚本会调用刚才的`generateDocumentationAndDeploy.sh`

创建脚本`.travis.yml`放在 `项目根目录`

修改`<your_repo>`为你的项目名，`<your_name>` 你的github用户名

```yml
# This will run on Travis' 'new' container-based infrastructure
sudo: false

# Blacklist
branches:
  except:
    - gh-pages

# Environment variables
env:
  global:
    - GH_REPO_NAME: <your_repo>
    - DOXYFILE: $TRAVIS_BUILD_DIR/<Doxyfile>
    - GH_REPO_REF: github.com/<your_name>/<your_repo>.git

# Install dependencies
addons:
  apt:
    packages:
      - doxygen
      - doxygen-doc
      - doxygen-latex
      - doxygen-gui
      - graphviz

# Build your code e.g. by calling make
script:
  - make

# Generate and deploy documentation
after_success:
  - cd $TRAVIS_BUILD_DIR
  - chmod +x generateDocumentationAndDeploy.sh
  - ./generateDocumentationAndDeploy.sh
```

## 给Travis CI提供授权
Travis CI发布到分支需要GitHub的授权，搞一个GitHub私钥给它吧。

### 生成GitHub私钥
进入GitHub的 [私钥页面](https://github.com/settings/tokens) ，点 `Generate new token` 新建，`Token description` 随便填个别搞混就行。`Select scopes`勾选`repo`下的 `public_repo`就好，不用给太多权限。点绿绿的`generated token`按钮生成秘钥，存下你的秘钥，就这一次机会，关页面就没了。

### Travis CI添加秘钥
参考 [Travis CI 设置变量文档](https://docs.travis-ci.com/user/environment-variables/#Defining-Variables-in-Repository-Settings)。进入Travis CI项目的settings页面，在Environment Variables列 `Name`填`GH_REPO_TOKEN`，`Value`填刚才的秘钥。 `Display value in build log` 不要开，不然秘钥就公开了。最后点击`Add`完成设置。

## 看看范例吧
 [一个小项目](https://github.com/rockf91/zFrame)和它`gh-pages`分支上的[文档页面](rockf91.github.io/zFrame/html/) 

两个文中提到的脚本：

​	[`genDocsAndDeploy.sh`](https://github.com/rockf91/zFrame/blob/master/genDocsAndDeploy.sh)

​	[`.travis.yml`](https://github.com/rockf91/zFrame/blob/master/.travis.yml)

配置过程中有什么疑问的话，欢迎留言！