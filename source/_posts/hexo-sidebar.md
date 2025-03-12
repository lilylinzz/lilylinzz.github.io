---
title: hexo_sidebar
date: 2025-03-12 14:28:19
description: Hexo_NexT 增加側欄功能
categories: 個人網頁
tags: hexo
---

## 加入頭像

`~/_config.next.yml` 調整設定檔中 avatar url

```bash
# Sidebar Avatar
avatar:
  url: /images/avatar.png
  rounded: true #改圓型
  rotated: false #旋轉
```

在 `hexo_root/source/images/avatar.png` 設置自己的圖片

---

## 加入 social link - Email & Linkedin

`~/_config.next.yml`  設定檔中，Github、Email移除註解，Linkedin要自己新增

```bash
social:
  GitHub: https://lilylinzz.github.io/ || fab fa-github
  E-Mail: mailto:lingaadin@gmail.com || fa fa-envelope
  Linkedin: https://www.linkedin.com/in/lilylin-94a835229 || fab fa-linkedin-in
```

social_icons > icons_only 改為true，只顯示icon

```bash
social_icons:
  enable: true #預設
  icons_only: true
  transition: false #預設
```

---

## 添加分類、標籤功能

`~/_config.next.yml` 設定檔中找到 menu 的設置，取消註解 tag、categories
(如果要添加"about_me"頁面，也是相同作法)
```
menu:
  home: / || fa fa-home
  #about: /about/ || fa fa-user
  tags: /tags/ || fa fa-tags
  categories: /categories/ || fa fa-th
  archives: /archives/ || fa fa-archive
  #schedule: /schedule/ || fa fa-calendar
  #sitemap: /sitemap.xml || fa fa-sitemap
  #commonweal: /404/ || fa fa-heartbeat
```
建立page，建立後會在 source 目錄下長出 tags、categories 資料夾，底下有 [index.md](http://index.md/) 檔案

```bash
hexo new page "tags"
hexo new page "categories"
```

編輯 [index.md](http://index.md) 加入對應的 type

```bash
---
title: tags
date: 2025-03-10 17:53:58
type: tags
---

---
title: categories
date: 2025-03-12 13:51:42
type: categories
---
```
後續在每篇文章頂部，輸入標籤及分類格式如下
```
title: 
date: 
tags: 
      - tagA
      - tagB
categories: 
- [AA分類, aa子分類]
```

---

## 搜尋功能

跟目錄安裝

```bash
npm install hexo-generator-searchdb
```

`~/_config.yml` Hexo設定檔加入

```bash
# Search Service
search:
  path: search.xml
  field: post
  content: true
  format: html
```

`~/_config.next.yml` NexT設定檔 local_search > enable 改為 true
```
local_search:
  enable: true
  # Show top n results per article, show all results by setting to -1
  top_n_per_article: 1
  # Unescape html strings to the readable one.
  unescape: false
  # Preload the search data when the page loads.
  preload: false
```