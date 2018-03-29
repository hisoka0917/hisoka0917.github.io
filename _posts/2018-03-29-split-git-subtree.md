---
layout: post
title: 从现有项目中拆分Git Subtree
date: 2018-03-29
categories: git
tags: [git, subtree, git subtree]
---

使用Git Subtree来管理不同项目中的公共部分。这是平时非常常见的需求。

比如说一个node后端服务使用express框架，前端静态页面放在dist文件夹中。前端工程在另外一个git仓库中，编译生成的静态文件放到这个dist文件夹里面。我们把前端工程叫做P1项目，后端项目叫做P2项目，dist文件夹所在的git项目叫做S项目。
通常情况下都是P1和P2项目都已经存在了，发现这两个项目需要提取公共模块，于是第一步是要拆分现有的工程项目。

#### 1. 拆分现有项目

前端项目P1中已经有了`dist`文件夹，我们不想更改文件夹的名字。那么将现有的文件夹当作subtree的路径。

```bash
cd P1_PATH
git subtree split -P <S项目的相对路径> -b <临时branch>

// 注： -P 与 --prefix 等价
```

在这个例子中具体的命令如下，临时分支名叫做`subtree`

```bash
git subtree split -P dist -b subtree
```

#### 2. 创建子repo

在`dist`目录下新建一个git repo。由于现在`dist`目录下是有文件的，在pull之前的历史记录时会有问题，所以这个时候把现有的文件都删掉。

```bash
cd dist
rm -rf *
git init
git pull <P1项目路径> <临时branch>
git remote add origin <S项目的git仓库>
git push origin -u master
```

在这里具体的pull命令如下

```bash
git pull ../ subtree
```

#### 3. 清理数据

在原项目中清除`dist`下的文件。清理步骤如下：

```bash
cd P1_PATH
git rm -rf <S项目相对路径>
git commit	#提交删除记录
git branch -D <临时branch> #删除临时分支
```

本例中具体命令如下：

```bash
cd ..
git rm -rf dist
git commit -m '删除dist'
git branch -D subtree
rm -rf dist
```

在这里把`dist`物理文件都删除没有关系。

#### 4. 添加subtree

接下来将subtree仓库地址链接到当前项目，命令如下：

```bash
git subtree add --prefix=<S项目相对路径> <S项目git地址> <分支> --squash
```

S项目目前只有一个master分支，所以现在具体的命令如下：

```bash
git subtree add --prefix=dist S_GIT_REMOTE_PATH master --squash
```

`--squash`参数表示将subtree repo中的提交记录合并成一次commit，这样就不用拉取子项目的完整历史记录。

然后我们用`git push`把当前改动提交到远程就可以了。

#### 5. 提交与更新subtree

完成subtree的分离和添加以后我们可以正常的改动代码和提交。这些改动可能涉及到S项目的改动，这些都没有关系。等到我们需要同步S项目的时候，例如前端完成了一个新版本的开发，编译出一份静态资源，需要同步到后端测试。这时候用`git subtree push`命令来提交S项目。

```bash
git subtree push --prefix=<S项目相对路径> <S项目git地址> <分支>
```

Git会遍历之前的commit，找到对S目录的更改，然后把这些更改记录提交到S的Git仓库中。在本例中具体命令如下：

```bash
git subtree push --prefix=dist S_GIT_REMOTE_PATH master
```

在后端的P2项目中使用`git subtree pull --prefix=<S项目相对路径> <S项目git地址> <分支>`来更新

```bash
git subtree pull --prefix=dist S_GIT_REMOTE_PATH master
```

当然在P2项目中重复上述步骤添加subtree，之后就可以愉快的使用git subtree来使用公共模块了。
在本例中我们是分离一个现有的目录当作subtree，如果是新建一个目录来来当作subtree就会稍微简单一点。修改一下步骤2就可以了。

需要注意的是，并不是任意的git仓库都可以添加为subtree的，因为subtree的commit会被合并到当前项目历史记录中，所以如果两个仓库的历史记录无关的话是无法合并的。所以正确的姿势就应该是从当前项目中split出一个子项目来当作subtree。

