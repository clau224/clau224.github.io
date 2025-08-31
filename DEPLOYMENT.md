# 部署到 GitHub Pages 指南

## 当前状态

你的 Hugo 博客已经配置好了自动部署到 GitHub Pages 的设置。

## 部署步骤

### 1. 确保 GitHub 仓库设置正确

1. 进入你的 GitHub 仓库 `clau224.github.io`
2. 进入 `Settings` → `Pages`
3. 在 `Source` 部分选择 `GitHub Actions`
4. 确保仓库是公开的（public）

### 2. 推送代码触发自动部署

```bash
# 确保你在项目根目录
cd /Users/gvliew/Documents/github-repo/clau224.github.io

# 添加所有文件
git add .

# 提交更改
git commit -m "Setup GitHub Actions deployment"

# 推送到 GitHub
git push origin main
```

### 3. 检查部署状态

1. 在 GitHub 仓库页面，点击 `Actions` 标签
2. 查看最新的工作流运行状态
3. 等待构建和部署完成

### 4. 访问你的博客

部署成功后，你的博客将在以下地址可用：
- https://clau224.github.io

## 本地开发

### 启动本地服务器

```bash
cd myblog
hugo server -D
```

### 构建静态文件

```bash
cd myblog
hugo
```

## 文件结构说明

```
clau224.github.io/
├── .github/workflows/deploy.yml  # GitHub Actions 部署配置
├── .gitignore                    # Git 忽略文件
├── myblog/                       # Hugo 博客项目
│   ├── content/                  # 博客内容
│   ├── themes/                   # 主题文件
│   ├── hugo.toml                 # Hugo 配置
│   └── public/                   # 生成的静态文件（自动生成）
└── README.md                     # 项目说明
```

## 注意事项

1. **不要手动提交 `public/` 目录**：这个目录会被自动生成和部署
2. **每次推送代码都会自动部署**：确保代码质量
3. **部署需要几分钟时间**：请耐心等待
4. **如果部署失败**：检查 GitHub Actions 日志中的错误信息

## 故障排除

### 常见问题

1. **部署失败**：检查 Hugo 配置语法是否正确
2. **页面显示 404**：确保 `baseURL` 设置正确
3. **样式丢失**：检查主题是否正确安装

### 获取帮助

如果遇到问题，可以：
1. 查看 GitHub Actions 日志
2. 检查 Hugo 官方文档
3. 在 GitHub Issues 中提问
