## 寻找并删除 Git 记录中的大文件

### 首先通过 rev-list 来找到仓库记录中的大文件：

```shell
    git rev-list --objects --all | grep "$(git verify-pack -v .git/objects/pack/*.idx | sort -k 3 -n | tail -5 | awk '{print$1}')"
```

### 然后通过 filter-branch 来重写这些大文件涉及到的所有提交（重写历史记录）

```shell
git filter-branch -f --prune-empty --index-filter 'git rm -rf --cached --ignore-unmatch your-file-name' --tag-name-filter cat -- --all
```

### 再删除缓存的对象

```shell
git for-each-ref --format='delete %(refname)' refs/original | git update-ref --stdin
git reflog expire --expire=now --all
git gc --prune=now
```

### 强制推送改动到远端

```shell
git push origin --force --all
```
