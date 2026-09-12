---
title: Projects
description: Selected open-source projects by Roman Efremenko.
showTitle: false
showMeta: false
class: projects
---

{{<header class="center">}}
██████╗░██████╗░░█████╗░░░░░░██╗███████╗░█████╗░████████╗░██████╗
██╔══██╗██╔══██╗██╔══██╗░░░░░██║██╔════╝██╔══██╗╚══██╔══╝██╔════╝
██████╔╝██████╔╝██║░░██║░░░░░██║█████╗░░██║░░╚═╝░░░██║░░░╚█████╗░
██╔═══╝░██╔══██╗██║░░██║██╗░░██║██╔══╝░░██║░░██╗░░░██║░░░░╚═══██╗
██║░░░░░██║░░██║╚█████╔╝╚█████╔╝███████╗╚█████╔╝░░░██║░░░██████╔╝
╚═╝░░░░░╚═╝░░╚═╝░╚════╝░░╚════╝░╚══════╝░╚════╝░░░░╚═╝░░░╚═════╝░
{{</header>}}

<p class="projects-intro">
Sometimes I find a little time and write code. These are the projects where
curiosity turned into something useful.
</p>

<section class="project-list" aria-label="Selected projects">
  <article class="project-card">
    <div class="project-card__visual">
      <img
        src="octa.webp"
        alt="A cyberpunk build engine routing several parallel task pipelines"
        width="1400"
        height="788"
        loading="lazy"
        decoding="async"
      >
    </div>
    <div class="project-card__body">
      <span class="project-card__number" aria-hidden="true">01</span>
      <p class="project-card__kicker">Build automation</p>
      <h2>Octa</h2>
      <p>
        A fast, plugin-driven task runner and build tool written in Rust. Octa
        reads declarative YAML task files, resolves dependencies and includes,
        and runs independent work in parallel.
      </p>
      <ul class="project-card__stack" aria-label="Octa technologies">
        <li>Rust</li>
        <li>YAML</li>
        <li>Plugins</li>
      </ul>
      <nav class="project-card__links" aria-label="Octa links">
        <a href="https://github.com/OctaHive/octa">Source code <span aria-hidden="true">↗</span></a>
        <a href="https://github.com/OctaHive/octa/releases">Releases <span aria-hidden="true">↗</span></a>
      </nav>
    </div>
  </article>

  <article class="project-card">
    <div class="project-card__visual">
      <img
        src="octabot.webp"
        alt="A punk robot dispatching jobs from a circular scheduling console"
        width="1400"
        height="788"
        loading="lazy"
        decoding="async"
      >
    </div>
    <div class="project-card__body">
      <span class="project-card__number" aria-hidden="true">02</span>
      <p class="project-card__kicker">Task scheduler</p>
      <h2>Octabot</h2>
      <p>
        A plugin-based scheduler and automation service. It stores projects and
        recurring jobs in SQLite, exposes a REST API, and executes extensions as
        sandboxed WebAssembly components.
      </p>
      <ul class="project-card__stack" aria-label="Octabot technologies">
        <li>Rust</li>
        <li>WebAssembly</li>
        <li>SQLite</li>
      </ul>
      <nav class="project-card__links" aria-label="Octabot links">
        <a href="https://github.com/OctaHive/octabot">Source code <span aria-hidden="true">↗</span></a>
        <a href="https://github.com/OctaHive/octabot#setup">Setup guide <span aria-hidden="true">↗</span></a>
      </nav>
    </div>
  </article>

  <article class="project-card">
    <div class="project-card__visual">
      <img
        src="nezumo.webp"
        alt="An infinite collaborative canvas filled with connected visual ideas"
        width="1400"
        height="788"
        loading="lazy"
        decoding="async"
      >
    </div>
    <div class="project-card__body">
      <span class="project-card__number" aria-hidden="true">03</span>
      <p class="project-card__kicker">Visual collaboration</p>
      <h2>Nezumo</h2>
      <p>
        An open collaborative workspace built around an infinite canvas for
        planning, workshops, and presentations. It combines real-time editing,
        browser and desktop clients, rich media, and board export.
      </p>
      <ul class="project-card__stack" aria-label="Nezumo technologies">
        <li>Rust</li>
        <li>React</li>
        <li>WebAssembly</li>
      </ul>
      <nav class="project-card__links" aria-label="Nezumo links">
        <a href="https://nezumo.ru">Website <span aria-hidden="true">↗</span></a>
        <a href="https://app.nezumo.ru">Open app <span aria-hidden="true">↗</span></a>
        <a href="https://github.com/OctaHive/nezumo">Source code <span aria-hidden="true">↗</span></a>
      </nav>
    </div>
  </article>
</section>
