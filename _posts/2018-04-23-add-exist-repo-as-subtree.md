---
layout: post
title: 添加现有的repo为Git Subtree
date: 2018-04-23
categories: git
tags: [git, subtree, git subtree]
---

前一篇讲述了从现有项目中拆分subtree的步骤，这一篇说一下如何把一个现有的项目当成subtree加到当前项目中来。

#### 1. Add Remote

我们先把要添加的subtree repo添加为remote，这样能简化后续的操作命令。

```bash
git remote add sub_origin GIT_REMOTE_REPO
git fetch sub_origin
```

#### 2. Add Subtree

用`git subtree add --prefix=local/subtree/path GIT_REMOTE_REPO branch`来添加subtree

```bash
git subtree add --prefix=/path/to/sub sub_origin master --squash
```

`--squash`参数表示将subtree repo中的提交记录合并成一次commit，这样就不用拉取子项目的完整历史记录。

也有用`read-tree`来把子repo添加进来的，比如

```bash
git merge -s ours --no-commit sub_origin/master
git read-tree --prefix=/path/to/sub -u sub_origin/master
git commit -m "Import sub as a subtree"
```

#### 3. 更新subtree

将远程repo的代码拉取下来

```bash
git pull -s subtree sub_origin master --allow-unrelated-histories --squash
```

或者用`git subtree pull`来更新

```bash
git subtree pull --prefix=/path/to/sub sub_origin master --squash
```

这样会有2个commit，一个是squash的commit，一个是merge的commit。
如果不加`--squash`参数的话会报错`fatal: refusing to merge unrelated histories`。因为目前`git subtree`命令还不支持`--allow-unrelated-histories`参数。

#### 4. 改动代码，更新

之后可以正常的写代码了，等到需要更新subtree的代码的时候把更新push上去就可以了。

```bash
git subtree push --prefix=/path/to/sub sub_origin master
```



