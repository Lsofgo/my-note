### 推送本地到远程

```shell
git push origin master
```

### 同步远程到本地

1. 查看远程分支

   ```shell
   git remote -v
   ```

2. 从远程获取最新版本到本地

   ```shell
   git fetch origin master:temp
   ```

3. 比较本地仓库与下载的temp分支

   ```shell
   git diff temp
   ```

4. 合并temp分支到本地的master分支

   ```shell
   git merge temp
   ```

5. 删除temp分支

   ```shell
   git branch -d temp
   ```

   

