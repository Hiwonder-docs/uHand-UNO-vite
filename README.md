# uHand_UNO Documentation

This repository contains the uHand_UNO VitePress documentation site. The
documentation source files are Markdown files under `docs/page/`.

## Local development

Install dependencies and start the local documentation server:

```bash
npm ci
npm run docs:dev
```

Build the production site:

```bash
npm run docs:build
```

The production files are generated in `docs/.vitepress/dist/`.

## GitHub Pages deployment

The workflow in `.github/workflows/deploy-docs.yml` builds the site and
publishes it to the `gh-pages` branch whenever `main` is pushed. For a repository named
`uHand_UNO-vite`, the initial GitHub Pages URL is:

```text
https://hiwonder-docs.github.io/uHand_UNO-vite/projects/uHand_UNO/en/latest/
```

After the workflow creates the `gh-pages` branch, open **Settings > Pages**,
select **Deploy from a branch**, and choose **gh-pages** and **/(root)**.

When a custom domain is configured later, the same site path becomes:

```text
https://wiki.hiwonder.com/projects/uHand_UNO/en/latest/
```
