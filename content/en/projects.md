---
layout: article
title: Projects
permalink: /en/projects/
lang: en
key: page-projects
header_nav: global-en
sidebar:
  nav: nav-en
---

<section class="projects-grid">
  <div class="container">
    <div class="grid">
      {%- for project in site.data.projects -%}
      {%- assign loc = project.en | default: project.de -%}
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
            <a href="{{ '/en/projects/' | append: project.slug | append: '/' | relative_url }}" class="button">Learn more</a>
          </div>
        </div>
      </div>
      {%- endfor -%}
    </div>
  </div>
</section>
