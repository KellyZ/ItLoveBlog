这一节重点讲.git/config文件，也就是常用命令git config;

## 配置文件

首先配置文件分为三种：

1. 系统全局配置：在git安装目录下etc/gitconfig，对应git config --global;
2. 用户家目录下配置：~/.gitconfig, git config --system;
3. 仓库目录下配置：.git/config, git config --local或git config;

配置项优先级：仓库配置 > 用户家目录配置 > 系统全局配置；

## 配置分类

Git的配置是以键值对的形式保持，其中key/name值常用的有如下几类：

1. core
    * core.filemode : 是否记录文件权限的修改，默认为true；
    * core.autocrlf : 配置换行符的处理，在仓库中加入.gitattributes是更好的处理方式，参考后面的.gitattributes配置。
    * core.editor : 配置git commit/tag的编辑器；
2. alias.* : 可以配置的别名；
3. branch
    * branch.autoSetupMerge
    * branch.autoSetupRebase
4. color
    * color.branch=always/false/never/true/auto: git branch的输出是否带颜色；
    * color.diff=auto/true/false: git log/show时的颜色输出；
5. user
    * user.name
    * user.email


## .gitattribute参考

    # Set the default behavior, in case people don't have     core.autocrlf set.
    * text=auto

    # Explicitly declare text files you want to always be normalized and converted
    # to native line endings on checkout.
    *.c text
    *.h text

    # Declare files that will always have CRLF line endings on checkout.
    *.sln text eol=crlf

    # Denote all files that are truly binary and should not be modified.
    *.png binary
    *.jpg binary


## 参考文档

1. https://help.github.com/articles/dealing-with-line-endings/
