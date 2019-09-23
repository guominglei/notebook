# git 

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

