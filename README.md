# skills

A small collection of agent skills for `npx skills`. Tight, opinionated, copy-paste-runnable.

[![skills.sh](https://skills.sh/b/VanColt/skills)](https://skills.sh/VanColt/skills)

## Install

```bash
npx skills add VanColt/skills
```

Install specific skills:

```bash
npx skills add VanColt/skills --skill scaffold-nextjs-16
```

## Skills

| Skill | What it does | Install |
|---|---|---|
| [scaffold-nextjs-16](./skills/scaffold-nextjs-16/SKILL.md) | Scaffold a new Next.js 16 project the right way, or work in an existing one. Forces the agent to read the local `node_modules/next/dist/docs/` before writing code, and to use v16 conventions (Turbopack by default, `proxy.ts` not `middleware.ts`, async params, `cacheComponents`, React 19.2). Also covers upgrading from Next 15. | `npx skills add VanColt/skills --skill scaffold-nextjs-16` |

## Adding a new skill

```bash
npx skills init my-new-skill
```

Drop the new `SKILL.md` into `./skills/<name>/`, update this README, and push.

## License

MIT. See [LICENSE](./LICENSE).
