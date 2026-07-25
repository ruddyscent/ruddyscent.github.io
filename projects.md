---
layout: page
title: 진행 중(?)인 프로젝트
subtitle: 고민하거나 만지작거리는 것들
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
---

## HoverPilot

강화학습을 이용해서 RC 비행기를 호버링하는 프로젝트입니다. [RealFlight 시뮬레이터](https://www.realflight.com/)를 이용한 [SITL 환경](https://ardupilot.org/dev/docs/sitl-with-realflight.html)에서 진행하고 있습니다.

{% assign date_format = site.date_format | default: "%B %-d, %Y" %}
<div class="post-list">
{% for post in site.tags['hoverpilot'] %}
    <div class="tag-entry">
        <a href="{{ post.url | relative_url }}">{{- post.title | strip_html -}}</a>
        <div class="entry-date">
            <time datetime="{{- post.date | date_to_xmlschema -}}">{{- post.date | date: date_format -}}</time>
        </div>
    </div>
{% endfor %}
</div>
