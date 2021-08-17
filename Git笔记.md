[toc]

# Git 核心概念和基础操作

## Git 的工作区域

* Working Directory
* Stage
    * 文件在 .git/index，没什么内容
* Repository (Git Directory / History)
    * Repository 里面有提交的所有版本的数据，其中的 HEAD 指向最新放入 repo 的版本
    * `vim .git/HEAD` 可以打开 HEAD 文件
    * 初始 HEAD 文件的内容：ref: refs/heads/master
* Remote Directory

## HEAD

HEAD 实际上指向的是某个具体的 commit 记录

所以比较当前 commit 记录和前一个 commit 记录内，文件的不同的时候，可以使用 `git diff HEAD HEAD^`

## 连接本机仓库

这里的「本机仓库」实际上也是远程仓库，只是可以在本机用绝对地址访问。

```bash
git clone --bare file://绝对地址 [自定义命名]
```

注意：

- `--bare` 表示不带工作区的裸仓库，以后 push 的时候会方便一点
- 这里使用了智能协议，所以要在「绝对地址」前添加 `file://`
- 后面还可以添加一个自定义命名，也就是 clone 之后的文件夹的名称
	- 如果不添加自定义命名，且没有使用 `--bare` ，文件夹的名称就是 clone 仓库文件夹的名称
	- 如果不添加自定义命名，且使用了 `--bare` ，则名称为 clone 仓库的名称加上后缀 `.git`

## 连接远程仓库

`git remote -v` 查看连接了哪些远程仓库

方法一：创建远程仓库后，直接在本地 `git clone [远程仓库地址]` 即可

方法二：

1. `git init` ：在本地的某个文件夹内，创建一个仓库
2. `git remote add origin [远程链接]` ：添加远程仓库的链接，并将远程仓库命名为 origin（一般都要命名为 origin）

## 将本地项目同步到远程仓库

如果在本地已经有了一个项目，需要连接远程，并将本地内容上传到远程仓库：

**在本地项目没有被 Git 管理的情况下**：

1. 在远程创建一个仓库
2. 在本地随便找一个地方，使用 `git clone [远程地址]` 
3. `cd` 刚刚 clone 到本地的目录（文件夹）
4. `mv ./* ./.git [需要上传的本地项目的目录]` 将该目录下的所有文件，以及 .git 目录拷贝到自己已经创建了项目的本地目录下
5. 最后在需要上传的项目的目录使用 add 加上 commit 来上传

原理：.git 目录下有远程仓库等 Git 需要的所有信息

***

**在本地项目已经被 Git 管理的情况下**：

首先，在本地项目中，连接远程仓库：

```bash
git remote add origin [远程地址]
```

此时可以尝试 `git push origin --all` 来将本地项目的所有分支推送到远程。

如果远程没有该名称的分支，就没有问题；如果远程有该分支，很可能会出现冲突错误，比如 master 基本上都会出错。

此时有 merge 和 rebase 两种方案来解决冲突，这里用 merge 方式来演示：

```bash
git merge --allow-unrelated-histories [origin/远程分支的名称]
```

因为本地的该分支和远程分支是两条独立的分支，所以 merge 的时候要使用 `--allow-unrelated-histories` 来合并。

这样就完成了，如果还有文件冲突，就根据需求自行解决即可。

补充：其实也可以先删除本地的 .git 目录，然后再按照之前的方法操作，但是不推荐。

## 忽略文件 gitignore

* 创建 .gitignore 文件
* `#` 用于注释
* `*` 代表任意多个字符，`?` 代表一个字符，`[abc]` 代表可以是 a 或 b 或 c，`{str1, str2}` 代表可以是 str1 或 str2
* `*.txt` 表示所有以 txt 后缀结尾的文件都被忽略，如果还有 `!abc.txt`，表示忽略的时候，排除掉 abc.txt
* `*.[aB]` 表示忽略所有 .a 和 .B 结尾的文件
* `dbg` 忽略所有 dbg 样式的<u>文件和目录</u>
* `dbg/` 只忽略名为 dbg 的目录，不忽略名称为 dbg 的文件
    * `build/` 忽略 build/ 文件夹及其下的所有文件
* `/dbg` 只忽略当前目录下的 dbg 文件和目录，子目录的 dbg 不在忽略范围内
    * `/TODO` 仅在当前目录下忽略 TODO 文件，但不包括子目录下的 subdir/TODO
