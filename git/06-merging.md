# 6-Merging

## 6.1Merge

### 什么是Merge？

从底层来讲，merge commit仅仅就是一个commit。如下图所示：

![merge_commit](https://user-images.githubusercontent.com/37855991/218303873-277c1c90-0738-4925-87e6-533621922a58.png)

### Merge的类型

merge有**两种类型**：

- FAST FORWARD
- NOT FAST FORWARD

#### FAST FORWARD

Fast-forward发生在分支合并时基础分支后只有一个feature分支指向基础分支时。通过`git merge feature`默认会进行Fast-forward合并，如果基础分支后只有一个feature派生，**那么master分支指针会移动到feature分支的最新一个commit上。**

![fast-forward](https://user-images.githubusercontent.com/37855991/218303882-cea3d260-3a98-4335-be18-228253c7b4fe.png)

### NO FAST FORWARD

为了获取合并commit的历史，即使基础分支上没有任何变化，我们还可以通过**not-fast-forward**的方式合并commit，代码如下：

```
git merge --no-ff
```

使用上面的命令会产生一个merge commit，即使创建这个merge commit是没必要的。如下图所示：

![no-ff](https://user-images.githubusercontent.com/37855991/218303894-52f92cce-9d77-43b5-9a7d-230f682ab99b.png)

## 合并冲突（MERGE CONFLICTS）

当尝试将分支进行合并，文件出现分歧时，Git会停止合并直到冲突解决为止。

![merge_conflict](https://user-images.githubusercontent.com/37855991/218303902-78d8e656-65a9-47eb-84e2-cb5d37e6c9ed.png)

### GIT RERERE - REUSE RECORD RESOLUTION

git会保存你解决冲突的解决方案，在下一次出现冲突时会复用相同的解决方法。这对于以下两种情况特别有用：

- 长时间存在的特征分支（例如重构）
- rebasing

可以通过以下命令开始它，添加--global标志将会应用到全部项目

```
git config rerere.enabled true
```

![git_rerere](https://user-images.githubusercontent.com/37855991/218303919-1421b249-3cf7-4b5c-b3f4-8d88bf7d0025.png)

在取消合并后，再次尝试合并，发现git自动应用了之前解决冲突的方法。

![auto_fix_conflict](https://user-images.githubusercontent.com/37855991/218303930-81cd985f-827f-4fc0-8f15-c491ea6be2c2.png)
