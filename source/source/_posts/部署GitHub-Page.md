---
title: 部署GitHub Page
date: 2017-07-03 17:45:04
tags: 部署GitHub Page
---

##### 环境

+ git
+ nodejs
+ hexo

##### 配置gitHub

+ 申请github账号，在本地生成ssh秘钥，并配置GitHub
+ 在GitHub上创建一个与GitHub用户名一致的仓库，eg：yourUserName.github.io
+ 将新建的仓库clone到本地目录

##### 安装nodejs

+ nodejs 的安装有多重方式，本例是通过nvm来安装nodejs。首先，[安装nvm](http://weizhifeng.net/node-version-management-via-n-and-nvm.html) 。

+ 配置文件

    + 配置 `~/.bashrc` 文件
    + 配置 `~/.bash_profile` 文件
	

> ~/.bashrc配置

```
export NVM_DIR="~/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
```

> ~/.bash_profile配置

```
if [ -f ~/.bashrc ]; then
   . ~/.bashrc
fi
```

+ 执行sh  `source ~/.bash_profile`

<!-- more -->
##### 安装Hexo
+ 安装

```
# npm install -g hexo-cli
```
+ 初始化

    在本地新建一个文件夹A，然后在终端执行：

```
# mkdir A
# cd A
# hexo init
# npm install
```

+ 启动服务

```
# hexo server
[info] Hexo is running at http://localhost:4000/. Press Ctrl+C to stop.
```

这样在本地就能看到你的blog了

##### 调试

```
# hexo new "newPost" 新建文章
# vim newPost.md  编辑文章
# hexo new page "pageName" #新建页面
# hexo generate #生成静态页面至public目录
# hexo server #启动本地服务，进行文章预览调试
```

[Hexo参考](https://hexo.io/docs/)


##### 配置并发布
+ 新建一篇文章

```
# hexo new "newPost"  //会在目录下生成 source\_posts\My-New-Post.md
```

+ 配置_config.yml

    将文件中的如下内容

```
# Deployment
## Docs: http://hexo.io/docs/deployment.html
deploy:
  type:
```

修改为：

```
...
# Deployment
## Docs: http://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@github.com:yourUserName/yourUserName.github.io.git
  branch: master
```

+ 发布

```
# hexo deploy
```

##### 常见错误

执行 hexo deploy 后,出现 <font color=red>error deployer not found:github</font> 的错误
hexo 更新到3.0之后，deploy的type 的github需要改成git, 接着
** npm install hexo-deployer-git -save ** 改了之后执行，然后再部署


#### 日常部署步骤

```
# hexo s  //预览
# hexo clean
# hexo generate
# hexo deploy
```

##### 主题安装

hexo的支持的[主题](https://github.com/hexojs/hexo/wiki/Themes)很多，可以根据个人爱好随意切换主题。以安装modernist为例：

+ 从GitHub上clone代码到hexo的themes目录下：

```
git clone https://github.com/heroicyang/hexo-theme-modernist.git themes/AA
```

**备注：在切换主题时，只需要将_config.yml配置文件中的theme对应的值指定为clone到的目录，即AA**

+ 修改hexo下_config.yml配置文件

修改

```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: landscape
```

为

```
# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: AA
```

+ 配置主题

    进去hexo\themes\modernist目录，编辑主题配置文件_config.yml：

```
menu:
  Home: /
  Archives: /archives
  rss: /atom.xml
archive_date_format: MMM DD
fancybox: true
duoshuo_shortname:
google_analytics:
favicon: /favicon.ico
```

进行相应的配置。

**添加“Fork me on Github” ribbon**

打开这个[ribbon](https://github.com/blog/273-github-ribbons/) 把a 标签的代码粘贴到 hexo\themes目录下的\layout\layout.xxx 中，放置在最后，记得修改你的github
地址