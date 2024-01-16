---
title: Hexo 框架(一) 搭建自己的博客
date: 2021-07-27 10:03:27
tags:
---

## 前置准备

### **1. 安装nodejs**

> [参考nodejs](https://www.runoob.com/nodejs/nodejs-install-setup.html)

- 可能会遇到的问题
  **nodejs 版本过高**
  : 可以使用 npm 进行版本控制
  : >[参考地址](https://blog.csdn.net/weixin_34051201/article/details/89093598)
<!--more-->
### **2. 安装,使用git**

> [安装地址](https://git-scm.com/download/win)

- 设置 user.name 和 user.email

  ```
  COPYgit config --global user.name "你的GitHub用户名"
  git config --global user.email "你的GitHub注册邮箱"
  ```

- 生成ssh密钥文件

  ```
  COPYssh-keygen -t rsa -C "你的GitHub注册邮箱"
  ```

  找到

   

  ```
  ~/.ssh
  ```

   

  文件中的 id_rsa.pub密钥,将内容全部复制

打开[GitHub->Settings->SSH and GPG keys](https://github.com/settings/keys) ,新建new SSH key,然后将刚刚复制`id_rsa.pub`的内容全部粘贴进去，然后点击`Add SSH key`。

输入

```
COPYssh -T git@github.com
```

当出现 `Hi,你的GitHub名称` 时成功

## 安装 Hexo

### 1. 使用 npm 命令安装Hexo

```
npm install -g hexo-cli
```

### 2. 初始化博客

```
hexo init blog
```

### 3. 新建一个博客内容

```
# hexo new 的缩写
hexo n 第一个博客
```

### 4. 开启hexo

```
COPY# hexo generate 的缩写
hexo g

# hexo server 的缩写 启动项目 ，如果要后台运行 hexo s & ,默认地址 localhost:4000
hexo s 
```

### 5. 查看地址

```
COPYlocalhost:4000
# 或者 ip:4000
```

## hexo 基本操作

- 本地启动项目，s表示server

```bash
hexo s --debug
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/752110235a9f43bc95eab89b69fef772~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 创建不可

```bash
hexo n [layout] <title>
```

- layout : 文章布局，可选（post page draft），默认使用 `_config.yml` 中的 default_layout 参数代替（post）。 
- title : 文章标题。

![其他参数](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f88af813766745c9abb784f32671abce~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 生成静态文件，g表示generate

```bash
hexo g
```

![img](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c40809a7560f40cba08410e207a75c17~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp)

- 部署Hexo网站，d表示deploy

```bash
hexo d
```

- 清除缓存文件 (db.json) 和已生成的静态文件 (public)

```bash
hexo clean
```

参考官网：[hexo.io/zh-cn/docs/…](https://link.juejin.cn?target=https%3A%2F%2Fhexo.io%2Fzh-cn%2Fdocs%2Fcommands)
