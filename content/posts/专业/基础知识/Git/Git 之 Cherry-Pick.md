---
title: Git 之 Cherry-Pick
date: 2025-10-28T00:19:15+08:00
lastmod: 2025-10-28T00:19:15+08:00
draft: false
tags:
  - Git
  - 后端知识
categories:
author: 小石堆
---

## 什么是摘樱桃？

在分支协作中，出现以下这种情况，假设我的 main 分支只想要 feat 分支的 E 或者 F 的改动，此时就需要用到像摘樱桃一样给那几个提交摘过来（一个个小提交就像小樱桃），也就是 cherry-pick。
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251028010455368.png)
理解起来不难，但在实际操作中，还是有一些小问题的。我把目前摘樱桃的情况归为三类：

1. 第一类是无冲突直接摘，例如 E 提交创建了一个新文件，这时 main 直接摘 E 过来是没问题的。
2. 第二类是假设 F 是基于 E 的改动，但是我摘的时候，只摘了 F 过来，没有摘 E，此时摘樱桃操作会卡住。
3. 第三类是你摘过来的提交和你本地的提交产生了冲突，此时需要解决冲突，cherry-pick 操作会卡住。

## 无冲突直接摘

这个没什么好说的，很简单，所以这里介绍一下几个 cherry-pick 的命令，分别是

```git
git cherry-pick A // 只摘提交 A
git cherry-pick A..B // 摘提交(A, B]
git cherry-pick A^..B // 摘提交[A, B]
git cherry-pick --abort // cherry-pick 操作终止，回滚
git cherry-pick --continue // 解决冲突后，cherry-pick 操作继续执行
```

## 所摘提交的依赖提交未摘

现在 feat 分支中有一个 main 分支中不存在的文件 READMEX.md，feat 中分别有两个提交，提交 A 是创建文件并写入一些内容，提交 B 是在文件中追加了一些内容。

```
# gitTest base02 cherry-pick 提交 4 // A


# gitTest base02 cherry-pick 提交 5 // B
```

此时，假设我在分支 main 上直接 pick B 会怎么样？这时会报错，提示我们提交 B 所依赖的前置提交不存在，该怎么解决呢？

```
❯ git cherry-pick 722ba9ec56966187e96
CONFLICT (modify/delete): READMEX.md deleted in HEAD and modified in 722ba9e (base02 cherry-pick test5).  Version 722ba9e (base02 cherry-pick test5) of READMEX.md left in tree.
error: could not apply 722ba9e... base02 cherry-pick test5
hint: After resolving the conflicts, mark them with
hint: "git add/rm <pathspec>", then run
hint: "git cherry-pick --continue".
hint: You can instead skip this commit with "git cherry-pick --skip".
hint: To abort and get back to the state before "git cherry-pick",
hint: run "git cherry-pick --abort".
```

两种方案：

1. --abort（计划有变，终止交易）
2. 先摘 A 再摘 B
3. 把 AB 全摘过来
4. 删库跑路
   这里我们看第三种方案的具体实操，这会就需要用到我们前面说的命令了

```
git cherry-pick A..B // 摘提交(A, B]
git cherry-pick A^..B // 摘提交[A, B]
```

这里我们选择第二个，因为我们需要 A 提交

```
❯ git cherry-pick 3ce4aae8d99d^..722ba9ec56966187e96
[detached HEAD 942b448] base02 cherry-pick test4
 Date: Tue Oct 28 00:43:20 2025 +0800
 1 file changed, 1 insertion(+)
 create mode 100644 READMEX.md
[detached HEAD 7d07657] base02 cherry-pick test5
 Date: Tue Oct 28 00:43:31 2025 +0800
 1 file changed, 4 insertions(+), 1 deletion(-)
```

顺利执行，main 分支成功创建文件并且 Get 到了提交内容。

## 所摘提交与本地提交冲突

下面是 main 分支文件 A 的内容：

```
# gitTest Base1 第一次提交
# gitTest Base2 第二次提交
# gitTest Base3 第三次提交
# gitTest Base1 第四次提交
# gitTest Base1 第五次提交
# gitTest Base1 第六次提交
# gitTest Base1 测试多分支变基追溯
# gitTest Base1 测试多分支变基追溯02

# gitTest main cherry-pick 提交 1
# gitTest main cherry-pick 提交 2
```

下面是 feat 分支文件 A 的内容：

```
# gitTest Base1 第一次提交
# gitTest Base2 第二次提交
# gitTest Base3 第三次提交
# gitTest Base1 第四次提交
# gitTest Base1 第五次提交
# gitTest Base1 第六次提交
# gitTest Base1 测试多分支变基追溯
# gitTest Base1 测试多分支变基追溯02

# gitTest Base1 cherry-pick 提交 6 测试冲突
```

gitTest Base1 cherry-pick 提交 6 测试冲突 是 feat 的最新提交，我们可以看出与 main 分支的 A 中的后两行发生了冲突，此时执行 cherrpick，将 feat 的提交摘过来，会发生什么？如下
![image.png](https://img.xiaoshidui.top/blog-pic/images/20251028012201978.png)
只需要解决冲突，然后将冲突文件加入到暂存区，然后执行继续摘就行

```git
git add <file_name>
git cherry-pick --continue
```

或者如果你想中止这次操作，可以使用回滚

```git
git cherry-pick --abort
```

## 总结

其实 cherry-pick 是很好理解的，但是要注意一些小细节，比如说在解决冲突后，要把冲突文件添加到暂存区；以及摘来的提交实际上与原分支上的 CommitID 是不同的，这些等等都需要我们在实践中不断学习。加油，LLL。