* `doc/*.txt`：忽略 doc/notes.txt, 不包括 doc/server/arch.txt

## 安装配置 Git

`yum install -y git` 安装到 Cent OS

配置文件的目录在：~/.gitconfig（或 ~/.config/git/config）

`git config --global --list` 查看全局配置

`git config --global user.name "zhou"` 配置用户名

`git config --global user.email "imzhouhepeng@outlook.com"` 配置邮箱

## SSH 配对 GitHub

1. 输入 `ssh` 查看是否安装了 SSH，还可以去 /etc 目录下查看是否有 ssh-keygen
2. 输入 `ssh-keygen -t rsa`，使用 rsa 算法生成 id_rsa（密钥）和 id_rsa.pub（公钥）
    * 本地的 id_rsa 密钥会跟 GitHub 上的 id_rsa.pub 公钥进行配对，授权成功才可以提交代码
3. 在 ~/.ssh 目录下找到 id_rsa.pub，并将其内容添加到 GitHub 上
    1. 找到 SSH and GPG keys
    2. 点击 New SSH key
    3. 复制到 Key 栏
4. 可以在命令行输入 `ssh -T git@github.com` 查看是否添加成功
5. clone 的时候，记得复制的链接是 SSH 的 git@github.com 格式的那个



# 文件相关操作

## 重命名和删除文件

重命名文件：

```bash
git mv [原名称] [新名称]
```

删除文件：

```bash
git rm [文件名]
```

也可以在工作目录（工作区）下重命名或删除文件，然后再 add 到 Stage 中

## 让暂存区的文件恢复到和 HEAD 相同的状态

如果 Working Directory 中的文件比 Stage 中的文件更好，此时需要抛弃 Stage 中的文件的话：

```bash
git reset HEAD -- [文件名] # 指定文件，可以指定多个，以空格为分隔
git reset HEAD # 所有文件
```

## 将工作区的文件恢复到和暂存区相同的状态

将某文件从 Working Directory `git add` 到 Stage 后，又继续在 Working Directory 修改该文件。

但是改了之后，觉得还是 Stage 中的该文件比较好一点。

所以想抛弃刚刚在 Working Directory 对其的新修改，还原到之前已经被 add 到 Stage 的该文件的状态：

```bash
git checkout -- [文件名或 . 表示全部文件]
```

# Stash（暂时藏起来）

## git stash 指令

查看暂时收起来的 stash：

```bash
git stash list
```

收起，并添加 message（信息）：

```bash
git stash save "自定义信息"
```

查看收起来的 stash 的改动：

```bash
git stash show -p stash@{数字} # 比如 stash@{3}；如果省略，表示最近的一个，也就是 stash@{0}
# 如果不加 -p，就是无法看到完整的变动
```

恢复，但是仍然保留记录（也就是 `git stash list` 依旧可以查看到）：

```bash
git stash apply stash@{数字} # 比如 stash@{3}；如果省略，表示最近的一个，也就是 stash@{0}
```

恢复，且删除该条记录：

```bash
git stash pop stash@{数字} # 比如 stash@{3}；如果省略，表示最近的一个，也就是 stash@{0}
```

删除某个 stash 记录：

```bash
git stash drop stash@{数字}
```

清空所有 stash 记录：

```bash
git stash clear
```

## git stash 使用场景

**场景1：在修改当前某文件的时候，有紧急的需求需要对当前这个文件立刻进行修改并提交**

我们需要暂时收起当前的修改，也就是回到修改之前这个文件的状态。然后处理紧急需求并提交，最后再恢复收起来的修改，继续编辑这个文件。（注意，在这个场景下，这个文件在 HEAD 中是存在的）

首先，暂时藏起来：

```bash
git stash
```

然后完成紧急需求，最后再将暂时藏起来的修改进行恢复：

```bash
git stash pop
```

这一步可能有冲突，根据情况自行修改即可。

***

**场景2：在一个错误的 branch（分支）上编辑了内容**

如果是 *新建的文件*，需要先将新建的文件，通过 add 指令，放到 Stage(Index) 上：

```bash
git add [文件名或 .]
```

如果是已有文件有了新编辑的内容，可以通过 stash 暂时将新内容收起来（新建的文件在 add 之后才能这样操作）：

```bash
git stash
```

然后，切换到正确的分支，再将收起的内容，重新放回到这条正确的分支上：

```bash
git stash pop
```

# Branch 分支相关基础

## `git branch` 和 `git checkout`

