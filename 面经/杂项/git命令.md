# git常用命令

git init 项目初始化
git clone 拉取项目
git add . 添加到暂存区
git commit -m 添加commit信息
git push 将本地分支推送到服务器上去
git pull origin master 本地与服务器端同步

git log 查看日志
git status 查看当前状态
git tag 查看版本号
git diff 查看尚未提交的更新

# git fetch

git fetch 是从远程分支拉取代码。看似与 git pull 类似，而在大多数情况下 git pull 的含义是一个 git fetch 紧接着一个 git merge 命令。即 git pull 是 git fetch 和 git merge 的两步的和。

## 四种基本用法

1. **git fetch**

 这将更新 git remote 中所有的远程 repo 所包含分支的最新 commit-id，并将其记录到 .git/FETCH_HEAD 文件中。

2. **git fetch remote_repo**

 这将更新名称为 remote_repo 的远程 repo 上的所有 branch 的最新 commit-id，并将其记录到 .git/FETCH_HEAD 文件中。

3. **git fetch remote_repo remote_branch_name**

 这将更新名称为 remote_repo 的远程 repo 上的 remote_branch_name 分支，并设定当前分支的 FETCH_HEAD 为该分支.

4. **git fetch remote_repo remote_branch_name:local_branch_name** 

这将更新名称为 remote_repo 的远程 repo 上的 remote_branch_name 分支，并在本地创建 local_branch_name 本地分支保存远端分支的所有数据。

> FETCH_HEAD： 是一个版本链接，记录在本地的一个文件中，指向目前已经从远程仓库取下来的分支的末端版本。

# git rebase

当执行rebase操作时，git会从两个分支的共同祖先开始提取待变基分支上的修改，然后将待变基分支指向基分支的最新提交，最后将刚才提取的修改应用到基分支的最新提交的后面。

## 举例

代码仓库中有master和feature两个分支。

```c++
a - b - c - d    Master
     \
      e - f      Feature
```

当在 Feature 分支上执行`git rebase master`时，git 会从 Master 和 Featuer 的共同祖先 b 开始提取 Feature 分支上的修改，也就是 e 和 f 两个提交。然后将这两个提交指向 Master 分支的最新提交 d 上，也就是把提取的 e 和 f 接到 d 后面，但这个过程是删除原来的 e 和 f ，生成新的 e' 和 f' ，它们的提交内容一样，但commit id不同。操作完成后，代码库就变成了下面的样子。

```
a - b - c - d            Master
     \
      c - d - e' - f'    Feature
```

**总结：** rebase，即变基，可以直接理解为改变基底。Feature 分支是基于 Master 分支的 b 拉出来的分支，Feature 的基底是 b。而 Master 在 b 之后有新的提交，就相当于此时要用 Master 上新的提交来作为 Feature 分支的新基底。实际操作为把 b 之后 Feature 的提交存下来，然后删掉原来这些提交，再找到 Master 的最新提交位置，把存下来的提交再接上去（新节点新commit id），如此feature分支的基底就相当于变成了M而不是原来的B了。

> 注意，如果 Master 上在 b 以后没有新提交，那么就还是用原来的 b 作为基，rebase操作相当于无效，此时和git merge就基本没区别了，差异只在于git merge会多一条记录 Merge 操作的提交记录。

## 推荐场景

* 自己单机的时候，拉公共分支最新代码的时候使用rebase，也就是`git pull -r`或`git pull --rebase`。这样的好处很明显，提交记录会比较简洁。但有个缺点就是 rebase 以后我就不知道我的当前分支最早是从哪个分支拉出来的了，因为基底变了嘛，所以看个人需求了。
* 往公共分支上合代码的时候，使用 merge。如果使用 rebase，那么其他开发人员想看主分支的历史，就不是原来的历史了，历史已经被你篡改了。举个例子解释下，比如张三和李四从共同的节点拉出来开发，张三先开发完提交了两次然后 merge 上去了，李四后来开发完如果 rebase 上去（注意李四需要切换到自己本地的主分支，假设先 pull 了张三的最新改动下来，然后执行 git rebase 李四的开发分支，然后再 git push 到远端），则李四的新提交变成了张三的新提交的新基底，本来李四的提交是最新的，结果最新的提交显示反而是张三的，就乱套了。

## git rebase -i用法

使用下述命令进入Interact交互界面，选择变基操作。

```
$ git rebase -i [startPonit] [endPoint]
```

