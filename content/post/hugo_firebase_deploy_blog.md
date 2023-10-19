---
title : "使用Hugo和Firebase部署个人博客"
author : "Jun"
date : 2021-03-01T12:39:18+08:00
---
## Hugo的使用

[Hugo](https://gohugo.io/)号称是世界上最快的构建网站的框架，我们可以简单的编写MarkDown文件，而通过Hugo进行渲染．由此便可以迅速地构建一个静态网站．此外，Hugo还有大量的开源主题，进一步地方便了开发者们．这次博客的搭建，我们就将使用Hugo

### Hugo的安装

#### ubuntu下

```
sudo snap install hugo
```

#### Mac OS下

```
brew install hugo
```

**注意你需要先安装好Homebrew，参见[这里](https://brew.sh/)**

### 创建项目

我们将通过Hugo搭建一个博客，域名为`example.com`


```
hugo new site example.com
```

此将创建一个目录，结构如下

```
.
└── example.com
    ├── archetypes
    │   └── default.md
    ├── config.toml //网站的配置文件
    ├── content　//存放所写文章
    ├── data
    ├── layouts
    ├── static　//一些静态文件，可以用来存放图片，css，JavaScript文件等
    └── themes　//存放主题，之后从网上下载的主题就存放于此

```
**注意，创建完项目后再使用`Hugo`命令时，当前目录下必须要有配置文件，即`config.toml`，否则会报以下错误**

```
Error: Unable to locate config file or config directory. Perhaps you need to create a new site.
       Run `hugo help new` for details.
```



**static引用时相当于根目录下的一个目录，比如引用static目录下的一张图片，直接写相对目录， 即`/static/test.jpg`**



**`content`可以理解为访问时网站的＂根目录＂**

举例来说，我们创建了`content/post/demo.md`这个文件，网站运行时即可通过`http://example.com/post/demo`访问如果创建了`content/index.md`，网站运行时直接访问首页即是此文件



### 主题的配置

这里我们选用的是[Even](https://themes.gohugo.io/hugo-theme-even/)，如果你想要更多自定义的主题的话，请访问[Hugo官方主题网页](https://themes.gohugo.io/)

#### 下载主题

**注意，执行命令时你应该在博客根目录下！**

首先我们下载主题到theme目录下

```bash
git clone https://github.com/olOwOlo/hugo-theme-even themes/even
```
#### 使用其配置文件

```bash
mv themes/even/exampleSite/config.toml .
```

### 编写文章

首先创建post文件夹，它将保存我们所有文章

```bash
mkdir -p content/post
```

在post文件夹下编写我们的第一篇文章，名称为`demo.md`

```bash
hugo new post/demo.md
```

查看此文件我们可以发现，Hugo会自动在文件上加上一些信息．**此信息很重要，确保了它是否能正确渲染，请不要乱删！**

```
---
title: "Demo"
date: 2021-03-01T11:29:55+08:00
draft: true
---
```

除了默认的参数，Hugo还提供了其它选项供我们使用

- author 表示作者
- categories 表示类别
- tags 表示所属标签
- draft 表示是否为草稿，可删

举例来说，下面是我的一篇文章的头信息

```
---
title: "Hugo & firebase"
author : "Jun"
date: 2021-03-01T11:06:32+08:00
categories : ["hugo"]
tags : ["hugo","firebase"]
---
```



### 测试运行

```
hugo serve
```
首先，Hugo会创建一个public目录，其中也就是它渲染的静态网页．
此后，Hugo会在本地开启一个服务，默认是`http://localhost:1313/`，浏览器访问这个链接即可，每次正式发布文章时可以首先在本地运行下．防止出错．



## Firebase的使用

[Firebase](https://firebase.google.com/)是谷歌开发的一个用于创建移动和网络应用的平台，通过它我们可以轻易地构建服务而不需要管后端的架构．

我们本次博客便部署在Firebase上面．为什么不使用Github Pages呢？我认为Firebase至少有以下几个优点：

- 快．Firebase使用了谷歌云的CDN，相比与Github Pages国内缓慢的访问速度，我这里实测网站延迟不到90ms
- 隐私．我们使用Github Pages需要新建一个公共的仓库，任何人都可以查看，而Firebase则可以直接上传后端
- 廉价．Firebase本身提供了免费计划，对于个人站点绰绰有余，如果不够，官方也提供弹性收费，即用即付

### 注册帐号

由于Firebase为谷歌旗下产品，所以可以直接用谷歌帐号登录即可．接着我们创建一个新项目。

注意，创建过程中页面会默认生成一个 Project ID，我们也可以自己填写，它将决定我们的网站托管在 firebase 上的子域名，而且创建后就不能再修改。

### 命令行工具

Firebase提供了基于`nodejs`的命令行工具，我们将通过它使用Firebase的功能

#### 安装

```
npm install firebase-tools
```

这里我们使用了[npm](https://en.wikipedia.org/wiki/Npm_(software))，需要先安装nodejs，如果你没有安装，请看[这里](https://nodejs.org/en/download/package-manager/)

#### Firebase授权

我们安装好了命令行工具后需要授权认证，这也好理解，不然Firebase怎么知道你是你呢？

```
firebase login
```

如果提示命令没有找到，请执行

```
npx firebase login
```

此后会打开一个链接，按照提示操作即可

**有的读者反映会显示`authorization Error`，这主要是因为你的终端没有走代理．
请看[这篇文章](https://www.junz.org/post/v2_in_linux)的末尾**，使用proxychains便可解决。

#### 初始化并关联项目

切换到博客根目录，执行

```
firebase init // 或者 npx firebase init
```

1. 我们安装提示，选择`Hosting: Configure and deploy Firebase Hosting sites`
2. 关联此前创建的项目
3. 他会询问是否使用`public`目录，`What do you want to use as your public directory? (public)`，回车即可

**经过上述操作Firebase会生成一个json文件，它应该在你的博客根目录下，也就是和config.toml同级，如果不是则说明你的操作有误，没有在博客根目录下init**

#### 部署上传项目

```
hugo && firebase deploy  //或者 hugo && npx firebase deploy
```

此后便会上传我们的项目至Firebase，我们按照提示便可以访问



### 自定义域名

Firebase 默认会给我们分配一个二级域名，但我们可能会想使用自定义的域名，操作如下

#### 添加域名

点击[这里](https://console.firebase.google.com/project/_/hosting/main?hl=zh-cn)，根据提示添加你的域名，我的建议是先添加一个`www.example.com`，再添加一个顶级域名`example.com`，接着让顶级域名重定向到`www`上，这对我们的SEO有好处

#### 验证所有权

我们需要为我们的域名做一个TXT记录，通过它，Firebase可以验证我们对域名的所有权，再此不做详细说明，按照提示在DNS提供商添加记录即可．

#### 上线

返回域名提供商的 DNS 管理网站，创建 DNS A 记录，从而将网站指向 Firebase 托管．

对于我们来说，要分别为顶级域名和一个`www`二级域名做两条A记录

- 对于`www`二级域名，第一栏填`www`，值填FIrebase为我们分配的IP

- 对于顶级域名，第一栏不填或填`@`，值填FIrebase为我们分配的IP

  最后等待解析生效，这个时间根据DNS提供商而异．此外FIrebase还会为自动为我们分配一个[ssl证书](https://www.junz.org/post/https_ssl)，这个过程可能比较漫长，需要我们耐心等待．在生效前访问网站均会显示隐私错误，忽略即可．



