# GitHub

## SSH

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

