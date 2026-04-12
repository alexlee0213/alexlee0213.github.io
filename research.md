---
layout: page
title: Research
permalink: /research/
---

연구 조사 및 기술 분석 자료를 정리하는 공간입니다.

{% for item in site.research %}
### [{{ item.title }}]({{ item.url | relative_url }})
{{ item.summary }}

*{{ item.date | date: "%Y년 %m월 %d일" }}* · {{ item.tags | join: ", " }}

---
{% endfor %}
