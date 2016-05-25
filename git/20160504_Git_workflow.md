## Git工作流

版本管理中分支模型的目的是什么？

好的**分支模型**需要基于什么考虑？

一个commit是管理文件的历史记录，那分支模型可以看做项目的版本历史记录，也称之为**里程碑**。

因此好的**分支模型**需要考虑以下：

1. 项目的生命周期：开发需求 ---> 解决bug ---> 不断趋向稳定 ---> 预发布 ---> 发布 ---> 突发问题修复；
2. 分支合并的可操作性：自动化或人工审核半自动化；

## 常用策略

好的分支策略可以使得版本库的演进保持简洁，主干清晰，各个分支各司其职、井井有条；

常用策略为：主分支master + 日常开发分支Develop + 三个临时分支（功能分支feature+预发布分支pre-release+修补分支hotfix）

![Git分支模型](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/7cc829d3gw1en76ivwj9yj20vy16cdmb.jpg)

## 分支模型可操作性

考虑下实际项目情况（脑洞大开）

1. 开发人员提交到开发分支；  --->  参与者：开发人员
2. 输出开发分支版本； ---> 参与者：配置管理员
3. 开发分支版本验证； ---> 参与者：QT
4. 临时特性需求：开发人员提交到功能分支 --> 参与者： 开发人员
5. 输出功能分支版本，功能分支版本验证 --->  参与者：配置管理员，QT
6. 需求基本开发完毕，趋向稳定，自动推出预发布分支 ---> 参与者：配置管理员
7. 预发布分支修复问题，趋向稳定，输出预发布分支的版本，测试验证 ---> 参与者：开发人员，配置管理员，QT
8. 预发布分支稳定性确认评审，发布到主分支（里程碑），输出版本最终验证 ---> 参与者：配置管理员，QT
9. 版本发布后紧急问题修复到hotfix，输出hotfix版本验证 ---> 参与者：开发人员，配置管理员，QT
10. 推送到主分支输出版本验证 ---> 参与者：配置管理员，QT

模型简单了，可操作过于复杂，如：

1. 开发人员需要在不同时期往不同分支提交：理解成本、沟通成本、管理成本过高；
2. 配置管理员要配合输出不同分支版本：五个分支在不同时期都要考虑输出版本，如果加上版本的不同渠道，版本数量将翻倍增长；
3. 配置管理员要配合做半自动分支合并（也有审核人员承担该部分工作）：临时分支往开发分支（可能有冲突）和主分支，开发分支往预发布分支，预发布分支往主分支；
4. QT需要在不同版本上测试，提交的bug需要区分不同的版本，及不同版本问题验证，及问题验证后的可持续确认性（即该分支修复后其他分支确认已修复）

因此需要将操作精简简化：

1. 分支模型文档化，并组织相关培训，让开发人员理解分支模型并达成一致；
2. 版本输出：只输出develop和master版本，其他由开发自行输出（或通过工具）输出验证；
3. 分支合并：为避免合并混乱，同时方便解决冲突，只开发部分模块负责人有合并权限；
4. QT提交问题不区分分支版本，开发分支验证问题只做标注（协同开发），关闭问题需在master分支版本；

## GitFlow工具

![Git flow模型](https://raw.githubusercontent.com/KellyZ/ItLoveBlog/master/images/git-workflow-release-cycle-4maintenance.png)

1. 安装git-flow

        sudo apt-get install git-flow
    
2. 项目初始化

        git flow init
        
        No branches exist yet. Base branches must be created now.
        Branch name for production releases: [master]
        Branch name for "next release" development: [develop]
        
        How to name your supporting branch prefixes?
        Feature branches? [feature/]
        Release branches? [release/]
        Hotfix branches? [hotfix/]
        Support branches? [support/]
        Version tag prefix? []

3. 查看初始化结果

        > git branch -a
            * develop
              master
              
        > cat .git/config
            [core]
                    repositoryformatversion = 0
                    filemode = true
                    bare = false
                    logallrefupdates = true
            [gitflow "branch"]
                    master = master
                    develop = develop
            [gitflow "prefix"]
                    feature = feature/
                    release = release/
                    hotfix = hotfix/
                    support = support/
                    versiontag =
        
4. 开始需求开发,如登录,会创建feature/login分支

        git flow feature start login
        
        > git branch
              develop
            * feature/login
              master

5. 登录需求开发完成，feature/login分支合并到develop分支，并删除feature/login分支

        git flow feature finish login
        
        > git branch
            * develop
              master

6. 需求开发完，开始版本预发布：创建了release/v1.0分支

        git flow release start v1.0
        
7. 预发布测试验证，正式对外发布：合并release/v1.0分支到develop和master，并删除release/v1.0分支，同时创建v1.0标签
        
        git flow release finish v1.0
        
8. 修复版本发布后的紧急bug：创建hotfix/jira-3312分支

        git flow hotfix start jira-3312
        
9. 验证修复后发布：合并hotfix/jira-3312分支到develop和master，并删除hotfix/jira-3312分支，同时创建jira-3312标签

        git flow hotfix finish jira-3312


## 多项目批量管理（如Android）



## 参考

1. http://nvie.com/posts/a-successful-git-branching-model/
2. http://www.ruanyifeng.com/blog/2012/07/git.html
3. http://blog.jobbole.com/76857/
4. http://blog.jobbole.com/76867/