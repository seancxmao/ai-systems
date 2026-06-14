# GitHub Pages + MkDocs Material

## 背景

GitHub Pages + MkDocs Material是当前技术圈非常主流、非常成熟的一种方案。

用GitHub Pages和MkDocs Material搭建网站进行个人知识管理：

* 学习日志（Learning Notes）
* 项目文档（Project Docs）
* 知识库（Knowledge Base）

## 首次搭建

### 1. 创建GitHub Repository, 并Clone到本地


### 2. 创建Python虚拟环境

```
uv venv --python 3.12
source .venv/bin/activate
```

### 3. 安装MkDocs Material

```
pip install mkdocs-material
mkdocs --version
```

### 4. 初始化站点

```
cd ai-systems
mkdocs new .
```

### 5. 配置 MkDocs

mkdocs.yml

```
site_name: AI Systems

theme:
  name: material

plugins:
  - search
```

### 6. 创建第一篇文章

```
mkdir docs/ai-infra
vi docs/ai-infra/vllm.md
```

### 7. 配置导航

```
site_name: AI Systems

theme:
  name: material

plugins:
  - search

nav:
  - Home: index.md

  - AI Infra:
      - vLLM: ai-infra/vllm.md
```

### 8. 本地预览

```
mkdocs serve
```

```
http://127.0.0.1:8000
```

### 9. 提交到 GitHub

```
git status
git add .
git commit -m "Initial MkDocs site"
git push origin main
```

从2021年开始，GitHub已经彻底关闭了HTTPS密码认证，推荐用SSH。

### 10. 手动构建静态网站

```
mkdocs build
```

### 11. 手动发布Github Pages

```
mkdocs gh-deploy
```

### 12. 开启 Github Pages

Repository -> Settings -> Pages


### 13. 访问网站

https://seancxmao.github.io/ai-systems/

## 后续更新

### 1. 新增文章

```
vim docs/ai-infra/sglang.md
```

### 2. 本地预览

```
mkdocs serve
```

### 3. 提交

```
git add .
git commit -m "add sglang notes"
git push
```

### 4. 重新发布

```
mkdocs gh-deploy
```

## 总结

