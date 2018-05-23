# 通过Git解决fork之后仓库更新

### 1、查看远程状态

```
git remote -v //查看远程状态
```


### 2、添加远程仓库

```
git remote add center[自行命名] git@github.com:coom163/chat-app.git[仓库地址]
```

### 3、再次确认远程状态

```
git remote -v //再次查看确认状态是否配置成功
```


### 4、更新代码

将center仓库中的代码更新到分支branch1上。

```
git fetch center master:branch1
```

### 5、合并分支

将center/master的代码合并到origin/master上

```
git merge center/master
```

**确保此时的分支在master上，若不是则可以通过执行以下代码修改分支**

```
git checkout master
```

### 6、提交到远程仓库

更新到fork的仓库中

```
git push origin master
```

