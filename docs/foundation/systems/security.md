# Security

## Mental Model

Authentication = 你是谁

Authorization = 你能做什么

Symmetric Key = 快速传输数据

Asymmetric Key = 身份认证 + 密钥交换

Certificate = 可信第三方签发的身份声明

TLS = Authentication + Key Exchange + Encrypted Communication

HTTPS = HTTP + TLS

mTLS = Mutual TLS. Kubernetes组件之间经常使用. 普通TLS：Server证明自己；mTLS：Server证明自己 + Client证明自己。

## Terminology

身份（Identity）

认证（Authentication）

授权（Authorization）

信任（Trust）

加密（Encryption）

SSH（Secure Shell）是一种安全远程通信协议。OpenSSH是SSH协议最主流的开源实现。

