# git 
## 全局设置gitignore
    python相关的配置为： 
        https://github.com/github/gitignore/blob/master/Python.gitignore
    
    git config --global core.excludesfile ~/.gitignore 

## submodule
git 工程引用外部工程使用git submodule

### 初始化:
    git submodule add xx.git(git地址) yy/y(项目中的地址) 
    在项目中会有一个.gitmudule 文件用于记录当前项目中引用子模块的情况。
    这时候会把引用的外部git项目clone到指定的项目路径中。

### clone 一个带子模块的项目：
    git clone xx.git #克隆基础项目
    git submodule #查看子模块情况
    git submodule init #初始化子模块
    git submodule update # clone 子模块文件

    简化步骤:
    git clone - - recursive xx.git  比克隆基础项目多了一个参数  - - recursive

### 文件更新:
    进入到子模块的本地目录中
    git checkout master
    git pull

### 批量更新:
    git submodule foreach git pull

### 删除子模块:
    删除git cache 和物理文件夹
        git rm -r - -cached xx
    删除.gitmodules 里边的对应条目
    删除.git/config 里边对应的条目
    提交更改

## reset , revert 区别
    reset 假如有5个版本。 现在要从版本5 reset 到版本2 那么版本3，4，5这三个版本数据
        在reset命令操作后会丢失。
    revert 是利用把当前版本利用之前的某个版本给替换。版本提交的历史数据都还在。

## rebase 命令
    1、本地分支更新主分支最新代码。
        1、更新主分支代码到本地仓库。
            git fetch xx
        2、rebase操作
            git rebase -i xx
            解决冲突后。git rebase --continue 接着处理下一个合并
            git会把commit进行排序。自己分支的commit会排在前边。

    2、合并commit
        git rebase -i comitid  把commit合并成若干个。

下游分支更新上游分支内容的时候使用 rebase。如Dev更新master上的最新代码
上游分支合并下游分支内容的时候使用 merge。 如master合并dev上的代码。 

## 合并提交
    git commit -a --amend






