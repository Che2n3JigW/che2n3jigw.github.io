---
# 引导页面布局
layout: splash
# 标题
title: "che2n3jigw"
# 摘录
excerpt: "记录与分享互联网技术的学习与实践."
classes: wide
# 自我介绍
intro: 
  - excerpt: '你好，我是 **che2n3jigw**，一名Android应用开发者。'
  - excerpt: '在这里我记录编程实践、开发环境配置，以及各种互联网技术的探索。'
  - excerpt: '写博客是为了学习、整理和分享，也希望这些内容能对你有所帮助。'
# 设置header背景
header:
  overlay_image: /assets/images/index/header.jpg

feature_row:
  - image_path: assets/images/index/cover/env.jpg
    title: "环境搭建"
    excerpt: "开发环境搭建"
    url: "/categories/Env/"
    btn_label: "详情"
    btn_class: "btn--primary"

  - image_path: assets/images/index/cover/nat_t.jpg
    title: "飞牛"
    excerpt: "飞牛使用心得"
    url: "/categories/fn/"
    btn_label: "详情"
    btn_class: "btn--primary"

  - image_path: assets/images/index/cover/android.jpg
    title: "Android"
    excerpt: "记录 Android 应用开发中的实践经验、踩坑总结和优化技巧"
    url: "/categories/Android/"
    btn_label: "详情"
    btn_class: "btn--primary"
  # - image_path: assets/images/default_posts_cover.jpg
  #   title: "工具"
  #   excerpt: "分享开发过程常用的工具与技巧"
---

{% include feature_row id="intro" type="center" %}

{% include feature_row %}

<!-- {% include feature_row id="feature_row2" type="left" %}

{% include feature_row id="feature_row3" type="right" %}

{% include feature_row id="feature_row4" type="center" %} -->