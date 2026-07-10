# Git 使用指南

## 基础命令

```bash
git init
git clone https://github.com/user/repo.git
git add .
git commit -m "message"
git push origin main
git pull origin main
```

## 分支管理

```bash
git branch feature-1
git checkout feature-1
git checkout -b feature-1
git merge feature-1
git branch -d feature-1
```

## 撤销操作

```bash
git checkout -- file.txt
git reset HEAD file.txt
git reset --soft HEAD~1
```