`git branch dev`：新建 dev 分支（但是位置还没有移动到 dev 分支）

`git checkout dev`：移动到 dev 分支

创建新分支，拷贝旧分支的内容到新分支上（最后还会移动到新分支）。旧分支可以是远程分支；旧分支省略时，表示拷贝当前所在位置的分支。

```bash
git checkout -b [新分支] [旧分支]
```

`git branch`：查看本地分支，结果中，带上星号的就是当前所处的分支

`git branch -r` ：只查看远程分支

`git branch -a` ：查看本地和远程的所有分支

`git branch -av` ：`-v` 表示查看 branch 的标识，`-av` 表示查看所有 branch 并列出标识

回到上一个分支：

```bash
git checkout -
```

重命名当前分支

```bash
git branch -m [当前所在分支的新名称]
git branch -m [某分支的名称] [某分支的全新名称]
```

删除本地 dev 分支（如果当前位置在 dev 分支就不能删除，要先 checkout 到其他分支才行）：

```bash
git branch -d dev # 如果完成了 merge，就可以直接删除
git branch -D dev # 如果还没有 merge，可以这样强制删除
```

## 将分支推送到远程仓库，以及删除远程分支

推送到远程仓库：

```bash
git push [origin] [branch的名称]
```

（**只是加个冒号**）删除远程分支，并保留本地分支：

```bash
git push [origin] :[远程branch名称]
```

## 本地分支的名称和远程分支的名称不同的情况

在本地仓库创建一个 local_branch 分支，并连接远程的 dev 分支：

```bash
git checkout -b local_branch origin/dev
```

推送的时候，如果要推送到远程的 dev 分支：

```bash
git push origin HEAD:dev
```

推送的时候，如果要推送到远程的同名（这里是 local_branch）分支：

```bash
git push origin HEAD
```

# 合并分支

## merge 方式

> 推荐场景：将其他分支，合并到 master 分支上

### merge 基础

先切换到最终要汇总的分支，比如：`git checkout master`

`git merge dev`：如果现在的位置是 master 的话，就会把 dev 分支合并到 master 中

也可以合为一步，将 dev 分支合并到 master 上：

```bash
git merge dev master
```

### 使用 merge 解决冲突

获取远程（remote）的最新修改，并 merge 到自己的分支上，然后修改冲突文件后，再 push：

1. `git fetch origin` 获取 origin（远程仓库）的数据，但是不会下载最新的文件（也就是不会更新文件）

2. `git merge origin/dev` 将远程的 origin 的 dev，合并到本地的当前分支，此时才会在本地下载最新的文件
3. 此时，会有文件冲突的提示。自己根据需求，确定最终的文件，然后再 `git add .` 、`git commit -m "..."` 和 `git push origin dev` 来完成最终修改

上面的 fetch + merge 的步骤，可以简化为：

```bash
git pull [origin（可省略）] [branch名称（可省略）]
```

然后自行修改有冲突的部分，再 add + commit + push 即可。 

`git merge --abort` 可以恢复到 merge 之前的状态。

## rebase 方式

### rebase 基础

**千万不要对公共的 commit 使用 rebase，只能在自己个人的环境下使用**。

> 推荐场景：将远程的分支，放到本地分支
>
> 禁止场景：和团队协作（包括需要集成 / 汇总）相关的环境下，不要使用 rebase，因为它会改变 commit 记录
>
> 总结：和集成（汇总）分支无关，也就是个人开发的环境下使用；其他情况下不要使用

使用方法，参考 merge 即可。

> merge 会有多个分支记录，而 rebase 将之前的 commit 纪录，整合到当前分支上，所以将远程的分支放到本地的时候，推荐使用 rebase

1. `git fetch origin` 获取远程信息
2. `git rebase origin/dev dev` 将远程的 dev 分支，放到本地的 dev 上
3. `git push origin` 推送到远程仓库中（没有冲突的情况下）
4. 如果是在出现冲突的场景下，要使用 `git push -f origin` ，也就是强制推送
	- 必须注意： `-f` 参数尽量禁止在集成（汇总）的分支上使用
	- 因为强制推送的时候，会将远程的 commit 记录全部替换为本地的 commit 记录
	- 这种情况下，如果本地没有 rebase 远程的记录，也就是没有把远程的记录放在本地上的话，就会造成远程记录永久丢失

### 使用 rebase 解决冲突

如果使用 git rebase 的时候出现了冲突，就先根据需求，解决冲突，然后保存文件。

