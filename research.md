---
layout: archive
title: "Research"
permalink: /research/
author_profile: false
---

연구 조사 및 기술 분석 자료를 정리하는 공간입니다.

{% assign research_posts = site.posts | where_exp: "post", "post.categories contains 'research'" %}
{% for post in research_posts %}
  {% include archive-single.html %}
{% endfor %}
