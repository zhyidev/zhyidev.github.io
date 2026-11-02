# zhyi 的个人主页

Java 后端 / AI 应用开发方向的个人站点：首页简介、简历、项目、笔记，都在同一个仓库里。

线上地址：<https://www.10010506.xyz>

## 这个仓库里有什么

静态站点，Jekyll 构建，跑在 al-folio 提供的 `al_folio_core` 主题上。**仓库里只有内容，没有布局**——`_layouts`、`_includes`、`_sass`、`assets/css|js` 都在主题 gem 里，所以改样式要去改主题，改内容只动下面这些目录：

| 路径 | 放什么 |
|---|---|
| `_config.yml` | 站点名、描述、URL、导航与功能开关 |
| `_pages/about.md` | 首页正文 |
| `_data/cv.yml` | `/cv/` 简历页的数据 |
| `_data/socials.yml` | 邮箱、GitHub 等社交链接 |
| `_projects/` | 项目卡片，一个文件一个项目 |
| `_posts/` | 笔记，文件名 `YYYY-MM-DD-title.md` |
| `_news/` | 首页「动态」时间线条目 |
| `_bibliography/papers.bib` | 论文条目（暂时为空） |
| `assets/img/` | 图片，头像为 `prof_pic.jpg` |

### 加论文条目

`_bibliography/papers.bib` 目前是空的，只留了一个 `@string` 定义。加条目时**不要写 `%` 注释**——bibtex-ruby 不认 `%`，会把注释里的内容当条目解析，直接报 `Failed to parse BibTeX`。常用字段：`abbr`（期刊缩写）、`selected`（是否上首页精选）、`preview`（预览图）、`bibtex_show`（是否显示 BibTeX 按钮）。

## 本地预览

```bash
bundle install
bundle exec jekyll serve --livereload
```

打开 <http://localhost:4000>。想省掉本地 Ruby 环境，也可以用容器：

```bash
docker compose up
```

同样是 4000 端口。

## 发布

推 `main` 分支，GitHub Actions 里的 `deploy.yml` 会构建并发布到 `gh-pages`，由 GitHub Pages 提供服务，域名走 `CNAME`（`www.10010506.xyz`）。

## 说明

主题来自 [al-folio](https://github.com/alshedivat/al-folio)，按 MIT 协议使用。站点内容是自己的，模板自带的演示文章、示例论文和示例课程已全部清掉。
