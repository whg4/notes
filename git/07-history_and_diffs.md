# 07-History&Diffs

[TOC]

## History

### COMMIT

#### 糟糕的commit信息
![image](https://github.com/whg4/notes/assets/37855991/480d5bbc-d15f-455a-a511-ad63efd759d0)

#### 好的commit十分重要

- 好的commit能够帮助保持代码库的维护记录
- 有以下好处：
  - 调试（debugging）和解决重大故障（troubleshooting）
  - 生成releases日志
  - 代码审查（code review）
  - 回滚（rolling back）
  - 将issue或标签（ticket）关联在commit上

#### 一个好的commit message示例
![image](https://github.com/whg4/notes/assets/37855991/a1f58ea4-8364-496b-89f6-6fdf19177c57)

#### 好的commit的构成

- 良好的commit message
- 封装一个逻辑主题（encapsulates one logical idea）
- 不会引入breaking changes
  - i.e. tests pass

### GIT LOG

git log，最基础的用来打印git仓库历史记录的命令。
![image](https://github.com/whg4/notes/assets/37855991/2a78fd3a-71d1-4f58-91ae-08f0602d4e00)

#### GIT LOG --GREP

- 搜索符合正则表达式的commit messages

  - git log --grep <regexp>
  - 可以和其他flags一起使用

- Example：
  ![image](https://github.com/whg4/notes/assets/37855991/e743f246-8a81-40ea-a073-3ada5157ab9f)

#### GIT LOG --DIFF-FILTER

- 选择性地包含或排除一些状态的文件，例如：

- （A)dded, (D)eleted, (M)odified & more...
  ![image](https://github.com/whg4/notes/assets/37855991/32f228e7-f90c-4234-af42-acb778eaf51c)

#### GIT LOG: REFERENCE COMMITS

- ^ 或 ^n
  - 没有参数：==^1，等于第一个父commit
  - n：第n个父commit
- ~ 或 ~n
  - 没有参数：==~1，顺着第一个父commit的第一个回退commit
  - n：顺着第一个父commit的第n个回退commit

注意：^和~可以结合使用


#### REFERENCE COMMIT

- A和B都是节点A的父亲节点，父亲节点从左到右组织
- A是最新的commit
  ![image](https://github.com/whg4/notes/assets/37855991/c46ed466-0969-4325-8b98-6a792c358380)

### GIT SHOW

- 展示commit和内容
  - git show <commit>
- 展示commit中的文件改动
  - git show <commit> --stat
- 从另一commit中查看文件
  - git show <commit>:<file>



## DIFF

- Diff可以在以下内容展示改动（changes）：
  - commits之间
  - 在暂存区域（staging area）和仓库（repository）
  - 工作区（working area）
- 未暂存的改动：
  - git diff
- 暂存的改动
  - git diff --staged

#### DIFF COMMITS AND BRANCHES
![image](https://github.com/whg4/notes/assets/37855991/7f4c99df-14d0-409d-baf1-fd3adbccb142)

#### "DIFF" BRANCHES

- 哪些分支合并到了master，可以被清理
  - git branch --merged master
- 哪些分支还没有合并到master
  - git branch --no-merged master
