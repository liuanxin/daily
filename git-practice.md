### 分支说明

+ master: 主要用于正式发布. 每增加一个版本都要打上一个 tag 并给定版本
+ develop: 主要用于日常开发. 每次都推送到这个分支上来

**线上(生产)发布代码时基于 tag 进行发布, 每向 master 合并一次代码就需要打一个 tag**


-----


### 开发实践

1. 基于 gogs 建立项目. 将相关人员加进项目并设置项目管理员. ==> 管理员操作

2. 将项目 clone 到本地, 此时本地会有一个 master 分支 ==> 开发人员和仓库负责人都要操作这一步
```
git clone http://url:port/user/project.git
```

3. 基于当前本地分支(master)创建本地的 develop 分支 ==> 仓库负责人操作
```
git branch develop <==> git checkout -b develop master # <==> 表示左右两边的意思等同
```

4. 将本地 develop 分支推到远程(然后远程会有两个分支: master 和 develop) ==> 仓库负责人操作
```
git push -u origin develop
```

5. 从远程 develop 分支拉取代码 ==> 开发人员操作
```
git checkout develop
git pull
```

6. 本地开发 & 提交代码 ==> 开发人员操作
```
git add . 
git commit -m "xxx"
```

7. 向远程的 develop 分支推送代码 ==> 开发人员操作
```
git push
```

8. 准备发布生产了 --> 从远程 develop 分支拉取代码 ==> 仓库负责人操作
```
git checkout develop
git pull
```

9. 切换回本地 master 分支, 并将本地 develop 分支合并到本地 master 分支 ==> 仓库负责人操作
```
git checkout master
git merge --no-ff develop
```

10. 基于本地 master 分支打一个 tag 并给一个说明 ==> 仓库负责人操作
```
git tag -a 1.2.3 -m "更新内容" master
```

11. 基于本地 master 分支向远程 master 分支推送, 并将 tag 推送至远程 ==> 仓库负责人操作
```
git push
git push --tags
```

其中, 第 1 2 3 4 对一个项目而言只需要做一次(第一步由管理员操作, 剩下三步由项目负责人操作)  
开发人员平时的操作是 5 6 7 这几个步骤, 与 svn 相比, git 多了一个向远程分支推送的步骤  
第 8 9 10 11 这几个步骤是每次需要发布到线上(生产)环境时需要做的(由项目负责人操作)


-----


因为每个线上环境的代码都基于 tag 发布的, 而 tag 与当前项目最新的 master 是一致的, 当线上环境遇到 bug 需要紧急修复时需要这样处理

1. 将代码切换到当前线上环境对应的 tag
```
git checkout master
```

2. 修复 bug 并提交
```
git add .
git commit -m "修复生产环境中的 xxx bug"
```

3. 基于本地 master 分支向本地 develop 合并代码并推送到远程的 develop 分支
```
git checkout develop
git merge --no-ff master
git push
```

4. 基于本地 master 分支打一个 tag 并给一个说明
```
git checkout master
git tag -a 1.2.4 -m "修复生产环境中的 xxx bug, 递增版本号" master
```

5. 基于本地 master 分支向远程 master 分支推送, 并将 tag 推送至远程
```
git push
git push --tags
```
