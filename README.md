# DHIS2 App Skills

Agent skills for building [DHIS2](https://dhis2.org) custom applications. These skills give Claude Code reliable, up-to-date knowledge about the DHIS2 App Platform, API, component library, and common development patterns — areas where LLMs typically hallucinate.

## Install

```bash
npx skills add devotta-labs/dhis2-app-skills
```

## Included skills

| Skill | What it does |
|-------|-------------|
| `dhis2-app-dev` | Scaffolding, data fetching, mutations, metadata (querying, schemas, type recipes, sharing), UI patterns, routing, and deployment for DHIS2 apps |

## Why

DHIS2 has its own app framework, component library (`@dhis2/ui`), data layer (`@dhis2/app-runtime`), and Web API — none of which general-purpose LLMs handle well. These skills bundle curated reference docs and patterns so Claude produces correct DHIS2 code without reaching for generic libraries or deprecated endpoints.

## Contributing

PRs welcome. Each skill lives under `skills/<skill-name>/` with a `SKILL.md` and optional `references/` directory.
