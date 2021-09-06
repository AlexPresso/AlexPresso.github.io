---
title: "{{ replace .Name "-" " " | title }}"
description: default description
date: {{ .Date }}
draft: true
cover: "/img/posts/{{ replace .Name "-" " " | title }}/cover.jpg"
images: [ /img/posts/{{ replace .Name "-" " " | title }}/cover.jpg ]
tags:
    - no-tag
---

