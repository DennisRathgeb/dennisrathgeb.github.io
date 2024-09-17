---
layout: article
titles:
  en: Projects
key: page-about
header_nav: empty
sidebar:
  nav: home-nav
---

<section class="projects-grid">
  <div class="container">
    <div class="grid">
      {%- for project in site.data.projects -%}
      <div class="cell">
        <div class="card">
          <div class="card__image">
            <img class="image" src="{{ project.image }}" alt="{{ project.title }}">
          </div>
          <div class="card__content">
            <div class="card__header">
              <h4>{{ project.title }}</h4>
            </div>
            <p>{{ project.description }}</p>
            <a href="{{ project.url }}" class="button">Learn More</a>
          </div>
        </div>
      </div>
      {%- endfor -%}
    </div>
  </div>
</section>
