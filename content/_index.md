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
  # ① 新增：首页滚动相册
  - block: slider
    content:
      slides:
        - title: '四姑娘山'
          content: '2023年7月31日 于阿坝藏族羌族自治州'
          align: center
          background:
            image:
              # 只写文件名，图片放在 assets/media/ 里
              filename: siguniangshan.jpg
              filters:
                brightness: 0.7
            position: center
        - title: '正阳门'
          content: '2023年9月13日 于北京'
          align: center
          background:
            image:
              filename: zhengyangmen.jpg
              filters:
                brightness: 0.7
            position: center
    design:
      # 自动高度；想固定高度可以写 '400px'
      slide_height: ''
      is_fullscreen: true   # 全屏轮播；不想全屏就改成 false
      loop: true            # 自动循环
      interval: 3000        # 每张停留 3000ms
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
