---
layout: post
title: "jekyll使用markdown创建文章时，table没有边框的问题"
date: 2018-01-24 22:21:38
categories: jekyll
---
直接在`_layout`文件中的`default.html`中的`body`区块中加上下面代码, 就能正常显示

<!-- more -->

```html
<style>
  table{
    border-left:1px solid #000000;border-top:1px solid #000000;
    width: 100%;
    word-wrap:break-word; word-break:break-all;
  }
  table th{
  text-align:center;
  }
  table th,td{
    border-right:1px solid #000000;border-bottom:1px solid #000000;
  }
</style>
```

