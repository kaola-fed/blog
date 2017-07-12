---
title: 便于 Code review 的 Git 流程方案
date: 2017-07-04
---

> 设定读者已经了解基本的 Git 操作和 Git 分支管理策略。

为了强化代码记录的可读性并协助 Code review 的执行，通过参考已有流程方案，设定一种适合的 Git 流程方案。

<!-- more -->

## 流程步骤

1. 新建分支
2. 提交 commit 记录
3. 合并 commit 记录
4. 推送到对应的远程仓库
5. 提交 Merge Request 申请

- 附：该流程的另一个好处：git cherry-pick
- 附：使用 git pull --rebase 去掉多余的 Merge Request

## 第一步：新建分支

每次开发新功能，都应该从当前主开发分支新建一个功能分支。

```
# 从当前分支新建功能分支，分支代号按照 JIRA 号设定
$ git checkout -b feature-KJDS-00000
```

## 第二步：提交 commit 记录

添加功能后进行 commit，commit 时需要携带足够的修改信息。

```
# 添加到暂存区并完成提交
$ git add -A
$ git commit -m "header: some message"
```

这里设定：

 - 杂项的提交使用 `misc:` 开头：

```
$ git commit -m "misc: some awesome modification"
```

 - bugfix 或合并后的 commit 记录使用 `jira-No:` 开头：

```
$ git commit -m "jira-00001: fix a specific bug"
```

