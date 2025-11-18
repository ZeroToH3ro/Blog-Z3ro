# Repository Guidelines

## Project Structure & Module Organization
- `src/pages` holds routing entries; each file becomes a route. Keep page-level logic thin and delegate UI fragments to components.
- `src/components` stores reusable Astro or TSX components in PascalCase (`Header.astro`). Co-locate accompanying styles in `src/styles` when global, or alongside the component for scoped styles.
- `src/content` contains MD/MDX posts managed by `content.config.ts`. Use descriptive folders per topic (e.g., `move-basics/index.mdx`). Static assets that should bypass the Astro pipeline belong in `public/`; processed assets live in `src/assets`.
- Build outputs land in `dist/`; do not edit manually.

## Build, Test, and Development Commands
- `npm install` — install dependencies; rerun whenever `package.json` changes.
- `npm run dev` — start the Astro dev server with hot reload at `http://localhost:4321`.
- `npm run build` — create a production bundle in `dist/`; run before submitting PRs.
- `npm run preview` — serve the built site locally to verify production behavior.
- `npx astro check` (or `npm run astro -- check`) — type-check content collections and Astro components.

## Coding Style & Naming Conventions
- Use TypeScript modules with ES imports. Export constants from `consts.ts` and import via relative aliases.
- Favor two-space indentation across Astro, TS, and Markdown files. Wrap lines at ~100 characters.
- Components and layouts use PascalCase (`BlogLayout.astro`); utility functions use camelCase.
- Keep frontmatter concise: `title`, `description`, `pubDate`, `tags`. Asset filenames are kebab-case (`layer2-overview.png`).

## Testing Guidelines
- No automated suite ships yet; prioritize lightweight smoke tests before merging. Add Vitest integration tests under `src/tests` using `*.test.ts` filenames when touching logic-heavy helpers.
- For content or styling-only changes, validate via `npm run dev` in multiple viewports and run `npm run build` to ensure Astro’s content collections compile.
- Document manual test steps in the PR description so reviewers can reproduce.

## Commit & Pull Request Guidelines
- Follow the existing history: short, imperative summaries in lowercase (`update format`, `setup project`). Limit to 50 characters when possible and describe the scope, not the implementation details.
- Each PR should describe the change, list testing performed, and link related issues. Include before/after screenshots for visual tweaks and note any new content collections or config changes.
- Keep PRs focused; split large features into reviewable chunks. Rebase onto `main` before requesting review to avoid merge commits.
