---
title: "Hugo + Github Pages搭建个人博客"
date: 2021-03-03T13:11:51+08:00
tags: ["Hugo", "搭建个人博客"]
categories: ["静态网站"]
draft: false
---
#### 下载Hugo

##### 二进制安装
```
1. 上github（https://github.com/gohugoio/hugo/releases）下载对应的压缩包，然后进行解压，再将目录配置到环境变量中
2. 执行：hugo version，验证是否安装成功
```
##### 源码安装
```
1. 安装Git（https://git-scm.com/downloads）
2. 安装Golang（https://golang.org/dl/）
3. 安装Hugo
git clone https://github.com/gohugoio/hugo.git
cd hugo
go install

PS：如果需要下载支持编译SASS的版本，需要为clone时加上--tags extended参数，
安装go模块由go install改为执行：CGO_ENABLED=1 go install --tags extended，
如果安装go模块时报错：exec: "gcc": executable file not found in %PATH%，可按参考文献1做法安装gcc后再重试

```
PS：如果按源码的方式安装，后续执行hugo命令可能需要在git的客户端内才能执行，在windows cmd客户端内执行可能会报一下错误
![Windows窗口执行错误](/images/hugo_start-hugo_exe_error.png)

#### 创建项目
```
# 生成项目
hugo new site "项目名"
# 生成博文
hogo new posts/xxx.md
```

#### 安装主题

官网主题库：[Hugo Themes](https://themes.gohugo.io/)
```
# 引入主题
git submodule add <主题仓库地址> themes/<主题名>
```

#### 修改配置
一般主题都有提供一份config.toml或config.yml配置，将其拷贝到项目的根目录并进行修改即可，如果使用Github Pages发布的话，需要在config.toml内添加：publishDir = docs 配置

#### 生成静态站点的目录
```
# 根目录下执行hugo命令生成静态站点的目录，默认为 public 目录
hugo
```

#### 配置Github Pages
在仓库的Settings -> Options下方找到Github Pages配置，配置source为以上生成的博客文件夹
![GitHub Pages配置](/images/hugo_start-github_pages_config.png)

#### 参考文献
- [golang 编译cgo模块exec: "gcc": executable file not found in %PATH%](https://www.jianshu.com/p/c38483c30fb7)
- [Hugo 和 GitHub Pages 使用指南](https://jsonbruce.com/posts/userguide/)
- [浅谈我为什么从 HEXO 迁移到 HUGO](https://sspai.com/post/59904)