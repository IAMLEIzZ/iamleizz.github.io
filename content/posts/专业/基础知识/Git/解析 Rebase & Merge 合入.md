---
title: 解析 Rebase & Merge
date: 2025-10-27T03:19:20+08:00
lastmod: 2025-10-27T03:19:20+08:00
draft: false
tags:
  - Git
  - 后端知识
categories:
author: 小石堆
---
	老 merge 了，但 rebase 弱的一逼，面试被问到，遂复盘
## 场景
小 A 在 feat1 分支开发，他肯定不能直接在 main 上做修改（为了保护项目的安全性），此时，为了防止大量的冲突，小 A 每次开发前都要拉去 main 分支的代码来看看，没啥冲突就直接合入，这样能有效避免积压大量的冲突。  
此时就有问题了，git 中有两种合入代码的方式，分别是 merge 和 rebase，对应命令如下
```git
git merge origin/main
git rebase origin/main
```
到底用哪一个呢？小 A 挠挠头。  
这里先给出一个方案，如果你是 main / master 分支的维护者，尽量使用 Merge，如果你是要维护自己的分支，可以用 Rebase，原因我们娓娓道来。
## GitGraph
GitGraph 是直观的 Git 多分支提交记录的展示，下面是一张示意图
![image.png](http://43.139.219.135:9000/blog-pic/images/20251027174802542.png)
在 GitGraph 中一条分叉就是一个分支，比如说上图，从 C 这个提交分出了一个 Feat 分支，也就是说，Feat 分支拥有含 C 之前的所有 Main 的提交，从 C 之后，Main 和 Feat 形同陌路。
## Merge 合入
OK，我们先简单看看，基于上面这张 GitGraph，如果说在小 A 执行 merge 操作，会发什么？
![image.png](http://43.139.219.135:9000/blog-pic/images/20251027181459117.png)
很直观，如果执行 merge，会基于 main 分支的最新提交和 feat 分支的最新提交，进行合并，然后形成一个新的提交，非常直观。
## Rebase 合入
那么 Rebase 合入是怎么进行的呢？
![image.png](http://43.139.219.135:9000/blog-pic/images/20251027183216003.png)
可以看出，rebase 的合入，是将 main 中多出来的提交信息插入到 feat 分支的 GitGraph 中，直观来看就是整个 feat 分支的提交记录更加清爽直观了，没有那么多分叉了。但是这里有个细节就是，F‘ 和 G’，这里为什么不是 F 和 G 呢？这就是 Rebase 的危险所在，**他会重铸你的 GitLog**。  
我们知道，我们每次的 Git Commit 都会有一个 Hash 值来唯一标识一个分支上的一次提交，而用 Rebase 合入的时候，会将分支上的最新的 main 还未同步的提交进行”修改“，直观来看就是 Commit 的 Hash 值会改变。  
Rebase 合入时， GitCommit 的这个 Hash 值为什么会改变呢？我们先了解一下这个 Hash 值是怎么算出来的。**Git 通过对每个提交包含的元数据（如父提交、作者、提交者、日期和提交信息）以及内容（目录树的快照）进行SHA-1 哈希算法计算得出一次提交的 Hash 值。**  
那么回到问题的本源就能理解了，因为 rebase 要维护线性的提交记录，所以会导致执行 rebase 的分支的最新提交的父提交修改（至少），进而引发 Hash 值的改变。
## 孰优孰劣？
既然 rebase 能让我们的 git 提交信息更加清爽，那我们猛 rebase 不就好了？merge 有啥用？其实不然，收回伏笔，在一般开发中，我们比较推荐的是，在开发分支上rebase，在 main / master 分支上 merge。  
假设现在有这样一种情况：
![image.png](http://43.139.219.135:9000/blog-pic/images/20251027220438077.png)
我们可以看出 Feat2 分支比 main 多了一个 F 提交，而 Feat1 是建立在 Main 分支的 K 提交之上的。此时，我们用 rebase 合入 Feat2 的 F 提交。Main 分支会变成下面这样，进而造成一些麻烦。
![image.png](http://43.139.219.135:9000/blog-pic/images/20251027220657106.png)
## 结论
综上，其实对于自己的分支，你怎么完都无所谓，用 rebase 会让你的分支提交信息更加简洁，但是如果你掌管了 main / master 分支，那最好用 merge 合入代码，以防一些不必要的麻烦。（留个种子，交互式变基等 rebase 的扩展操作后面还要开一章学习）