> 注意：
>
> * 这是前开后闭区间，这里的 [startPonit] 是指需要操作的 commit 的前一个 commit；
> * [endPoint] 默认省略，即默认表示从起始 commit 一直到最后一个，但是一旦填写了 [endPoint]，则表示 [endPoint] 后面的commit 全部不要了；

|  命令  | 缩写 |                            含义                             |
| :----: | :--: | :---------------------------------------------------------: |
|  pick  |  p   |                        保留该 commit                        |
| reward |  r   |          保留该 commit，但需要修改该 commit 的注释          |
|  edit  |  e   |   保留该 commit，但需要停下来修改该提交（不仅仅修改注释）   |
| squash |  s   |               将该 commit 合并到前一个 commit               |
| fixup  |  f   | 将该 commit 合并到前一个 commit，但不要保留该提交的注释信息 |
|  exec  |  x   |                        执行shell命令                        |
|  drop  |  d   |                        丢弃该 commit                        |

## rebase解决冲突

1. 先解决冲突
2. git add .
3. git rebase --continue

# git cherry-pick

git cherry-pick可以理解为"挑拣"提交，它会获取某一个分支的单笔提交，并作为一个新的提交引入到你当前分支上。 当我们需要在本地合入其他分支的提交时，如果我们不想对整个分支进行合并，而是只想将某一次提交合入到本地当前分支上，那么就要使用git cherry-pick了。将指定的提交应用于当前分支，会在当前分支产生一个新的提交，并且它们的哈希值是不一样的。

## 基本用法

```
git cherry-pick [<options>] <commit-ish>...

常用options:
    -e, --edit            打开外部编辑器，编辑提交信息
    -n, --no-commit       只更新工作区和暂存区，不产生新的提交
    -x                    在提交信息的末尾追加一行(cherry picked from commit ...)，方便以后查到这个提交是如何产生的
    -s，--signoff         在提交信息的末尾追加一行操作者的签名，表示是谁进行了这个操作
    --quit                退出当前的chery-pick序列
    --continue            继续当前的chery-pick序列
    --abort               取消当前的chery-pick序列，恢复当前分支
```

## 举例

代码仓库中有master和feature两个分支。

```c++
a - b - c - d    Master
     \
      e - f - g  Feature
```

用如下命令将提交f应用到master分支。

```c++
# 切换到master分支
$ git checkout master
 
# Cherry pick操作
$ git cherry-pick f
```

操作完成后，代码库就变成了下面的样子，master分支的末尾增加了一个提交f'。

```
a - b - c - d - f'   Master
     \
      e - f - g      Feature
```

git cherry-pick命令的参数，不一定是提交的哈希值，分支名也是可以的。如果是分支名，则表示转移该分支的最新提交。如下命令就表示将Feature分支的最近一次提交g，转移到当前分支。

```
$ git cherry-pick Feature
```

## 更多用法

1. Cherry pick 支持一次转移多个提交。

下述命令将 A 和 B 两个提交应用到当前分支，这会在当前分支生成两个对应的新提交。

```
$ git cherry-pick <HashA> <HashB>
```

如果想要转移一系列的连续提交，可以使用下列的简便语法。

```
$ git cherry-pick A..B
$ git cherry-pick A^..B 
```

前一条命令可以转移从 A 到 B 的所有提交。它们必须按照正确的顺序放置：提交 A 必须早于提交 B，否则命令将失败，但不会报错。

但是，使用前一条命令，提交 A 将不会包含在 Cherry pick 中。如果要包含提交 A，就需要使用后一条命令。

以下两个命令作用相同，表示应用所有提交引入的更改，这些提交是 branchname 的祖先但不是 HEAD 的祖先

```
git cherry-pick ..<branchname>

git cherry-pick ^HEAD <branchname>
```

还是使用之前的例子，如果使用`git cherry-pick ..Feature`或者`git cherry-pick ^HEAD Feature`，那么会将属于 Feature 的祖先但不属于 Master 的祖先的所有提交引入到当前分支 Master 上，并生成新的提交。

2. 代码冲突

如果操作过程中发生代码冲突，Cherry pick 会停下来，让用户决定如何继续操作。

* 如果要继续 Cherry pick ，则首先需要解决冲突，通过`git add .`将文件标记为已解决，然后可以使用`git cherry-pick --continue`命令，继续进行 Cherry pick 操作。
* 如果要取消这次 Cherry pick ，则使用`git cherry-pick --abort`，这种情况下会放弃合并，当前分支恢复到 Cherry pick 前的状态。
* 如果要退出这次 Cherry pick ，则使用`git cherry-pick --quit`，这种情况下不会回到操作前的样子，当前分支中未冲突的内容状态将为`modified`。