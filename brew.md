# Homebrew
    
    Mac下安装包管理工具
    homebrew 安装目录位于 /usr/local
    用homebrew 安装的包 位于 /usr/local/Cellar。 

## 安装
    
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"

## 卸载
    /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/uninstall)"

## 常用命令
    
    搜索包
    brew search mysql

    安装包
    brew install mysql

    查看包信息，比如目前的版本，依赖，安装后注意事项等
    brew info mysql

    卸载包
    brew uninstall wget

    显示已安装的包
    brew list

    查看brew的帮助
    brew –help

    更新， 这会更新 Homebrew 自己
    brew update

    检查过时（是否有新版本），这会列出所有安装的包里，哪些可以升级
    brew outdated
    brew outdated mysql

    升级所有可以升级的软件们
    brew upgrade
    brew upgrade mysql

    清理不需要的版本极其安装包缓存
    brew cleanup
    brew cleanup mysql

