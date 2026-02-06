# ZZJ的个人主页项目
## 本项目Fork自leleo886开发的[个人主页项目](https://github.com/leleo886/leleo-home-page)

基于Vue + Vuetify构建的现代化个人主页模板，支持响应式设计和自定义配置。

## 目录
- [项目简介](#项目简介)
- [功能特性](#功能特性)  
- [技术栈](#技术栈)
- [快速开始](#快速开始)
  - [本地运行](#本地运行)
  - [在线部署](#在线部署)

## 项目简介

这是一个现代化的个人主页项目，用于展示个人信息、技能水平、项目作品等。采用响应式设计，完美适配桌面端、平板和移动设备。

## 功能特性

- **响应式设计**：自动适配不同屏幕尺寸的设备
- **个人信息展示**：支持头像、个性标签、个人简介、技能雷达图等
- **项目作品展示**：可展示多个项目，包含项目描述、技术栈和外部链接
- **动态背景**：支持静态/动态壁纸，PC端和移动端分别配置
- **音乐播放**：集成音乐播放器功能
- **主题定制**：支持自定义颜色主题和视觉效果
- **一键部署**：支持Vercel、CloudFlare Pages等多种部署方式

## 技术栈

- **前端框架**：Vue 3
- **UI组件库**：Vuetify 3
- **构建工具**：Vite
- **图表库**：Chart.js
- **样式预处理器**：Less
- **图标库**：Material Design Icons

## 快速开始

### 本地运行

1. 克隆项目到本地：

```bash
git clone <你的项目仓库地址>
```

2. 进入项目目录：

```bash
cd <项目目录名>
```

3. 安装依赖：

```bash
npm install
```

4. 启动开发服务器：

```bash
npm run dev
```

5. 打开浏览器访问 `http://localhost:5173`

### 在线部署

#### Vercel部署

1. Fork本项目到你的GitHub账号
2. 访问 [Vercel](https://vercel.com) 并登录
3. 点击"New Project"，选择你fork的仓库
4. 配置项目名称，点击"Deploy"
5. 等待部署完成即可访问

#### CloudFlare Pages部署

1. Fork本项目到你的GitHub账号
2. 登录CloudFlare控制台，进入"Workers & Pages"
3. 点击"Create application" → "Pages"
4. 连接GitHub账号，选择项目仓库
5. 构建设置：框架预设选择"Vue"
6. 点击"Save and Deploy"