---
title: 'Home'
date: 2023-10-24
type: landing

sections:
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



