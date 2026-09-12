# [thecodepunk.com](https://thecodepunk.com)

The source code for my personal website: software engineering
articles, projects, a résumé, and book about the Zig
programming language.

The site is built with [Hugo](https://gohugo.io/) and a custom
[hugo-theme-codepunk](https://github.com/KyKyPy3/hugo-theme-codepunk) theme,
included as a Git submodule.

## Run locally

You will need Git and **Hugo Extended 0.143.1**, the minimum version declared by
the theme.

Clone the repository together with its theme:

```sh
git clone --recurse-submodules https://github.com/KyKyPy3/thecodepunk.com.git
cd thecodepunk.com
```

If you have already cloned the repository without its submodules, initialize
the theme separately:

```sh
git submodule update --init --recursive
```

Start the development server:

```sh
hugo server --buildDrafts
```

The site will be available at <http://localhost:1313>.

## Build

Create a minified production build with:

```sh
hugo --minify
```

The generated site is written to `public/`. This directory is ignored by Git.

## Add a post

Create a new page bundle:

```sh
hugo new content posts/my-article/index.md
```

Hugo will populate its front matter from `archetypes/default.md`. Posts with
`draft: true` are visible when the local server runs with `--buildDrafts`, but
are excluded from a regular production build. Images used by a single article
can be stored next to its `index.md` file.

## Project structure

```text
.
├── archetypes/   # Front matter templates for new content
├── content/      # Pages, articles, and the Zig Book table of contents
├── static/       # Images, favicons, and robots.txt
├── themes/       # hugo-theme-codepunk Git submodule
└── hugo.toml     # Main site configuration
```

## License

This project is available under the [MIT License](LICENSE).
