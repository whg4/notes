# 2-Commits

## 2.1什么是Commit？

### Commit对象

一个**commit**指向：

- 一个tree

同时它还包含metadata：

- **作者和提交者**
- **日期**
- **message信息**
- **父commit（一个或多个）**

**commit的SHA1值**是根据它的全部信息生成的。

### Commit表格形式示例

![commit](https://user-images.githubusercontent.com/37855991/218303349-93256ba3-ce7b-4ae3-b0e2-3bd5db7af954.png)



### Commit指向父Commits和Trees

![commit_g](https://user-images.githubusercontent.com/37855991/218303362-501ffd87-7d98-4251-bad4-600fc65c758b.png)

## 2.2一个Commit是代码的快照

### 提交一个commit

初始化Git仓库，在当前目录添加一个hello.txt文件，提交一个commit，具体过程如下

![make_acommit](https://user-images.githubusercontent.com/37855991/218303383-73ce802e-3bb5-4738-b974-83017ccba9bb.png)

通过`tree .git/objects`命令，我们可以发现多出了三个hash值。

![look_in_objects](https://user-images.githubusercontent.com/37855991/218303391-e7645af3-2301-498f-af02-3399eed4ef4e.png)

### 查看hash-object的内容

通过`cat 文件名`查看某个hash-object的内容，我们发现其内容是压缩后的数据。

![cat_object](https://user-images.githubusercontent.com/37855991/218303407-afcb9342-81c6-4843-af27-4822c223b05c.png)

为了查看hash-object的信息，Git还提供了我们两个命令来查看hash-object的信息，命令如下

```
git cat-file -t hash值 # -t 表示打印hash-object的类型
git cat-file -p hash值 # -p 标志打印hash-object的内容
```

分别查看三个hash-object的信息，得到以下结果

#### Blob对象

![blob_object](https://user-images.githubusercontent.com/37855991/218303419-cd6105eb-85c3-4976-9898-51368129483d.png)

#### Tree对象

![tree_object](https://user-images.githubusercontent.com/37855991/218303429-95ecc9e3-22ca-4284-891f-661f7f0df0a4.png)

#### Commit对象

![commit_object](https://user-images.githubusercontent.com/37855991/218303437-b212449d-5746-46af-9923-8b617d74d10b.png)

通过查看hash-object的数据，验证了Git的Blob、Tree、Commit对象的结构。

### 为什么我们不能改变Commits?

如果你改变了当前commit的任何数据，当前commit会有一个新的SHA1值。由commit对象的构成元素可知，即使文件没有发生变化，但是commit的创建时间会发生变化。

## 2.3引用

### 引用Commits的指针

有三个指针指向引用着commits：

- Tags
- Branches
- HEAD - 一个指向当前commit的指针

![pointer_to_commit](https://user-images.githubusercontent.com/37855991/218303453-00f02d24-27f1-473d-bd79-10313afe034f.png)

### 引用-底层细节

通过`tree .git`查看**.git**目录的结构，如下图所示

![re_stru](https://user-images.githubusercontent.com/37855991/218303466-749c6a6e-7f12-41ca-9cab-bd49173e12aa.png)

**refs/heads目录保存了branches指针。**

使用`git log --oneline`查看当前commit的hash值，然后使用`cat .git/refs/heads/master`查看master指针的值，可以发现master指针指向了**Initial commit**。

![branch_point](https://user-images.githubusercontent.com/37855991/218303478-8a225e39-cc92-499c-a8c4-75a4356b40f7.png)

最后，通过`cat .git/HEAD`命令读取HEAD的值，发现**HEAD指向了当前的master分支**。

![head_pointer](https://user-images.githubusercontent.com/37855991/218303493-d1da9f29-743c-4a5b-bd16-a15892c40ff5.png)
