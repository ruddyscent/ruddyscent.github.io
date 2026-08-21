---
layout: page
title: 진행 중(?)인 프로젝트
subtitle: 고민하거나 만지작거리는 것들
cover-img: /assets/img/develop.jpeg
thumbnail-img: /assets/img/toy-plane.webp
share-img: /assets/img/develop.jpeg
---

<style>
.project-heading {
  display: flex;
  align-items: center;
  gap: 0.65rem;
}

.project-heading img {
  width: 2.4rem;
  height: 2.4rem;
  flex: 0 0 2.4rem;
  margin: 0;
  border-radius: 0.45rem;
  object-fit: cover;
}
</style>

<h2 class="project-heading">
  <img src="/assets/img/toy-plane.webp" alt="" aria-hidden="true">
  <span>HoverPilot</span>
</h2>

강화학습을 이용해서 RC 비행기를 호버링하는 프로젝트입니다. 환경으로는 [RealFlight 시뮬레이터](https://www.realflight.com/)를 씁니다. [SITL 구성](https://ardupilot.org/dev/docs/sitl-with-realflight.html)으로 모든 요소를 소프트웨어로 진행합니다.

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

<h2 class="project-heading">
  <img src="/assets/img/fdtd-sinc-field.webp" alt="" aria-hidden="true">
  <span>GMES</span>
</h2>

맥스웰 방정식을 유한차분 시간영역법(FDTD)으로 푸는 전자기 시뮬레이터입니다. 최신 FDTD 알고리즘을 적용하고 다양한 하드웨어와 AI를 활용해 재미있는 실험을 시도하고 있습니다. 코드는 [GMES 저장소](https://github.com/ruddyscent/gmes)에 공개되어 있습니다.

{% assign date_format = site.date_format | default: "%B %-d, %Y" %}
<div class="post-list">
{% for post in site.tags['gmes'] %}
    <div class="tag-entry">
        <a href="{{ post.url | relative_url }}">{{- post.title | strip_html -}}</a>
        <div class="entry-date">
            <time datetime="{{- post.date | date_to_xmlschema -}}">{{- post.date | date: date_format -}}</time>
        </div>
    </div>
{% endfor %}
</div>

<h2 class="project-heading">
  <img src="/assets/img/ionosphere-fdtd-globe-simple.webp" alt="" aria-hidden="true">
  <span>IonosphereFDTD</span>
</h2>

지구 표면과 하부 전리층 사이에서 ELF 전자기파가 전파되는 과정을 계산하는 프로젝트입니다. 정이십면체에서 만든 측지 주·쌍대 격자 위에 FDTD를 구현하고 NumPy와 PyTorch로 최적화까지 적용합니다. 코드와 검증 자료는 [IonosphereFDTD 저장소](https://github.com/ruddyscent/ionosphere-fdtd)에 공개해 두었습니다.

{% assign date_format = site.date_format | default: "%B %-d, %Y" %}
<div class="post-list">
{% for post in site.tags['ionosphere'] %}
    <div class="tag-entry">
        <a href="{{ post.url | relative_url }}">{{- post.title | strip_html -}}</a>
        <div class="entry-date">
            <time datetime="{{- post.date | date_to_xmlschema -}}">{{- post.date | date: date_format -}}</time>
        </div>
    </div>
{% endfor %}
</div>
