title: Git知识总结

categories:
- Git
tags:
- Git

date: 2015/12/26
folder: git
en-title: git-knowledge
---
git是平常软件开发中经常使用的版本控制系统，本文对git的原理、基础和配置方式进行了介绍，同时总结了平时使用过程中常见的git操作。
<!-- more -->

## 1. 概念
1. 集中式版本控制系统：有一个单一的集中管理的服务器，开发人员都可以连接到这台服务器，进行协同工作。
* 优点是系统容易管理和维护
* 缺点是如果中央服务器出现故障，工作将不能进行
2. 分布式版本控制系统：不是只提取最新版本的文件快照，而是将整个代码仓库拷贝到本地，即使服务器故障也能通过本地仓库进行完整的恢复。

## 2. 基础
* git保存数据是保存整个文件系统的一个快照![git-snapshot](https://git-scm.com/book/en/v2/book/01-introduction/images/snapshots.png)

* 其他大部分系统保存的是文件的变更信息![delta](https://git-scm.com/book/en/v2/book/01-introduction/images/deltas.png)

* git大部分操作都是在本地完成的，因此速度很快。即使没有网络，也可以提交文件，之后再上传

* git通过SHA-1（哈希）计算校验和，由40个十六进制数组成的字符串。git数据库中保存的信息都是通过文件内容的hash值进行索引的而不是文件名。

* git的三个工作区域
 * git仓库：用来保存项目的数据以及数据库，当从其他地方clone项目时，拷贝的就是这里面的数据。
 * 工作目录：存放的是项目某个版本的内容，存放于磁盘可供修改等操作，来源于git仓库中的压缩数据库。
 * 暂存区域： 是一个文件，保存了将提交的文件信息。

* git工作流程
 * 在工作目录中修改文件(modified)
 * 将修改的文件暂存在暂存区域(staged)
 * 提交更新，将暂存区的文件存储到git仓库中(commited)

## 3. git配置
git config

* --system：系统上所有用户及仓库的通用配置
* --global：针对当前用户的配置
* --config：针对当前仓库的配置

上述配置具有优先级，由上往下递增，高优先级的覆盖低优先级的配置。

第一次使用时会配置个人信息
`$ git config --global user.name yourname`
`$ git config --global user.email your@email.com`
这些信息会写入每一次提交中，该配置只需配置一次。若想对某个项目使用特定的用户信息，通过git config来配置。
可通过git config --list查看所有配置信息，也可通过如git config user.name来查看用户名

## 4. git基础操作

### 4.1 创建git仓库
* 在已有项目中创建git仓库：通过`$ git init`命令创建一个.git子目录，里面存储了git仓库初始化时的必须文件，但不包括项目文件。
* 克隆仓库：通过`$ git clone theUrlYouWantToClone`，会克隆该仓库的所有版本的文件而不是最新版本的文件。执行该操作后会在当前目录下创建一个与该仓库同名的目录，并在该目录下初始化一个`.git`文件夹，将远程仓库中的数据拉取到该文件夹，并从中读取最新版本的项目文件，拷贝至其同级目录。
 * 可通过`$ git clone theUrlYouWantToClone myProjectName`修改仓库名称

**`theUrlYouWantToClone`**可支持多种协议，git中常用到的是https、git和SSH协议。

### 4.2 文件状态
文件具有两种状态**已跟踪**和**未跟踪**。

* 已跟踪的文件指的是已被纳入版本控制的文件，它们可能处于*未修改*、*已修改*、*已暂存*等状态。
* 将文件从git中移除、新建立一个文件等操作产生的文件都会处于未跟踪状态。
git版本控制下文件的生命周期
![life-cycle](https://git-scm.com/book/en/v2/book/02-git-basics/images/lifecycle.png)

### 4.3 git add命令
####  4.3.1 跟踪新文件
通过`$ git add filename`、`$ git add pathname`可分别跟踪某个文件和某个路径下的所有文件，或者通过`$ git add .`来跟踪所有文件。
#### 4.3.2 暂存已修改文件
与上述方法相同

因此，`git add`具有多种功能，可理解为**向下次提交中添加内容**

### 4.4 git status命令
该命令用于查看文件状态
也可通过`$ git status -s`或者`$ git status --short`查看简写的文件状态。

```
$ git status -s
M README
MM Rakefile
A  lib/git.rb
M  lib/simplegit.rb
?? LICENSE.txt
```
各种记号的解释如下：

* ??：未跟踪文件
* A：新添加到暂存区的文件
* _M：文件被修改但未添加到暂存区
* M_：文件被修改且已添加到暂存区
* MM：文件被修改且已添加到暂存区后又被修改

### 4.5 .gitignore文件
忽略文件的具体格式可见[忽略文件](https://git-scm.com/book/zh/v2/Git-%E5%9F%BA%E7%A1%80-%E8%AE%B0%E5%BD%95%E6%AF%8F%E6%AC%A1%E6%9B%B4%E6%96%B0%E5%88%B0%E4%BB%93%E5%BA%93#忽略文件)

### 4.6 git diff命令
git diff命令用于查看文件的修改，与git status相比可以具体显示修改的内容。

* `$ git diff`显示的是已修改但未暂存的文件内容
* `$ git diff --cached`或`$ git diff --staged`查看已暂存但未提交的内容

### 4.7 git commit命令
通过git commit命令可将暂存区内的文件进行提交，而未暂存的文件则不会被提交。
可用`$ git commit`命令按提示完成提交或直接通过`$ git commit -m "yourComments"`进行提交说明并提交。
若想对所有已暂存和未暂存的文件进行提交可使用`$ git commit -a -m "yourComments"`进行提交，这样可简化操作步骤。

### 4.8 移除文件
移除文件需要git remove操作。

* 通过`$ git rm fileName`即可将文件移出git版本控制并同时删除工作区中的文件，之后执行commit操作提交即可。
* 若是直接手动将文件删除，此时文件将会处于未暂存状态，通过git add或git rm操作便可进入已暂存状态，之后执行commit操作即可。
* 若文件进过修改并送入暂存区，需要用`$ git rm -f fileName`才能强制删除或通过上述第一种方式删除。
* 若是要将文件从git版本控制中删除但又不想从工作区中删除，则可通过`$ git rm --cached fileName`命令来实现。

### 4.9 移动文件
移动文件操作通过`$ git mv fileFrom fileTo`来实现，该操作执行过程实际上是执行了删除fileFrom文件之后添加fileTo文件。

### 4.10 撤销操作
1. 修改提交文件
 * 若提交后发现漏掉其他文件，可通过`$ git commit --amend`操作将当前暂存区的文件加入上次提交中，相当于只存在一次提交。
 * 若是想修改提交文件的提交信息，可通过`$ git commit --amend -m "newComments"`

2. 取消暂存文件
 * 通过`$ git reset HEAD fileName`可将文件从暂存区中移除，进入未暂存状态。

3. 取消已修改文件
 * 通过`$ git checkout -- fileName`将已修改的文件还原至上一次未修改时的状态。

### 4.11 远程仓库
git项目的协作时会用到远程仓库，当克隆一个项目时，会有一个默认的名为**origin**的远程仓库。

* 可通过`$ git remote`查看所有远程仓库或`$ git remote -v`查看所有远程仓库的具体信息。通过`$ git remote show remoteName`查看指定远程仓库的详细信息。
* 通过`$ git remote add shortNameForTheRemoteRepository RepositoryUrl`
* 通过`$ git fetch shortName`可获取该远程仓库中的所有信息，但不会合并或修改当前工作区的文件
* 通过`$ git pull`命令获取远程仓库的数据并合并到当前分支，默认情况下，本地的master分支会跟踪远程仓库的master分支。
* 通过`$ git push remoteName branchName`将当前分支推送到指定远程仓库的指定分支上。
* 重命名远程仓库可使用`$ git remote rename originalName currentName`
* 删除远程仓库可使用`$ git remote rm repositoryName`

## 5. git分支
使用分支可以将开发工作从主线上进行分离，git的默认分支是**master**分支。git使用**HEAD**指针指向当前分支，可看做当前分支的一个别名。
### 5.1 创建分支
创建分支实际上就是创建了一个指向当前项目的指针，通过`$ git branch`命令可以查看当前分支，`$ git branch branchName`用于创建分支。

### 5.2 切换分支
命令`$ git checkout branchName`用于切换分支，切换分支后当前的工作区的文件也会随之改变。
**git分支是以提交的文件为基础的，一般情况下在修改文件后只有当提交了文件才能切换到另一个分支，但也有特殊情况，以branch1和branch2两个分支为例，若在branch1中添加了新文件，此时切换到branch2分支可以看到该文件，若此时提交则该文件属于branch2分支，再次切回branch1中不会保留该文件**

### 5.3 删除分支
通过`$ git branch -d branchName`可以删除该分支

### 5.4 分支合并
首先通过`$ git checkout branchName`切换到某一分支，如master分支，接下来执行`$ git merge branchName`将对应分支合并到master中。
**合并过程中如果两个分支对同一个文件进行了操作，则会产生冲突，需要手动解决，产生冲突的内容会在工作区中的相应文件中被标记出来，需要自己去判断保留哪一份修改，之后执行提交即可。**

### 5.5 跟踪远程分支
若想跟踪远程分支可以通过`$ git checkout --track remoteRepository/remoteBranch`，也可通过`$ git checkout -b branchName remoteRepository/remoteBranch`新建分支并跟踪远程分支，两者的不同点只是新建分支的名字不同。
若本地已有分支，则可通过`$ git branch -u remoteRepository/remoteBranch`添加或修改跟踪。
通过`$ git branch -vv`命令可以查看远程分支的跟踪信息。

### 5.6 删除远程分支
命令`$ git push remoteRepository --delete remoteBranch`可用于删除远程分支。





