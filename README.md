# Stack — Airat Valiullin

A collection of opinionated how-to guides and reference pages for my dev stack.

**Live site:** [stack.airatvaliullin.com](https://stack.airatvaliullin.com)

## Local development

### Prerequisites

- [Hugo](https://gohugo.io/installation/) extended edition v0.150.0+
- [Node.js](https://nodejs.org/) (for Tailwind CSS)

### Run locally

```bash
npm install
hugo server -D
```

The `-D` flag includes draft content. The site is served at `http://localhost:1313/`.

### Create a new page

```bash
hugo new fastapi/my-new-guide.md
```

## Deployment (Cloudflare Pages)

The site auto-deploys on push to `main` via Cloudflare Pages.

### Cloudflare Pages settings

| Setting              | Value                       |
|----------------------|-----------------------------|
| Build command        | `npm ci && hugo --minify`   |
| Build output dir     | `public`                    |
| `HUGO_VERSION`       | `0.152.2`                   |
| `NODE_VERSION`       | `22`                        |

### Custom domain

The subdomain `stack.airatvaliullin.com` is configured as a custom domain in the Cloudflare Pages project settings with a CNAME record pointing to the Pages deployment.

## Theme

This site uses the [Hugo Neso](https://github.com/babeneso/hugo-neso) theme, installed as a git submodule in `themes/hugo-neso`.

To update the theme:

```bash
git submodule update --remote themes/hugo-neso
```
