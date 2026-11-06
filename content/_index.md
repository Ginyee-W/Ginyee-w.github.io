---
title: 'Home'
date: 2023-10-24
type: landing

sections:
  # ② 原来的简介 block 保持不动
  - block: resume-biography
    content:
      # The user's folder name in content/authors/
      username: admin
    design:
      spacing:
        padding: [0, 0, 0, 0]
      biography:
        style: 'text-align: justify; font-size: 0.8em;'
      # Avatar customization
      avatar:
        image : 'content/authors/admin/avatar.jpg'
        size: medium
        shape: circle
    # 2. 最新照片：横向可滚动
  - block: markdown
    id: latest-photos
    content:
      title: ''
      text: |-
        {{< photo-strip >}}
    design:
      spacing:
        padding: ['3rem', 0, '3rem', 0]
  - block: markdown
    id: paper-reading-library
    content:
      title: '论文精读馆'
      text: |-
        对照原文与中文译文，阅读论文推导、原始图表与逐段解读。

        [进入论文精读馆 →](https://ginyee-w.github.io/paper-reading-library/)
    design:
      spacing:
        padding: ['2rem', 0, '2rem', 0]
  # ③ 原来的文章列表 block 保持不动
  - block: collection
    content:
      filters:
        folders:
          - blog
    design:
      spacing:
        padding: ['3rem', 0, '6rem', 0]
---



