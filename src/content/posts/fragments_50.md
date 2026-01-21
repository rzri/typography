---
title: 为Hexo Oranges主题添加自定义导航链接
pubDate: 2025-11-23
categories: ['笔记']
description: '通过修改_config.oranges.yml配置文件，在Oranges主题导航栏中快速添加外部链接，无需新建EJS模板。'
slug: hexo-oranges-add-nav-link
---

Oranges 主题添加导航链接

只需修改主题配置文件，不用新建 EJS 文件。
步骤

1. 打开 _config.oranges.yml
2. 在 navbar 列表里加一项：

yaml
name: 开往
enable: true
path: https://
key: kaiwang

3. 保存后运行：

```bash
hexo clean && hexo g && hexo s
```

完成！导航栏会自动显示「开往」并跳转。

仅普通链接才不用 .ejs；需要图标或复杂结构时才要创建。