# AGENTS.md

Hexo 7.3 blog with Butterfly theme, deployed to GitHub Pages.

## Commands

```
npm run server    # local dev → http://localhost:4000
npm run build     # hexo generate → public/
npm run clean     # hexo clean (wipe public/ + cache)
npm run deploy    # hexo deploy (pushes to alex-tangt.github.io)
```

## Writing posts

- English: `npx hexo new post "title"`
- Chinese: `npx hexo new --scaffold zh-post "title"`
- Draft: `npx hexo new draft "title"`
- Page: `npx hexo new page "title"`

Every post **must** include `lang: en` or `lang: zh-CN` in frontmatter.
Post asset folders are enabled (`post_asset_folder: true`) — `hexo new` creates a folder with `index.md`.

## Architecture

- `source/_posts/` — blog articles (bilingual, each language is a separate `.md`)
- `source/about/`, `categories/`, `tags/` — standalone pages
- `source/_data/` — Hexo data files
- `source/img/` — images
- `scaffolds/` — post templates (post.md, zh-post.md, draft.md, page.md)
- Theme config lives in `_config.butterfly.yml`, not `themes/` (theme installed via npm)

## Deployment

- **CI** (`.github/workflows/deploy.yml`): on push to `main`, runs `npx hexo generate` then deploys `public/` via GitHub Pages actions. Does NOT use hexo-deployer-git.
- **Local**: `npm run deploy` uses `hexo-deployer-git` to push to `Alex-tangt/alex-tangt.github.io` main branch.

## Git-ignored

- `public/` — build output
- `db.json` — Hexo cache
- `.deploy_git/` — deployer working directory
- `node_modules/`
