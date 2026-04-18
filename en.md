---
layout: none
permalink: /en/
lang: en
key: page-home-en
title: Dennis Rathgeb — Electrical Engineer
---
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>Dennis Rathgeb — Electrical Engineer</title>
  <meta name="description" content="Dennis Rathgeb — Swiss electrical engineer. Embedded systems, signal processing and applied machine learning. I build real-world systems from hardware to software.">
  <link rel="icon" href="{{ '/favicon.ico' | relative_url }}">

  <link rel="preconnect" href="https://fonts.googleapis.com">
  <link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
  <link href="https://fonts.googleapis.com/css2?family=Fraunces:ital,opsz,wght,SOFT@0,9..144,300..900,0..100;1,9..144,300..900,0..100&family=Geist+Mono:wght@300..700&family=Geist:wght@300..700&display=swap" rel="stylesheet">

  <link rel="stylesheet" href="{{ '/assets/css/site.css' | relative_url }}">
</head>
<body>

  <nav class="nav" id="nav">
    <a href="#top" class="nav__brand">Dennis Rathgeb</a>
    <div class="nav__links">
      <a href="#projects" class="nav__link">Projects</a>
      <a href="#about" class="nav__link is-secondary">About</a>
      <a href="#contact" class="nav__link">Contact</a>
      <a href="{{ '/' | relative_url }}" class="nav__lang">DE</a>
    </div>
  </nav>

  <main id="top">

    <section class="sec hero" id="hero">
      <div>
        <div class="hero__kicker">
          <span class="hero__kicker-dot"></span>
          <span>Zurich · Available for projects</span>
        </div>

        <h1 class="hero__title">
          Engineer focused on<br>
          systems that work<br>
          in <em>practice</em>.
        </h1>

        <p class="hero__sub">
          Electrical engineer — embedded systems, signal processing and applied machine learning.
        </p>
        <p class="hero__lead">
          From STM32-based firmware to data pipelines and ML models. Work on the whole system — hardware and software, end to end.
        </p>

        <div class="hero__ctas">
          <a href="#projects" class="btn btn--primary">
            View projects <span class="btn__arrow">→</span>
          </a>
          <a href="#contact" class="btn btn--ghost">
            Contact
          </a>
        </div>

        <div class="hero__meta">
          <div class="hero__meta-item">
            <span class="hero__meta-key">Based in</span>
            <span class="hero__meta-val">Urdorf · Switzerland</span>
          </div>
          <div class="hero__meta-item">
            <span class="hero__meta-key">Studying</span>
            <span class="hero__meta-val">B.Sc. EE, ZHAW</span>
          </div>
          <div class="hero__meta-item">
            <span class="hero__meta-key">Focus</span>
            <span class="hero__meta-val">Embedded · DSP · ML</span>
          </div>
          <div class="hero__meta-item">
            <span class="hero__meta-key">Languages</span>
            <span class="hero__meta-val">DE · EN · FR</span>
          </div>
        </div>
      </div>

      <div class="hero__portrait reveal">
        <img src="{{ '/assets/images/portrait.jpg' | relative_url }}" alt="Dennis Rathgeb" loading="eager">
        <div class="hero__tag">
          <span class="hero__tag-dot"></span>
          <span>Electrical Engineer</span>
        </div>
      </div>

      <div class="hero__scroll">
        <span>Scroll</span>
        <span class="hero__scroll-line"></span>
      </div>
    </section>

    <section class="sec overview" id="overview">
      <div class="sec__label">01 · Overview</div>
      <h2 class="sec__title" style="max-width: 22ch;">Hardware, software and integrated systems.</h2>

      <div class="overview__grid">
        <div class="over__col reveal">
          <h3>Focus areas</h3>
          <ul class="over__list">
            <li><span class="k">Embedded Systems</span><span class="v">STM32 · ESP · nRF · C/C++</span></li>
            <li><span class="k">Data & Backend</span><span class="v">Python · APIs · Streaming</span></li>
            <li><span class="k">Full-Stack Development</span><span class="v">End-to-end system design</span></li>
            <li><span class="k">Signal Processing & ML</span><span class="v">DSP · Applied ML</span></li>
          </ul>
        </div>

        <div class="over__col reveal">
          <h3>What I bring</h3>
          <div class="over__bring">
            <div class="over__bring-item">
              <span class="over__bring-num">01</span>
              <span class="over__bring-text">A strong combination of hardware and software engineering.</span>
            </div>
            <div class="over__bring-item">
              <span class="over__bring-num">02</span>
              <span class="over__bring-text">Experience building complete systems — not just components.</span>
            </div>
            <div class="over__bring-item">
              <span class="over__bring-num">03</span>
              <span class="over__bring-text">Structured, maintainable architectures from day one.</span>
            </div>
            <div class="over__bring-item">
              <span class="over__bring-num">04</span>
              <span class="over__bring-text">A hands-on mindset with real-world delivery experience.</span>
            </div>
          </div>
        </div>

        <div class="over__col reveal">
          <h3>Short profile</h3>
          <p>
            B.Sc. Electrical Engineering (ZHAW), previously ETH Zurich. Focus on embedded systems, DSP and machine learning. Project and industry experience incl. STRABAG; officer training in the Swiss Armed Forces.
          </p>
        </div>
      </div>
    </section>

    <section class="sec projects" id="projects">
      <div class="sec__label">02 · Selected projects</div>
      <div class="projects__head">
        <h2 class="sec__title" style="max-width: 18ch; margin: 0;">Selected projects.</h2>
        <p class="projects__lead">Three projects that show the working approach — from production field deployment and a Bachelor's thesis to radar sensing.</p>
      </div>

      {%- assign featured_slugs = "nalps-chrome-extension,thermaltrack,doppler-radar-detector" | split: "," -%}
      <div class="listing__grid">
        {%- for slug in featured_slugs -%}
          {%- assign outer_idx = forloop.index | prepend: "0" | slice: -2, 2 -%}
          {%- for project in site.data.projects -%}
            {%- if project.slug == slug -%}
              {%- assign loc = project.en | default: project.de -%}
              <a class="listing-card reveal" href="{{ '/en/projects/' | append: project.slug | append: '/' | relative_url }}">
                <div class="listing-card__img">
                  <img src="{{ project.image | relative_url }}" alt="{{ loc.title | escape }}" loading="lazy">
                </div>
                <div class="listing-card__body">
                  <span class="listing-card__idx">{{ outer_idx }} / Project</span>
                  <h3 class="listing-card__title">{{ loc.title }}</h3>
                  <p class="listing-card__sum">{{ loc.summary }}</p>
                  {%- if project.tags -%}
                  <ul class="listing-card__tags">
                    {%- for tag in project.tags limit: 4 -%}
                    <li>{{ tag }}</li>
                    {%- endfor -%}
                  </ul>
                  {%- endif -%}
                  <span class="listing-card__cta">Read more →</span>
                </div>
              </a>
            {%- endif -%}
          {%- endfor -%}
        {%- endfor -%}
      </div>

      <a href="{{ '/en/projects/' | relative_url }}" class="projects__more">
        View all projects <span>→</span>
      </a>
    </section>

    <section class="sec about" id="about">
      <div class="sec__label">03 · About me</div>

      <div class="about__grid">
        <div class="about__media reveal">
          <img src="{{ '/assets/images/portrait.jpg' | relative_url }}" alt="Dennis Rathgeb portrait">
          <div class="about__media-caption">
            Winterthur, 1997 — now based in Urdorf, Zurich. Mountains, workbench, code.
          </div>
        </div>

        <div class="about__text reveal">
          <p class="lead">
            Electrical engineer focused on <em>practical systems</em> — things that actually work, not just on paper.
          </p>

          <p>
            The profile combines embedded systems, software engineering and data-driven modeling.
          </p>

          <p>
            I work across the whole stack — from low-level firmware to backend and data processing. The emphasis is on clean system architecture and measurable outcomes.
          </p>

          <p>
            On the non-technical side: several years of project experience in industry, plus officer training in the Swiss Armed Forces. Taking responsibility, structuring work and delivering under real constraints is part of the profile.
          </p>

          <p>
            Balance is found in the mountains: climbing, ski touring, paragliding, running.
          </p>
        </div>
      </div>
    </section>

    <section class="sec approach" id="approach">
      <div class="sec__label">04 · Approach</div>
      <h2 class="sec__title" style="max-width: 22ch;">Four principles that carry every project.</h2>

      <ol class="approach__list">
        <li class="approach__item reveal">
          <span class="approach__num">01</span>
          <span class="approach__head">System before component</span>
          <span class="approach__body">Focus on the whole system and its interfaces — not on individual building blocks in isolation.</span>
        </li>
        <li class="approach__item reveal">
          <span class="approach__num">02</span>
          <span class="approach__head">Architecture from the start</span>
          <span class="approach__body">Clean, maintainable structures defined before the first line of code is written.</span>
        </li>
        <li class="approach__item reveal">
          <span class="approach__num">03</span>
          <span class="approach__head">Physics and data</span>
          <span class="approach__body">Combining physical understanding with data-driven methods — more robust than either alone.</span>
        </li>
        <li class="approach__item reveal">
          <span class="approach__num">04</span>
          <span class="approach__head">Production systems, not demos</span>
          <span class="approach__body">Focus on solutions that run in production — with real users, real data, real failures.</span>
        </li>
      </ol>

      <div class="stack">
        <div class="stack__label">Core technologies</div>
        <div class="stack__tags">
          <span class="stack__tag">C / C++</span>
          <span class="stack__tag">Python</span>
          <span class="stack__tag">TypeScript</span>
          <span class="stack__tag">STM32</span>
          <span class="stack__tag">ESP32</span>
          <span class="stack__tag">nRF</span>
          <span class="stack__tag">Linux</span>
          <span class="stack__tag">Docker</span>
          <span class="stack__tag">FastAPI</span>
          <span class="stack__tag">React</span>
          <span class="stack__tag">PostgreSQL / PostGIS</span>
          <span class="stack__tag">Signal Processing</span>
          <span class="stack__tag">Machine Learning</span>
        </div>
      </div>
    </section>

    <section class="sec contact" id="contact">
      <div class="sec__label">05 · Contact</div>

      <div class="contact__grid">
        <div>
          <p class="contact__lead">
            Open to conversations about engineering roles spanning hardware and software. Usually reply within one working day.
          </p>
          <a href="mailto:dennis.rathgeb@gmx.ch" class="contact__big">
            dennis.rathgeb@gmx.ch
          </a>
        </div>

        <div class="contact__links reveal">
          <div class="contact__row">
            <span class="contact__row-k">Email</span>
            <a class="contact__row-v" href="mailto:dennis.rathgeb@gmx.ch">dennis.rathgeb@gmx.ch</a>
          </div>
          <div class="contact__row">
            <span class="contact__row-k">GitHub</span>
            <a class="contact__row-v" href="https://github.com/DennisRathgeb" target="_blank" rel="noopener">@DennisRathgeb ↗</a>
          </div>
          <div class="contact__row">
            <span class="contact__row-k">Location</span>
            <span class="contact__row-v">Urdorf · Canton of Zurich</span>
          </div>
          <div class="contact__row">
            <span class="contact__row-k">Phone</span>
            <a class="contact__row-v" href="tel:+41764463092">+41 76 446 30 92</a>
          </div>
        </div>
      </div>
    </section>

  </main>

  <footer class="foot">
    <span>© {{ site.time | date: "%Y" }} Dennis Rathgeb</span>
    <span>Built with Jekyll · Hosted on GitHub Pages</span>
  </footer>

  <script>
    (function() {
      var nav = document.getElementById('nav');
      var onScroll = function() {
        if (window.scrollY > 20) nav.classList.add('is-scrolled');
        else nav.classList.remove('is-scrolled');
      };
      window.addEventListener('scroll', onScroll, { passive: true });
      onScroll();
    })();

    (function() {
      var els = document.querySelectorAll('.reveal');
      if (!('IntersectionObserver' in window) || !els.length) {
        els.forEach(function(el){ el.classList.add('is-in'); });
        return;
      }
      document.documentElement.classList.add('has-js');
      var io = new IntersectionObserver(function(entries) {
        entries.forEach(function(e) {
          if (e.isIntersecting) {
            e.target.classList.add('is-in');
            io.unobserve(e.target);
          }
        });
      }, { threshold: 0.12, rootMargin: '0px 0px -5% 0px' });
      els.forEach(function(el){ io.observe(el); });
      setTimeout(function() {
        els.forEach(function(el){ el.classList.add('is-in'); });
      }, 2000);
    })();
  </script>
</body>
</html>
