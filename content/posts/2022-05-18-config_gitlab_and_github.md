---
categories:
- Git
date: '2022-05-18T12:50:00.000Z'
showToc: true
tags:
- Git
- Gitlab
- Github
title: 配置同时使用gitlab和github

---



## 配置步骤

### 1. **分别生成两份密钥文件**

git bash下执行以下命令，并将生成的文件分开存放：


```shell
ssh-keygen -t rsa -C 'your gitlab email'
ssh-keygen -t rsa -C 'your github email'
```

### 2. **设置SSH Key**

将生成的两份id_rsa.pub文件内容分别对应复制粘贴到github和gitlab的SSH Key配置下

### 3. **将key在本地存储起来**

git bash下执行以下命令，ssh-add后的路径为上面生成的id_rsa文件的路径：


```shell
ssh-agent -s
ssh-add ~/.ssh/github/id_rsa
ssh-add ~/.ssh/id_rsa
```

**PS：如果执行ssh-add时报了以下错误，需先执行： ssh-agent bash 命令**

> Could not open a connection to your authentication agent.

### 4. **.ssh目录下创建config文件来管理key**

在用户家目录下的.ssh目录（如果不存在可自己创建）下创建config文件，内容如下（需自行替换对应的内容）：


```plain text
Host github.com // 不动
    HostName ssh.github.com // 不动
    User xxxx@xxx.com // 你自己的github邮箱
    PreferredAuthentications publickey // 不动
    IdentityFile ~/.ssh/github/id_rsa // github用的rsa文件路径
    Port 443 
    // 如果ssh -T git@github.com的时候报 ssh: connect to host github.com port 22: Operation timed out就把Port这条加上吧

Host 192.168.0.231 // 你们公司gitlab的ip地址
    HostName 192.168.0.231 //与Host保持一致
    User xxx@xxxx.com // 你gitlab的邮箱
    IdentityFile ~/.ssh/id_rsa //  gitlab用的rsa文件路径
    Port 64222 // 你们公司gitlab的ip端口
```

### 5. **验证**

git bash下执行以下命名验证配置：


```shell
ssh -T git@github.com
ssh -T git@gitlab的hostname
```

如果有看到successfully或Welcome的相关信息，说明以及配置成功

## **设置仓库使用的提交用户**

### **针对项目配置**

在初始化项目（本地创建项目、通过git clone或IDEA下载仓库）后，可以通过以下命令设置该项目提交时使用的用户，git bash下执行：


```shell
git config --local user.name 'your name'
git config --local user.email 'your email'
```

命令执行后会修改项目根目录下的.git目录下的config文件，你也可以直接修改这个文件，加入以下配置：


```plain text
[user]
	name = your name
	email = your email
```

### **全局配置**

如果不想针对一个一个项目设置，可以通过配置全局的参数对每个项目生效（相当于缺省的默认参数），但缺点就是只能设置一个用户，可以通过以下命令设置全局使用的用户，git bash下执行：


```shell
git config --global user.name 'your name'
git config --global user.email 'your email'
```

### **Conditional Includes**

在 git 2.13 版本中，增加了 conditional includes 配置，可针对不同的根目录使用不同的.gitconfig配置文件，这样就可以针对不同项目使用不同的用户了。比如针对github和gitlab项目配置不同的用户，修改用户家目录下的.gitconfig文件，加入以下配置（项目存放路径需自己替换）：


```plain text
[includeIf "gitdir:D:/Dev/github/"]
    path = .github
[includeIf "gitdir:D:/Dev/gitlab/"]
    path = .gitlab
```

在用户家目录下分别创建.github和.gitlab，并加入以下配置：

- .github


```plain text
[user]
	name = your github name
	email = your github email
```

- .gitlab


```plain text
[user]
	name = your gitlab name
	email = your gitlab email
```

配置完成后可以到配置的目录下执行以下命令进行校验是否生效：


```shell
git config --show-origin --get user.name
git config --show-origin --get user.email
```

## **推荐资料**

- [Git 进阶指南](https://gb.yekai.net/)

- [Learn Git Branching](https://learngitbranching.js.org/?locale=zh_CN)

