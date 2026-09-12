---
title: Talks
description: Slides from talks by Roman Efremenko about software architecture and systems programming.
aliases:
  - /presentations/
showTitle: false
showMeta: false
class: projects talks
---

{{<header class="center">}}
████████╗░█████╗░██╗░░░░░██╗░░██╗░██████╗
╚══██╔══╝██╔══██╗██║░░░░░██║░██╔╝██╔════╝
░░░██║░░░███████║██║░░░░░█████═╝░╚█████╗░
░░░██║░░░██╔══██║██║░░░░░██╔═██╗░░╚═══██╗
░░░██║░░░██║░░██║███████╗██║░╚██╗██████╔╝
░░░╚═╝░░░╚═╝░░╚═╝╚══════╝╚═╝░░╚═╝╚═════╝░
{{</header>}}

<p class="projects-intro">
A collection of slides from my talks. Open a deck in the browser or download
the original PDF.
</p>

<section class="project-list" aria-label="Presentation slides">
  <article class="project-card">
    <div class="project-card__visual">
      <img
        src="clean-architecture.webp"
        alt="Title slide of the Clean Architecture presentation"
        width="1400"
        height="788"
        loading="lazy"
        decoding="async"
      >
    </div>
    <div class="project-card__body">
      <span class="project-card__number" aria-hidden="true">01</span>
      <p class="project-card__kicker">Software architecture · 53 slides</p>
      <h2>Clean Architecture</h2>
      <p>
        A practical tour of software modularity: coupling and cohesion, the path
        from layered and hexagonal designs to Clean Architecture, and the roles
        of domain objects, use cases, ports, repositories, and bounded contexts.
      </p>
      <ul class="project-card__stack" aria-label="Clean Architecture topics">
        <li>Architecture</li>
        <li>DDD</li>
        <li>CQRS</li>
      </ul>
      <nav class="project-card__links" aria-label="Clean Architecture presentation links">
        <a href="clean-architecture.pdf" target="_blank" rel="noopener">View slides <span aria-hidden="true">↗</span></a>
        <a href="clean-architecture.pdf" download>Download PDF <span aria-hidden="true">↓</span></a>
      </nav>
    </div>
  </article>

  <article class="project-card">
    <div class="project-card__visual">
      <img
        src="webassembly-plugins.webp"
        alt="Title slide of the Heroes of Modularity presentation"
        width="1400"
        height="788"
        loading="lazy"
        decoding="async"
      >
    </div>
    <div class="project-card__body">
      <span class="project-card__number" aria-hidden="true">02</span>
      <p class="project-card__kicker">WebAssembly · 32 slides</p>
      <h2>Heroes of Modularity</h2>
      <p>
        An exploration of cross-platform plugin systems with low overhead and
        strong isolation. The talk compares scripting, IPC, and dynamic
        libraries before moving into WebAssembly, WASI, WIT, and components.
      </p>
      <ul class="project-card__stack" aria-label="Heroes of Modularity topics">
        <li>WebAssembly</li>
        <li>WASI</li>
        <li>Plugins</li>
      </ul>
      <nav class="project-card__links" aria-label="Heroes of Modularity presentation links">
        <a href="webassembly-plugins.pdf" target="_blank" rel="noopener">View slides <span aria-hidden="true">↗</span></a>
        <a href="webassembly-plugins.pdf" download>Download PDF <span aria-hidden="true">↓</span></a>
      </nav>
    </div>
  </article>
</section>
