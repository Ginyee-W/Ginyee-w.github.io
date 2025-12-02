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
    # 2. 首页轮播相册（自定义 shortcode）
  - block: markdown
    id: home-slider
    content:
      title: ''
      text: |-
        {{< home-slider >}}
    design:
      spacing:
        padding: ['2rem', 0, '4rem', 0]
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
