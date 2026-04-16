# TamasCzaban.github.io

## What it does
Personal portfolio and blog site. Minimalist, high-performance static site showcasing work, projects, and technical writing. Targets 100/100 Lighthouse with all pages under 100KB.

## Tech stack
- **Framework:** Astro 4.4.13
- **Styling:** Tailwind CSS 3.4.1 + @tailwindcss/typography
- **Components:** SolidJS 1.8.15 (stateful UI only)
- **Content:** Markdown + MDX
- **Search:** fuse.js (client-side full-text)
- **SEO:** @astrojs/sitemap, @astrojs/rss
- TypeScript strict mode throughout

## How to run
```bash
npm install
npm run dev          # → http://localhost:4321
npm run build        # Build to ./dist/
npm run preview      # Preview production build
npm run lint         # ESLint check
npm run lint:fix     # Auto-fix ESLint issues
```

## Key files & structure
| Path | Role |
|------|------|
| `src/pages/index.astro` | Homepage |
| `src/pages/blog/` | Blog listing + individual posts |
| `src/pages/projects/` | Project showcase |
| `src/pages/work/` | Work history |
| `src/pages/search/` | Full-text search page |
| `src/consts.ts` | Site metadata, nav links, social profiles |
| `src/content/` | MDX/Markdown content (Astro content collections) |
| `src/content/config.ts` | Content collection schemas |
| `src/components/` | Astro + SolidJS components |
| `src/layouts/` | Page layout templates |
| `src/lib/` | Utility functions |
| `astro.config.mjs` | Astro config — integrations, site URL |
| `tailwind.config.mjs` | Tailwind customization |

## Deployment
- Builds to `./dist/` (static HTML/CSS/JS)
- Deployed to GitHub Pages via repo name `TamasCzaban.github.io`
- Also Netlify/Vercel compatible

## Conventions
- Use SolidJS (`.tsx`) only for stateful/interactive components; everything else is `.astro`
- Content goes in `src/content/` as MDX — never hardcoded in page components
- Don't disable Tailwind base styles (custom override in astro.config.mjs)
- Animated backgrounds (`MeteorShower`, `TwinklingStars`) are in `src/components/` — touch carefully as they affect performance budget
