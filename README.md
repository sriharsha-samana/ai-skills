# ai-skills

Agent skills in the open [Agent Skills](https://agentskills.io) `SKILL.md` format. Works with Claude Code, Codex, Copilot, Cursor and other agents that support the standard. For agents that don't, paste the body of a `SKILL.md` into the agent's custom instructions or rules.

## Skills

| Skill | Scope | Description |
|---|---|---|
| [vue-nuxt-vuetify-upgrade](skills/vue-nuxt-vuetify-upgrade/SKILL.md) | Frontend | Detect the current Vue / Nuxt / Vuetify setup and upgrade it to the latest Nuxt + Vue 3 + Vuetify + Pinia with feature parity, a mock API toggle and frontend-only security fixes |
| [play-framework-upgrade](skills/play-framework-upgrade/SKILL.md) | Backend | Detect a Java Play Framework setup, move it to `legacy/` and rebuild it on the latest Play and Java LTS with API parity, contract tests, a mock toggle for external integrations and in-service security fixes |
| [node-express-typescript-upgrade](skills/node-express-typescript-upgrade/SKILL.md) | Backend | Detect a Node.js + Express + TypeScript setup, move it to `legacy/` and rebuild it on the latest Node LTS, Express and TypeScript with API parity, contract tests, a mock toggle for external integrations and in-service security fixes |
| [angular-upgrade](skills/angular-upgrade/SKILL.md) | Frontend | Detect an AngularJS / Angular setup and bring it to the latest Angular (standalone, signals, built-in control flow): AngularJS rebuilt beside `legacy/`, Angular 2+ upgraded in place, with feature parity, a mock API toggle and frontend-only security fixes |
| [dotnet-upgrade](skills/dotnet-upgrade/SKILL.md) | Backend | Detect a .NET Framework / .NET setup, move it to `legacy/` and rebuild it on the latest .NET LTS and ASP.NET Core with API parity, contract tests, a mock toggle for external integrations and in-service security fixes |

## Install

Copy or symlink a skill folder into your agent's skills directory, e.g. for Claude Code:

```bash
ln -s "$PWD/skills/vue-nuxt-vuetify-upgrade" ~/.claude/skills/
```
