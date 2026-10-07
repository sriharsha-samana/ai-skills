# ai-skills

Agent skills in the open [Agent Skills](https://agentskills.io) `SKILL.md` format. Works with Claude Code, Codex, Copilot, Cursor and other agents that support the standard. For agents that don't, paste the body of a `SKILL.md` into the agent's custom instructions or rules.

## Skills

| Skill | Description |
|---|---|
| [vue-nuxt-vuetify-upgrade](skills/vue-nuxt-vuetify-upgrade/SKILL.md) | Detect the current Vue / Nuxt / Vuetify setup and upgrade it to the latest Nuxt + Vue 3 + Vuetify + Pinia with feature parity, a mock API toggle and frontend-only security fixes |

## Install

Copy or symlink a skill folder into your agent's skills directory, e.g. for Claude Code:

```bash
ln -s "$PWD/skills/vue-nuxt-vuetify-upgrade" ~/.claude/skills/
```
