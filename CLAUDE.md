# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build/Dev Commands
- **Build site**: `hugo`
- **Dev server**: `hugo server -D` (includes drafts)
- **Production build**: `hugo --gc --minify`
- **Clean build**: `rm -rf public resources && hugo`
- **New post**: `hugo new content/posts/post-name.md`

## Project Structure
- Hugo static site using `hugo-texify3` theme (git submodule at `themes/hugo-texify3`)
- Config: `hugo.toml` (TOML format)
- Content: Markdown files in `content/posts/`
- Theme overrides: `layouts/`, `assets/`, `static/` directories
- Deployment: GitHub Actions (`.github/workflows/deploy.yml`) to GitHub Pages

## Content Conventions
- Front matter: TOML format with `+++` delimiters
- Required fields: `date`, `title`, `draft`
- Supports: KaTeX math (`math = true` in config), Mermaid diagrams, syntax-highlighted code blocks
- Filename format: lowercase with hyphens (`my-blog-post.md`)

## Important Rules
- **DO NOT** modify files in `themes/hugo-texify3/` - it's a git submodule
- Override theme files by copying to root `layouts/` or `assets/` directories
- The theme requires Hugo extended version (uses Dart Sass)
