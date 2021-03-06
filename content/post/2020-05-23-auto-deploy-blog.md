---
title: 使用travis进行hugo自动化部署
date: '2020-05-23'
draft: 'true'
description: 使用travis进行hugo自动化部署
author: dashjay
---

> 原来博客写了个脚本，把public文件夹完全删除，然后再次`git init`再-f push到github.io的repo，总之每次源文件要推送，public要单独推送。因为 github page 不能自动运行 hugo 的部署，故写了此次博客记录一下。

1. 首先我在git上开了一个repo，然后把我的源博客给push上去，<https://github.com/dashjay/blog>

2. 紧接着又开了一个githubpage的空repo，如果你已经有了，就没有这个步骤了。确认一下github page 的设置。
  
![github page 配置](/post/博客自动化部署/9833528A-7A75-4324-9039-79702CFD2928.png)

设置没问题的话，githubpage的repo添加一个GITHUB_TOKEN，保证次TOKEN有repo的 `public_repoAccess public repositories`权限 [点此处去配置Token](https://github.com/settings/tokens)

我最后没有使用Token来访问，转而使用用户名和密码了。

3. 在项目的根目录下创建一个名字叫 .travis.yml 的文件内容如下。

本身应该使用go来get hugo，但是我发现直接下载最新的发行版不容易出错，也更快，因此就这么办了。

```php
dist: xenial
language: python
python: 3.7

before_install:
  - sudo apt-get update -qq
  - sudo apt-get -yq install apt-transport-https

install:
  # install latest hugo release version
  - wget -qO- https://api.github.com/repos/gohugoio/hugo/releases/latest | sed -r -n '/browser_download_url/{/Linux-64bit.deb/{s@[^:]*:[[:space:]]*"([^"]*)".*@\1@g;p;q}}' | xargs wget
  - sudo dpkg -i hugo*.deb
  - rm -rf public 2> /dev/null

before_script:
  - echo -e "Host github.com\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
  - git config --global user.email dashjay@aliyun.com
  - git config --global user.name dashjay

script:
  - hugo -d ./dist/

after_success:
  - DATE=$(date)
  - cd dist
  - git init
  - git remote add origin https://dashjay:${password}@github.com/dashjay/dashjay.github.io
  - git add .
  - git commit -m "${DATE}"
  - git push origin master -f
```

然后上面的配置改成你的，其中这个 `${password}` 是不得已而为之，本身是使用一个叫做deploy的命令，来把`git repo push`到github，但是往往又那么艰难，中间失败很多次，所以直接使用密码了......

配置`${password}`的位置在 ![密码配置](/post/博客自动化部署/93FA7205-51F8-4B95-A222-7F3CCCFA133D.png) 每个项目是独立的，配置空间。

一旦你推送之后，就会触发 `travis` 的构建，但是别忘了，要给 `travis` 提供你的public或者所有项目的 `access` 权，否则不会触发，你打开之后使用github登录即可看到权限配置·。

### 后记

像`travis`, `gitlab-ci/cd`，`jenkins`这类东西，当项目较大或者共同维护的的时候，就很有必要弄一个了，或者你的公司在多人开发的时候，大多都会用到这几个中的一个，一般互联网公司都有`gitlab`。

我的项目并没有多大，但是我在强迫自己学习这些知识，偶尔还是能学到新的知识的。
