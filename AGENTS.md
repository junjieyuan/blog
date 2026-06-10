# AGENTS.md

## Project

Hugo static blog using the [hugo-paper](https://github.com/nanxiaobei/hugo-paper) theme, deployed at `https://blog.junjie.pro/`.

## Commands

```bash
hugo server -D      # dev server (serve drafts)
hugo                # build to public/
```

- `-D` / `--buildDrafts` must be passed to preview posts with `draft: true`.
- Output goes to `public/` — it's gitignored.

## Content conventions

- Posts live at `content/posts/YYYY/MM/DD/slug.md`.
- Frontmatter date includes timezone: `date: 2026-06-10T23:00:00+08:00`.
- Frontmatter uses TOML-style delimiters (`---`).
- Optional fields: `categories` (list), `tags` (list). Both use square-bracket TOML syntax: `categories: ["A"]`, `tags: ["X", "Y"]`.

## Theme & modules

- Theme is pulled via Hugo Modules, tracked in `go.mod` (`require github.com/nanoxiaobei/hugo-paper`).
- Do not add `layouts/` or `assets/` overrides unless necessary — the theme handles everything.
- If adding a new Hugo module, run `hugo mod get` not manual `go get`.

## Language

Content is written in Chinese (`languageCode = 'zh-cn'`). Keep frontmatter titles and user-facing text in Chinese unless there's a good reason otherwise.
