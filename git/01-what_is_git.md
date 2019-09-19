# 1-什么是Git？

> Git是一个版本分发控制系统

## 1.1Git是怎么存储信息的？
Git的核心，就像是一个存储键值对的仓库。key为要存储数据的hash，value为要存储的数据data。

![image](https://user-images.githubusercontent.com/37855991/218303082-f9e25231-7e27-4335-ba00-7f2c4232e38f.png)

### key是这么生成的？

key是通过SHA1算法生成的，给定一片段数据，通过SHA1算法会生成40位的16进制数字。当给定相同的输入，SHA1算法总是会输出相同的40位16进制数。

### value是什么？
git将压缩后的原始data数据存储在blob中，blob会有一个metadata头部，blob的具体结构如下图：

![git_value](https://user-images.githubusercontent.com/37855991/218303140-c06f9d3e-5e43-4183-b23d-0f3ab41ff9cb.png)

### 底层原理 - git hash-object
我们可以向git询问内容的SHA1值，具体做法如下：

![g_sha1](https://user-images.githubusercontent.com/37855991/218303162-6f08ea4d-65a7-405a-bef6-296d5d07772a.png)

通过包含metadata的内容生成SHA1，代码如下：

![g_sha1_meta](https://user-images.githubusercontent.com/37855991/218303179-090ce1c1-6d3b-4e3e-8526-1d8d9c77da0a.png)

### Git在哪里存储数据？
当我们通过`git init`初始化Git仓库时，Git会创建一个**.git**目录来存放和我们仓库有关的数据。

![store_data](https://user-images.githubusercontent.com/37855991/218303191-eb73c0be-f909-4856-8560-131a91415646.png)

### Blobs存储在哪里？
git将blobs存储在**.git**目录下的**objects**子目录中，具体如下图所示

![blobs_store](https://user-images.githubusercontent.com/37855991/218303202-d11290b5-0c2c-47fd-bc05-b5e7e68a1432.png)

## 1.2我们还需要其他东西
通过上面的了解我们发现，blob还缺失了以下重要的信息。

1. **文件名**
2. **目录结构**

Git将这些信息存储在**tree**中。

### Tree
**tree**包含了一些指针（使用SHA1来确定一个指针)：

1. blobs指针
2. trees指针

还包含了一些**metadata**：

1. 指针的**类型**（blob指针/tree指针)
2. **文件名**或目录名
3. **mode**模式（可执行文件，符号链接，...）

### Tree的表格结构图

![t_tree_stru](https://user-images.githubusercontent.com/37855991/218303228-28bb70b1-422e-4f8b-8d0b-2520e512dd7f.png)

### Tree指向Blobs和Trees

![g_tree_stru](https://user-images.githubusercontent.com/37855991/218303232-fca28ac8-95f2-4409-86e8-36d440f151ce.png)

### 相同的内容只存储一次！
如果不同tree中的blob指针指向的blob内容相同，那么他们将会指向同一个blob。


## 1.3其他优化-Packfiles, Deltas
![o_optim](https://user-images.githubusercontent.com/37855991/218303253-6e11d14a-fd81-4959-89e9-086fea241245.png)
