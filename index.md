---
layout: page
title: 어서오소 Tech Blog
---

이화여대 **2026 오픈소스소프트웨어플랫폼** 팀 프로젝트 **어서오소(Ewha Market)** 팀의 기술 블로그입니다.
이화대동제 축제 부스의 상품을 등록하고, 둘러보고, 리뷰와 좋아요를 남기는 웹 서비스를 Flask와 Firebase로 만들면서 배운 것들을 정리했습니다.

- 프로젝트 저장소: [devleeji/2026_osp_project_welcome](https://github.com/devleeji/2026_osp_project_welcome)
- 기술 스택: Python Flask, Jinja2, HTML/CSS/JavaScript, Firebase Realtime Database (pyrebase)

## 글 목록

{% assign sorted_posts = site.posts | sort: "order" %}
| 유형 | 제목 | 작성자 |
|---|---|---|
{% for post in sorted_posts %}| {{ post.type }} | [{{ post.title }}]({{ post.url | relative_url }}) | {{ post.author }} |
{% endfor %}