> 拓展：可以配合「[gitmoji](https://gitmoji.carloscuesta.me/)」强化信息可读性。

## 第三步：合并 commit 记录

假设在 feature 功能分支上新增了 4 条开发记录：

```
commit 34d364d9d51dc94b264e99f7a92add50dd2c3987
Author: Aeo <A_doz@126.com>
Date:   Sun Jun 25 12:27:02 2016 +0800

    misc: forth commit

commit 27322cb4b3f99226ffa98240460b90d92ed55a17
Author: Aeo <A_doz@126.com>
Date:   Sun Jun 25 12:26:42 2016 +0800

    misc: third commit

commit 405b957a96a7dbe352cf7da9a422312a735f6081
Author: Aeo <A_doz@126.com>
Date:   Sun Jun 25 12:26:16 2016 +0800

    misc: second commit

commit cc12fc86a7738ee2f9a8a48c31a9435232c2b08f
Author: Aeo <A_doz@126.com>
Date:   Sun Jun 25 12:25:53 2016 +0800

    misc: first commit
```

现在我们已经完成了 feature 分支的开发和联调，需要把代码合并到当前主开发分支。

此时可以合并一些不必要的 commit 使主开发分支保持干净，优先推荐 **git rebase** 进行合并操作。

### rebase：一般翻译为「衍合」或「变基」

如果你想要合并最后 4 个 commit，你可以运行下列命令：

```
# -i 参数表示互动 (interactive)
$ git rebase -i HEAD~4
```

这时 git 会打开一个互动界面，方便用户对历史提交进行修改：

```
pick cc12fc8 misc: first commit
pick 405b957 misc: second commit
pick 27322cb misc: third commit
pick 34d364d misc: forth commit

# Rebase 2763481..34d364d onto 2763481 (4 command(s))
#
# Commands:
# p, pick = 使用该提交
# r, reword = 使用该提交，但需要编辑提交信息
# e, edit = 使用该提交，但此处暂停并提供修改机会
# s, squash = 使用该提交，但合并到上一个提交记录中
# f, fixup = 类似 squash，但丢弃当前提交记录的提交信息
# x, exec = 执行 shell 命令
# d, drop = 移除当前提交
```

我们需要的是利用 **`squash`** 或者 **`fixup`** 进行分支合并：

```
pick cc12fc8 misc: first commit
squash 405b957 misc: second commit
squash 27322cb misc: third commit
squash 34d364d misc: forth commit
```

若使用 `fixup` 的话，则直接丢弃其它记录，省去下一步操作。但使用 `squash` 时，则需要在下一步中对这 4 条 commit 信息进行修改和保存：

```
# This is a combinatin of 4 commits.
# This is the 1st commit message:

misc: first commit

# This is the commit message #2:

misc: second commit

# This is the commit message #3:

misc: third commit

# This is the commit message #4:

misc: forth commit

```

编辑并保存后，通过 `git log` 可以看到仅剩一条 commit 记录：

```
# 此时的 commit SHA 与之前的并不相同
commit da473276aa981f6e29577aa09a525109971547f2
Author: Aeo <A_doz@126.com>
Date:   Sun Jun 25 12:50:53 2016 +0800

    misc: first commit
```

### rebase 的风险：

使用 rebase 需要遵循一条准则：

**一旦分支中的提交对象发布到公共仓库，就千万不要对该分支进行衍合操作。**

简单来说，当待合并 commit 记录中杂糅着他人的 commit 记录，此时就不可以再对这部分 commit 记录做合并操作。

但只要新开分支且保持分支独立开发，杂糅的情况就不存在。

## 第四步：推送到对应的远程仓库

完成 commit 记录的合并后，此时可以反合目标分支的最新代码，避免后续提交 Merge Request 时存在冲突。

```
$ git push origin feature-KJDS-00000
```

## 第五步：提交 Merge Request 申请

当完成功能后提交到远程后，需要经过下述流程：

 1. 提交 Merge Request：

    通过「＋Create Merge Request」按钮创建一个 Merge Request；

 2. **必须**指定「Assignee」：

    指定需要 review 你代码的同学，禁止指定自己；

 3. 更改「Target branch」：

    改变为需要合并进去的目标分支；

 4. 设置合并后删除被合并分支的选项：

    勾选「Remove source branch when merge request is accepted.」选项，在合并完成后删除源分支，控制分支总数量；

 5. 使用「Submit Merge Request」提交合并请求。

完成上述设置后，相关同学将会收到邮件通知，此时可进入 GitLab 进行 code review。

如果发现问题则对问题代码进行点评并拒绝关闭申请，反之则通过合并申请。

合并申请通过后留下的 Merge Request 记录，也就记录了 code reviewer 。

## 附：该流程的另一个好处：git cherry-pick

通过该流程提交的代码，每个功能对应着唯一的提交记录。

此时如果单独提前上线部分功能，则可以在待上线分支中直接使用：

```
git cherry-pick commit-SHA
```

这里的 commit-SHA 可以是其它分支上的提交记录，此时可以直接将其抓到待上线分支上，避免从杂糅代码中扣取功能的时耗和风险。

### 附：使用 git pull --rebase 替代 git pull，去除多余的 Merge Request

在项目初始搭建阶段，可能需要多人同时在一个分支上进行开发，并有大量的并发操作，此时上述流程方案便不太适用。

但此时我们仍然可以利用 git pull --rebase 尽可能保持提交记录清晰。

如果自己在本地 commit 了代码，然后执行 `git pull`，这时候极有可能出现这样的合并记录：

    Merge branch 'feature' of ssh://domain/some-system into feature

但其实大部分情况下这里是不会有冲突的。所以拉取远程代码时，使用 `git pull --rebase` 就可以保持提交历史记录的整洁。

> 原理：把本地当前分支的未推送 commit 临时保存为补丁 (patch)（这些补丁放到 `.git/rebase` 目录中），然后将远程分支的代码拉取到本地，最后把保存的这些补丁再应用到本地当前分支上。

若要把 rebase 当做 `git pull` 的预设值，可以修改 `~/.gitconfig` 让所有 tracked branches 自动使用该设定：

    [branch]  
      autosetuprebase = always

## 附录

1. [Git 分支的衍合](https://git-scm.com/book/zh/v1/Git-%E5%88%86%E6%94%AF-%E5%88%86%E6%94%AF%E7%9A%84%E8%A1%8D%E5%90%88)
2. [Git 使用规范流程 - 阮一峰](http://www.ruanyifeng.com/blog/2015/08/git-use-process.html)