---
layout: post
title:  "NPM组件发布步骤"
date:   2016-12-16
desc: "NPM组件发布步骤"
keywords: "NodeJS"
categories: [JavaScript]
tags: [JavaScript]
---

## 发布前准备

##### npm仓库设置

设置仓库源

`npm set registry http://xx.xx.xx.xx:xxxxx`

##### npm用户登录

登录npm仓库源用户

`npm login`

输入用户名、密码后登录成功，可以使用

`npm whoami`

检查当前登录用户

##### babel转码

发布代码前，请注意如果使用了ES6或者React，请使用babel进行转码，步骤如下：

* babel安装（如果已安装，请跳过）

全局安装babel

`npm install --global babel-cli`

或者直接使用项目中的babel，路径如下

`project(具体项目)/node_modules/.bin/babel`

* babel转码

对目标文件进行转码

`babel src.js -o dest.js`

转码后的文件，就是最后需要发布的文件

##### css样式

目前CSS样式文件，暂时没有研究如何外联，建议如果少量的话，直接改为内联样式

## git repo创建

在GitLab里的[ufo-component](http://xx.xx.xx.xx:xxxx/groups/ufo-component)组中，创建一个新的工程，名字就是要发布的组件名，每一个发布的组件在ufo-component组中都必须有一个对应的git repo。

## npm组件初始化

将上一步创建的工程clone到本地，然后在该路径下打开cmd，执行

`npm init`

在npm初始化的过程中，有以下几步：

1. name: 发布的组件名称
2. version: 发布的组件版本
3. description: 组件描述
4. entry point: 组件的入口文件，默认是index.js 
5. test command: 不用管
6. git repository: 就是上一步创建的git repo地址
7. keywords: 组件关键字
8. author: 组件作者
9. license: 默认是ISC，建议修改成MIT

经过上面的配置，最后就会生成一个package.json文件，上面填写的信息还可以在这个文件中修改。

## npm组件编写

初始化后，就可以将babel转码后的文件直接拷贝过来即可，记住之前设置过的`entry point`，这个文件是组件的入口。

### 组织形式

为了方便其他开发人员使用组件，组件里最好同时提供源码和babel转码后的文件，以及README文档，因此建议组件的组织形式如下：

```
- component
- - dist
- - - babel转码后文件
- - src
- - - 源码文件
- - README.md
- - package.json
- - .gitignore
```

**这里修改了组件的entry point**，可以在package.json文件中修改

`"main": "index.js"` => `"main": "dist/index.js"`

### README

README文件应当包含以下内容：

* 组件描述
* 组件props
* 组件方法
* 组件使用的例子

## npm组件发布

组件的所有文件准备好后，就可以提交git repo，注意如果有需要可以创建**.gitignore**文件，最后发布组件。

`npm publish`

发布完成后，就可以在[仓库](http://xx.xx.xx.xx:xxxxx/)上查看到该组件，其他项目也可以使用该组件。

## npm组件更新

如果组件需要更新时，请牢记要修改package.json中的版本号，否则会提交失败，然后正常发布即可。

`npm publish`
