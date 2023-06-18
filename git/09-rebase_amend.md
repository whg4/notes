# Rebase，Amend

重新梳理历史（Rewrite History！）

## AMEND A COMMIT

amend可以快速且快捷地对前一个commit增加改动（changes）。

![image](https://github.com/whg4/notes/assets/37855991/4e1d7d50-8892-4916-bf0e-cc925dde8e0f)

## COMMIT CAN'T BE EDITED

- 记住，commit不能被编辑
- 一个commit是其所有数据的SHA值的引用（A commit is referenceb by the SHA of all its data）
- 即使commit在树中所指向的位置相同，作者相同，但是提交日期是不同的
- 一个新的commit会被创建

![image](https://github.com/whg4/notes/assets/37855991/56c77fd3-13dc-44c9-926c-956ebe99c26d)


## WHAT IS REBASE ANYWAY?

- 想象我们的tech_posts和master分支是分开演进的
- 我们不想引入混乱的合并commit在提交历史中
- 我们可以通过pull命令拉取master的最新改动，然后将我们的commit应用在master分支的commit之上，也就是将我们的parent commit改变成master的最新commit

**Rebase = give a commit a new parent**



## REBASE: REWINDING HEAD
![image](https://github.com/whg4/notes/assets/37855991/1e5c6ec1-0c0c-4bc9-940a-5a5cc5932730)

## REBASE: APPLY NEW COMMITS
![image](https://github.com/whg4/notes/assets/37855991/2f466e16-ea0d-43ab-98c9-d744b1139f43)

## MERGE VS REBASE
![image](https://github.com/whg4/notes/assets/37855991/0f976ad7-c387-4813-bd77-02578d2dcbb5)

## INTERACTIVE REBASE (REBASE -I OR REBASE --INTERACTIVE)

- 可交互式rebase会打开一个含有一些“todo”编辑器
  - todo的格式为：<command> <commit> <commit msg>
  - git会将commit以一定顺序进行挑选，或者在编辑或冲突发生时执行一些动作
- 使用快捷方式进行可交互式rebase：
  - `git rebase -i <commit_to_fix>^`
  - ^指定着父commit

## REBASE OPTIONS
![image](https://github.com/whg4/notes/assets/37855991/635e3092-18a8-41ab-8b59-749eaa2b1d2e)



## REBASE EXAMPLE
![image](https://github.com/whg4/notes/assets/37855991/6b8c7a54-7ee0-44db-aa8c-08d6eaf45418)



## TIP: USE REBASE TO SPLIT COMMIT

编辑一个commit可以将其分割成多个commit：

1. 使用rebase -i开启可交互式rebase
2. 将commit标记为edit
3. `git reset HEAD^`
4. `git add`
5. `git commit`
6. 重复（4）和（5）直到工作区是干净的
7. `git rebase --continue`


## TIP: "AMEND" ANY COMMIT WITH FIXUP & AUTOSQUASH

当需要对任意一个commit进行修补时，

1. `git add new files`
2. `git commit --fixup <SHA>`
   1. 这个命令会创建一个新的commit，messages会以“fixup!”开头
3. `git rebase -i --autosquash <SHA>^`
4. git会为你自动生成正确的todos，只要执行save和quit操作即可

## FIXUP & AUTOSQUASH EXAMPLE
![image](https://github.com/whg4/notes/assets/37855991/041cb9f8-cf65-4e96-a28a-8eda8abe772c)

![image](https://github.com/whg4/notes/assets/37855991/4b4c9370-0525-4852-9099-30f31d9d1e5e)



## REBASE --EXEC(EXCUATE A COMMAND)

`git rebase -i  -exec "run-test" <commit>`

exec有两个选项：

1. 当进行可交互式rebase，添加它作为一个command
2. 进行rebasing时使用它作为一个flag



- 当使用作为一个flag时，exec后面的comand会在每个commit被应用时执行
- 它可以用来跑单元测试
- 当command执行失败时，rebase会停止，此时可以排查出现问题的原因



## PULL THE RIP CORD

在rebase进行完成时，如果发现哪里不对，可以通过以下命令还原：

`git rebase --abort`



## REBASE PRO TIP

在进行rebase/fixup/squash/reorder之前：

- 备份当前的分支：
  - `git branch my_branch_backup`

- 如果rebase“成功”了，但是弄错了...，通过以下命令回到初始状态
  - `git reset my_branch_backup --hard`



## REBASE ADVANTAGES

- 可以切分git history
- 可以很容易在代码中修复错误
- 保持git history干净和整洁



## COMMIT EARLY & OFTEN VS GOOD COMMITS

- git最佳实践
  - commit often, perfect later, publish once
- 当在本地工作时：
  - 产生改动时尽可能commit
  - 这样会使你变成更加有生产力
- 在提交工作给公共仓库时：
  - 使用Rebase清理commit记录
 

## WARNING: NEVER REWRITE PUBLIC HISTORY
![image](https://github.com/whg4/notes/assets/37855991/f1765449-fa49-4fb2-a01b-91a9075c89a7)

