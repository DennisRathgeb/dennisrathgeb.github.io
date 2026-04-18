---
layout: article
title: Projekte
permalink: /projekte/
lang: de
key: page-projekte
header_nav: global-de
sidebar:
  nav: nav-de
---

<section class="projects-grid">
  <div class="container">
    <div class="grid">
      {%- for project in site.data.projects -%}
      {%- assign loc = project.de | default: project.en -%}
      <div class="cell">
        <div class="card">
          <div class="card__image">
            <img class="image" src="{{ project.image | relative_url }}" alt="{{ loc.title }}">
          </div>
          <div class="card__content">
            <div class="card__header">
              <h4>{{ loc.title }}</h4>
            </div>
            <p>{{ loc.summary }}</p>
            <a href="{{ '/projekte/' | append: project.slug | append: '/' | relative_url }}" class="button">Mehr erfahren</a>
          </div>
        </div>
      </div>
      {%- endfor -%}
    </div>
  </div>
</section>
