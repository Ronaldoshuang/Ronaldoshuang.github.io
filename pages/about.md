---
layout: page
title: About
description: 努力的意义，就是，在以后的日子里，放眼望去全是自己喜欢的人和事！
keywords: 爽哥
comments: true
menu: 关于
permalink: /about/
---

努力的意义，就是，在以后的日子里，放眼望去全是自己喜欢的人和事！

## 联系

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## Skill Keywords

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
