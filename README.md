# modern-web-skills

> The skills I reach for at the beginning, middle, and end of a modern web project.

A small, personal collection of agent skills for `npx skills`. Tight, opinionated, copy-paste-runnable. Built and maintained by [VanColt](https://github.com/VanColt) — every skill here is one I use in real projects (this site's stack, agency work, and a handful of side projects), not synthesized from documentation.

[![skills.sh](https://skills.sh/b/VanColt/modern-web-skills)](https://skills.sh/VanColt/modern-web-skills)

## Why this exists

I build modern web apps — Next.js, React 19, Tailwind 4, Vercel. Coding agents don't always have current information about the frameworks I use. They suggest `middleware.ts` (renamed to `proxy.ts` in Next 16), pass `--turbopack` to `next build` (the default now), and reach for `any` when they should reach for Zod.

This repo is the set of skills that **fix the agent's defaults** to match the way I actually build. Each skill is small, opinionated, and tries to be the only thing you need for one specific moment of a project.

## Install

```bash
npx skills add VanColt/modern-web-skills
```

Install a specific skill:

```bash
npx skills add VanColt/modern-web-skills --skill scaffold-nextjs-16
```

## Skills

| Skill | When to use it | Install |
|---|---|---|
| [scaffold-nextjs-16](./skills/scaffold-nextjs-16/SKILL.md) | Scaffolding a new Next.js 16 project, working in an existing one, or upgrading from Next 15. Forces the agent to read the local `node_modules/next/dist/docs/` before writing code, and to use the v16 conventions (Turbopack by default, `proxy.ts` not `middleware.ts`, async params, `cacheComponents`, React 19.2). | `npx skills add VanColt/modern-web-skills --skill scaffold-nextjs-16` |

## Adding a new skill

```bash
npx skills init my-new-skill
```

Drop the new `SKILL.md` into `./skills/<name>/`, update this README, and push.

## About the author

Hi, I'm Mert. I'm a tech enthusiast and early AI adopter, hooked since the GPT-3 days. I've decided to open up my skills library to the public to share some of what I've picked up building my own projects: everything from the simple things that just save me time, to the more advanced ones that have pulled me out of a dark place.

## License

MIT. See [LICENSE](./LICENSE).
