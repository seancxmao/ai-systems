# Git

## GitHub

### SSH

生成公钥和私钥对：

```
> ssh-keygen -t ed25519 -C "seancxmao@gmail.com"

> ls ~/.ssh
id_ed25519     id_ed25519.pub
```

把公钥放到GitHub上：

```
cat ~/.ssh/id_ed25519.pub
```

测试：

```
ssh -T git@github.com
```

* `-T`：禁用伪终端（pseudo-terminal）分配（Disable pseudo-terminal allocation）
* 为什么 GitHub 要用 -T：当你测试 SSH 密钥是否配置正确时，GitHub 并不会给你一个交互式 Shell（终端会话），而只是验证你的身份。

## 常用命令

以「Fork + PR」方式参与日常开发，90% 时间其实都在用下面这些命令。

| 命令                              | vLLM场景                    | 本质（直觉）            |
| ------------------------------- | ------------------------- | ----------------- |
| `git clone`                     | 第一次把 vLLM 拉到本地            | 复制整个仓库            |
| `git remote -v`                 | 查看 origin/upstream 是否配置正确 | 看仓库之间的连接关系        |
| `git fetch upstream`            | 查看官方最近有没有新提交              | 把远程信息抄到本地，但不动你的代码 |
| `git checkout main`             | 切回主分支同步代码                 | 切换工作区             |
| `git checkout -b fix-metal-bug` | 开发一个 Metal Bug Fix        | 从当前提交切出实验分支       |
| `git rebase upstream/main`      | 官方 main 更新了几百个 commit     | 把你的提交搬到最新主线后面     |
| `git merge xxx`                 | 偶尔合并别的分支                  | 把两条历史接起来          |
| `git status`                    | 开发过程中最常用                  | 查看工作区状态           |
| `git diff`                      | 看自己改了什么                   | 查看代码差异            |
| `git add file.py`               | 修改完成准备提交                  | 把改动放进暂存区          |
| `git commit -m "..."`           | 保存一个逻辑变更                  | 创建历史快照            |
| `git log --oneline`             | 查看提交历史                    | 看时间线              |
| `git push origin fix-metal-bug` | 推送分支准备 PR                 | 上传提交到 GitHub      |
| `git pull`                      | 自己的小仓库同步                  | fetch + merge     |
| `git stash`                     | 临时切任务                     | 把未提交修改塞进抽屉        |
| `git stash pop`                 | 回来继续开发                    | 从抽屉拿出来            |
| `git reset --soft HEAD~1`       | commit 写错了                | 撤销快照，保留改动         |
| `git restore file.py`           | 改坏一个文件                    | 恢复到上次状态           |

如果按频率排序，最核心的是：

```bash
git status
git diff

git fetch upstream
git rebase upstream/main

git checkout -b feature

git add .
git commit -m "..."

git push origin feature
```

可以把它们理解成：

```
status    看现场
diff      看改动

fetch     打听官方最新消息
rebase    跟上主线

checkout  开新任务
add       装箱
commit    存档
push      上传
```

对于日常开发，这套心智模型基本够用了。真正需要深入理解的只有两个概念：

```text
工作区 (Working Tree)
    ↓ add
暂存区 (Index)
    ↓ commit
提交历史 (Commit Graph)
```

几乎所有 Git 命令，本质上都在这三层之间搬东西。

## 维护生产系统

在公司里，除了日常的 `fetch/rebase/commit/push`，最常见的是下面这些。

| 场景          | 命令                         | 本质             |
| ----------- | -------------------------- | -------------- |
| 给线上版本打补丁    | `git cherry-pick`          | 把某个提交搬到另一条分支   |
| 发布版本        | `git tag v1.2.3`           | 给某个提交贴标签       |
| 紧急修复线上问题    | `git checkout release-x.y` | 切到发布分支修 Bug    |
| 回滚事故        | `git revert <commit>`      | 用新提交撤销旧提交      |
| 查看谁改坏了代码    | `git blame file.py`        | 查责任提交          |
| 定位 Bug 来源   | `git bisect`               | 二分查找引入 Bug 的提交 |
| 长期维护多个版本    | branch 管理                  | 一条主线多个发行版      |
| 给客户交付 Patch | `git format-patch`         | 导出提交为补丁文件      |
| 应用外部 Patch  | `git apply` / `git am`     | 导入补丁           |

### 1. Cherry-pick

假设：

```text
main
 ├─A─B─C─D
release/1.0
 └─A─B
```

线上 1.0 出 Bug。

你在 main 修好：

```text
main
 ├─A─B─C─D─E
```

只想把 E 搬过去：

```bash
git checkout release/1.0
git cherry-pick E
```

结果：

```
release/1.0
 └─A─B─E'
```

直觉：

```
复制一个提交
而不是复制整个分支
```

### 2. Tag

发布：

```bash
git tag v0.9.0
git push origin v0.9.0
```

以后：

```bash
git checkout v0.9.0
```

就能回到当时的代码。

直觉：

```
给某个历史快照起名字
```

### 3. Revert

某提交导致服务崩溃：

```text
A-B-C-D-E
      ↑
      坏提交
```

不要：

```bash
git reset
```

因为别人已经同步了。

应该：

```bash
git revert D
```

变成：

```text
A-B-C-D-E-F
        ↑
    撤销D
```

直觉：

```
历史不能删
只能增加一个反向操作
```

### 4. Blame（查锅）

```bash
git blame scheduler.py
```

看到：

```text
abc123 Sean
def456 Alice
```

知道是谁改的。

直觉：

```
代码考古
```

### 5. Bisect（神兵利器）

现象：

```
v1.0 正常
v1.8 崩了
```

中间：

```
1000 个 commit
```

不用一个个看：

```bash
git bisect start
git bisect bad
git bisect good v1.0
```

Git 自动二分。

```
1000 commit
↓
约10次测试
↓
定位元凶
```

直觉：

```
Git 帮你做二分查找
```

### 6. Format-patch（传统大厂常见）

导出：

```bash
git format-patch -1 HEAD
```

得到：

```text
0001-fix-metal.patch
```

别人：

```bash
git am 0001-fix-metal.patch
```

应用。

Linux Kernel 现在仍大量使用这种模式。

直觉：

```text
提交 = 邮件附件
```

## 常见的实际问题

### 问题1：线上版本不能升级

```text
main
  ↑ 每天变化
prod-2026.06
  ↑ 正在服务用户
```

解决：

```text
release branch
+ cherry-pick
```

### 问题2：某优化导致性能下降

解决：

```bash
git bisect
```

找到引入问题的 commit。

### 问题3：客户要求恢复旧行为

解决：

```bash
git revert
```

### 问题4：需要精确复现线上环境

解决：

```bash
git checkout v2.3.7
```

(tag)

## References

* https://git-scm.com/
* https://git-scm.com/book/en/v2
