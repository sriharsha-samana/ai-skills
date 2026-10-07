# ai-skills

Agent skills in the open [Agent Skills](https://agentskills.io) `SKILL.md` format. Works with Claude Code, Codex, Copilot, Cursor and other agents that support the standard. For agents that don't, paste the body of a `SKILL.md` into the agent's custom instructions or rules.

## Skills

| Skill | Description |
|---|---|
| [vue2-to-nuxt-vuetify-migration](skills/vue2-to-nuxt-vuetify-migration/SKILL.md) | Migrate a Vue 2 + Vuetify 2 app to Nuxt + Vue 3 + Vuetify 3 + Pinia with feature parity, a mock API toggle and frontend-only security fixes |

## Install

Copy or symlink a skill folder into your agent's skills directory, e.g. for Claude Code:

```bash
ln -s "$PWD/skills/vue2-to-nuxt-vuetify-migration" ~/.claude/skills/
```
