---
title: github + jekyll搭建个人blog
author: cotes
date: 2023-10-23 11:33:00 +0800
categories: [博客, 教程, 学习]
tags: [教程]
pin: true
math: true
mermaid: true
image:
  path: /commons/devices-mockup.png
---

## 环境配置

- 安装 ruby
  - 官网地址：<https://rubyinstaller.cn/>
  - 下载完安装直接点下一步就行了
- 安装 RubyGems

  - 直接到这里下载就可以了<https://github.com/rubygems/rubygems/releases/tag/v3.4.21>
  - 下载完了解压缩后，切换到 RubyGems 目录，执行以下 cmd 命令：
  - ```
    ruby setup.rb  #安装
    gem -v         #检查版本
    ```

- 安装 Jekyll
  - 以上两个步骤操作完成后，在 CMD 窗口执行如下命令安装 Jekyll：
  - ```
    gem install jekyll   #安装jekyll
    jekyll -v            #查看jekyll版本号
    ```
- 安装 bundler
  - 接下来安装 bundler，执行以下 cmd 命令即可：
  - ```
    gem install bundler   #安装bundler
    bundler -v            #查看bundler版本号
    ```

## 博客搭建与部署

- 启动项目
  - 执行以下命令，创建和启动我们的项目：
  - ```
      jekyll new Blog              #新建博客
      cd Blog                      #切换目录
      jekyll server                #启动项目
    ```
    > 执行 `jekyll new Blog` <br>
    > 可能会卡到 Running bundle install in /blog… 这一步，这时候`ctrl+c`终止掉，进入新建的项目，将 gemfile 的 source "https://rubygems.org"改为source ‘https://gems.ruby-china.com’, 然后在当前目录执行 `bundle install` 就行。<br> `bundle exec jekyll serve`启动服务
  - 启动日志如下：
  - ![Desktop View](setking.github.io/assets/img/sample/Code.png)
  - 浏览器访问：<http://127.0.0.1:4000>
- 添加 MarkDown 文档
  - 在项目根目录下的 \_posts 目录创建 markdown 文档。这里注意 md 文档命名要添加 “yyyy-mm-dd”的前缀。
    例如：2023-10-23-5 分钟搭建博客.md
- 部署代码到 Github
  - 1、 创建一个名称为 ‘账号名称.http://github.io’。例如：我的账号名是userName，仓库名就是 userName.github.io
  - 2、 在我们创建的博客的目录找到 \_site 目录，执行`git init`创建本地 git，`git add .`+`git commit -m "first commit"`把你的项目提交到 git，然后执行`git remote add origin https://gitee.com/githubName\userName.github.io.git`添加你创建好的远程仓库，最后执行`git push -u origin master`把本地代码都提交到远程仓库
  - 3、最后就可以访问你的 blog 了，地址是你之前在 github 创建好了的仓库：userName.github.io
- 切换主题

  - 这里推荐两款 Jekyll 主题的网站：
    - 1.官方主题网站：http://jekyllthemes.org/
    - 2.Github 上的博客模板：https://github.com/jekyll/jekyll/wiki/Sites
  - 找到你喜欢的主题后下载下来解压，切换到主题的目录，在 cmd 执行下面命令：

        - ```
             bundle install             #安装依赖
             bundle exec jekyll serve   #运行项目
          ```

    > 后记：markdown 加载图片的话可以从本地绝对位置加载，也可以从服务器通过 url 加载 <br>
    > 1、jekyll 部署在本机的服务器，如下加载图片: <br> `![image02](http://localhost:4000/asset/img/img01.jpg)` <br>
    > 2、放到 github 的话，将 github 视为托管你图片的一个大服务器，改变其 URL 地址就可以了 <br> `![image03](your-github-page-address/path/to/img01.jpg)`

[^footnote]: The footnote source
[^fn-nth-2]: The 2nd footnote source