将解决了冲突的文件放到 Stage 中：

```bash
git add .
```

然后继续执行 rebase 操作：

```bash
git rebase --continue
```

因为这是变基操作，所以可能还有其他被变基了的 commit 会有冲突。此时重复「修改冲突 + add + rebase continue」，也就是上面的操作，直到完全没有冲突为止。

或者使用 Git 的 Rerere 工具，来记录解决冲突的方式，这样就不用手动修改冲突。

# Rerere 工具解决反复出现的冲突

> Rerere = Reuse recorded resolution

因为 rebase 是变基操作，所以在不同 commit 的父 commit 变基之后，可能会出现同一个文件的同一个地方有冲突的情况。

这时， `git rebase --continue` 每次进入下一个 commit 的时候，都会在同一个文件处提示有冲突。为了自动记录并解决冲突，可以使用 Git 的 Rerere 工具。

***

> 冲突解决方案存放地址：`.git/rr-cache`

首先，在全局打开这个工具：

```bash
git config --global rerere.enabled true
```

在出现冲突的时候，先根据需求解决冲突，然后：

```bash
git add .
git commit -m "..." # 只要完成 commit，Rerere 就会记住冲突解决方案
```

这次提交的 commit 相当于解决某个可能重复出现冲突的情况，往后的 commit 就会自动按照这个解决了冲突的 commit 继续进行下去。所以只要继续 rebase 即可：

```bash
git rebase --continue
```

然后，每次 rebase，进入下一个 commit 之后：

```bash
git add .
git rebase --continue
git rebase --skip # 如果 continue 没有反应，就使用 skip
```

不需要每次都提交 commit 了，它会默认在已经解决了冲突的 commit 的基础上自动修改当前的 commit 内容。

所以只要重复上面的步骤，直到 rebase 完成即可。

参考资料：

* [Git 官网：Git 工具 - Rerere](http://git-scm.com/book/zh/v2/Git-%E5%B7%A5%E5%85%B7-Rerere)
* [What is git-rerere and how does it work?](https://stackoverflow.com/questions/49500943/what-is-git-rerere-and-how-does-it-work)

拓展阅读：[Fix conflicts only once with git rerere](https://medium.com/@porteneuve/fix-conflicts-only-once-with-git-rerere-7d116b2cec67)

# 回退版本

## 回退版本 - 基础

取消所有修改，回退到当前版本的初始状态：

```bash
git reset --hard HEAD
```

回退到上一个版本（回退到上一次 commit 纪录）：

```bash
git reset --hard HEAD^
```

> `^` 符号表示上一个版本，如果是 `^^` 就表示上上一个版本

回到前面第 5 个版本：

```bash
git reset --hard HEAD~5
```

通过 `git log` 查看版本标识，然后回退到特定的标识：

```bash
git reset --hard [记录唯一标识]
```

因为回退后，最新的记录会被删除，如果要查找完整 log 记录，需要：

```bash
git reflog
```

## detachced HEAD 分离头指针状态

退回到某个版本，需要使用 `git log` 查看版本的标识，然后再使用 reset 的命令来回退。

如果不如果使用的是 checkout 命令的话，比如有个记录是 9c6b，这个时候使用 `git checkout 9c6b`，就会进入 'detachced HEAD' state（分离头指针状态）

> 'detachced HEAD' state 相当于产生一个独立的环境，不和任何分支挂钩
>
> 这种场景，数据很容易丢失，所以只适合尝试修改变动，且不会有大损失的情况

在 'detachced HEAD' state 进行的操作，如果需要保存起来，就需要创建一个分支，并与其挂钩：

1. 必须在 detachced HEAD 状态下，先 commit 出一个版本记录，才能继续完成保存的操作
2. 获取 commit 记录的标识
	1. `git log` 来获取标识
	2. 也可以直接 `git checkout master` 切换到当前存在的某个分支（不一定是 master），命令行会显示标识
3. `git branch [新branch的名称] [commit记录的标识]` ：根据获取到的 commit 记录的标识，创建新的 branch

# 比较差异

总结：

1. `git diff` 比较工作区和暂存区的差异（如果暂存区没有文件，就比较 HEAD）
2. `git diff --cached` 比较暂存区 和 HEAD
3. `git diff [版本标识]` 比较工作区和版本标识
4. `git diff [版本标识1] [版本标识2]` 比较版本 1 和版本 2
5. 可以加上 `-- [多个文件名]` 来指定需要比较的文件

>`git diff --name-only` 只查看变更的文件名

## git diff

**如果 Stage 中有文件，比较 Stage 和 Working Directory 间所有文件的区别**

显示  Working Directory（工作区/工作目录）和 Stage（暂存区）之间，有哪些文件存在了差异：

```bash
git diff
```

注意，`git diff` ：

- 只显示有差异的文件及其差异处
- 如果 Stage 中没有该文件，而 Working Directory 中有该文件，也不会显示出来
- 如果 Stage 中有该文件，而 Working Directory 中没有该文件，则会显示

***

**如果 Stage 中没有文件，比较的是 Working Directory 和 HEAD 之间的差异** ：

```bash
git diff
```

***

**使用 `git diff` 比较某个（某些）共同文件的修改/区别**

```bash
git diff -- [共同的文件（可以有多个）]
```

>Git 中的 `--` 符号表示前面的是命令，后面的是路径或文件名，如果不会产生歧义的话，也可以省略

## git diff --cached

**比较 HEAD 和 Stage 间所有文件的区别**

当文件被 add 进入了 Stage（暂存区）后，比较 Stage 和 HEAD（也就是上一次 commit）之间的文件，有何区别：

```bash
git diff --cached
```

`--cached` 表示 Stage（暂存区）

***

**比较 HEAD 和 Stage 间某个（某些）文件的区别**

```bash
git diff --cached -- [某个（某些）文件]
```

> Git 中的 `--` 在没有歧义的情况下可以省略

## git diff [版本标识]

**比较 HEAD 和 Working Directory 之间的差异**（如果需要比较文件，使用 `---` + 文件名 即可）：

```bash
git diff HEAD
```

> 注意，如果是 HEAD 没有的文件，是不会显示出来的。也就是说，新增加的文件是不会显示的

***

**比较某个版本和 Working Directory 之间的差异**：

```bash
git diff [版本标识]
```

## git diff [版本标识1] [版本标识2]

可以比较多个版本及分支之间的差异：

```bash
git diff [标识1] [标识2]
```

如果需要指定特定的文件，添加 `-- [文件名]` 即可。

# 修改 commit

## 修改最近一次 commit 的 message（信息）

`git commit --amend` 进入 vim 模式来修改

## 让这次的 commit 归到上一次 commit 中

假设上一次 commit 是在编辑 index 文件，然后 commit 并加上信息 "edit index"。而这一次依旧在编辑 index，我不想新建 commit，而是把这次的 commit 和上次的 "edit index" 放在一起：

```bash
git add . # 将修改放到 Stage
git commit --amend --no-edit # 不生成新的 commit，而是和上次的 commit 融合到一起
```

## 修改之前的 commit 信息

> 为了不影响团队合作，这种修改只能在没有共享该分支的情况下进行

原理：通过 rebase 来操作

- 如果需要修改 commit 的信息，就相当于修改整个 commit，或者说就是生成新的 commit
- 为了生成新的 commit，就需要 rebase（变基）
- 所以要找到：需要修改的 commit 的之前的 commit，也就是该 commit 的父 commit
- 然后根据父 commit 来生成新的 commit（只要是该 commit 之前的 commit 其实就行了）

通过 `git rebase` 加上 `-i` 来进入变基的交互模式：

```bash
git rebase -i [需要修改的commit之前的commit的标识]
```

在交互模式的 vim 模式下，找到需要修改信息的 commit，假设显示的是：

```
pick f1a4224 需要被修改的commit
pick d48eb21 需要被修改的commit之后的commit
```

然后修改为：

```
reword f1a4224 需要被修改的commit
pick d48eb21 需要被修改的commit之后的commit
```

除了 `reword` 之外，还可以写成 `r` 。

然后保存退出，会进入新的交互界面（也是 vim），此时将第一行的内容，修改为新的 commit 信息，再保存退出即可：

```
需要被修改的commit，后面再添加半句新的commit信息
```

## 将连续多个 commit 合成一个 commit

这个也要使用 rebase：

```bash
git rebase -i [需要修改连续多个commit之前的commit的标识]
```

假设得到的是这样（注意，这和上面一样，是根据 commit 时间最早到最晚，也就是最旧到最新的顺序来排序的）：

```markdown
pick 2d02f18 one
pick ba3b159 two
pick c16ef40 three
pick 3b678fb four
```

把这些 commit 合并为一个 commit 的话，相当于生成一个新的 commit，并添加信息。

这里假设要把 2d02f18（信息为 one）、ba3b159（信息为 two）和 c16ef40（信息为 three）合并为一个 commit，并将信息修改为 one to three。

这样的话，需要以最早的  「2d02f18（信息为 one）」为基准，将其之后提交的 commit，也就是「ba3b159（信息为 two）」和「c16ef40（信息为 three）」`squash`（或使用 `s`） 到「2d02f18（信息为 one）」上，而「3b678fb（信息为 four）」这一行保持不动：

```
pick 2d02f18 one
squash ba3b159 two
squash c16ef40 three
pick 3b678fb four
```

然后会自动进入修改 commit 信息的页面，再修改保存即可。

这个时候查看 `git log --oneline` ，会得到：

```
8e690e3 (HEAD -> master) four
b0f93ee one to three
```

也就是说，合并了 3 条 commit 并生成了一个新的 b0f93ee（信息为 one to three），然后将它之后提交的 commit 全部内容不动，但是移动到新的 commit 上。

## 将多个（不连续）commit 合成一个 commit

假设 log 为：

```
0c3a6cd 第四次提交
86517f1 第三次提交
38b611c 第二次提交
4494912 第一次提交
```

现在要把「4494912 第一次提交」和「86517f1 第三次提交」合并为为新的「奇数次提交」为信息的 commit。

因为「4494912 第一次提交」是这个 branch 的第一次提交，所以它没有父 commit，但是依旧可以基于这个 commit 来 rebase：

```bash
git rebase -i 4494912
```

它会显示「4494912 第一次提交」之后的 commit：

```
pick 38b611c 第二次提交
pick 86517f1 第三次提交
pick 0c3a6cd 第四次提交
```

此时，可以自己手动添加最早的那次 commit 的 ID（不需要 message 信息）到第一行：

```
pick 4494912
pick 38b611c 第二次提交
pick 86517f1 第三次提交
pick 0c3a6cd 第四次提交
```

因为需要合并的 commit 信息是不连续的，所以自己手动修改顺序，让需要合并的 commit 连续放在一起，然后参考「合并连续的 commit 的方法」：

```
pick 4494912
s 86517f1 第三次提交
pick 38b611c 第二次提交
pick 0c3a6cd 第四次提交
```

这样的话，86517f1 就会被 `squash`（也就是上面的 `s` 的含义）到 4494912 内。

注意，此时和往常一样，保存退出，它不会继续进交互，而是进入命令行模式。

此时在命令行输入下面这行命令，来继续交互：

```bash
git rebase --continue
```

然后在 vim 中编辑合并后的 commit 信息，最后保存即可。

## 修改 commit 的时间

先使用 `date -R` 查看当前时间及格式，比如：`Sun, 01 Aug 2021 10:35:30 +0800`

如果是修改上次 commit 的时间为 2018 年 12 月 25 日，可以使用：

```bash
git commit --amend --date="Sun, 25 Dec 2018 19:42:09 +0800"
```

将上次的 commit 修改为当前时间：

```bash
git commit --amend --date="$(date -R)" # 可以说这个
git commit --amend --date=`date -R` # 也可以是这个
```

如果修改的不是上一次的 commit：

```bash
git commit --amend --date="$(新的时间)" -C $(需要修改的 commit 的标识)
```

参考资料：

- [修改 git 提交的时间 - xkcoding 的代码成长日记](https://xkcoding.com/2019/01/21/modify-git-commit-timestamp.html)
- [git 修改上次git commit的时间](https://blog.csdn.net/guoyajie1990/article/details/73824732)

# 查看 Log 和仓库内的文件

log 只显示一行：

```bash
git log --oneline
```
只显示最近的 2 条记录（以一行的形式）：

```bash
git log -n2 --oneline
```

图形方式：

```bash
git log --graph
```

**所有分支** 的 log：

```bash
git log --all
```

查看完整记录(使用了 `git reset --hard [唯一标识]` 后会强制退回到该>记录，且删除这之后的 log，所以要使用这种方法来查看完整的 log)：

```bash
git reflog
```

查看 git 仓库内的文件：

```bash
git ls-files
```

# git commit 提交规范

- [Git Commit Message Conventions](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit#heading=h.greljkmo14y0)
- [Commit message 和 Change log 编写指南](https://www.ruanyifeng.com/blog/2016/01/commit_message_change_log.html)
