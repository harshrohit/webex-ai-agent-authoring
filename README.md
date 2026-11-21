# webex-ai-agent-authoring

An agent skill for designing, creating, editing and reviewing **Webex AI Agent Studio** agents through the `ai-agent-studio` MCP server (`webex-ai-studio-*` tools).

It gives the assistant the judgment the MCP doesn't: what to ask before building, how to choose between speech-to-speech (S2S) and cascade engines, how to shape actions, prompt templates for each engine family, voice tuning, knowledge-base handling, a review checklist and test scenarios. The MCP still supplies the facts (schemas, engines, voices, functions) and does the saving.

The skill is a single `SKILL.md` in the [Agent Skills](https://agentskills.io) format, so the same file works in Claude (web, desktop, Claude Code) and OpenAI Codex.

## Before you install

- **The MCP server must be connected** in the client where you use the skill. The skill teaches how to use the `webex-ai-studio-*` tools; it does not provide them.
- **Studio access:** read access for discovery and review; Studio admin access plus `cjp:config_write` for anything that creates or changes agents and knowledge bases.
- **Writes are live.** The skill tells the assistant to show you a summary and wait for your yes before any create or update, but those calls change your real Studio org.

## Install

### Claude (claude.ai and the Claude desktop app)

Claude takes a ZIP whose top-level folder is the skill folder, and the folder name must match the skill's `name` (`webex-ai-agent-authoring`).

> Don't use GitHub's **Code → Download ZIP**: it names the folder `webex-ai-agent-authoring-main`, which won't match.

1. Build the ZIP from a clone of this repo:

   ```bash
   git clone https://github.com/harshrohit/webex-ai-agent-authoring
   cd webex-ai-agent-authoring
   git archive --format=zip --prefix=webex-ai-agent-authoring/ \
     -o ../webex-ai-agent-authoring.zip HEAD SKILL.md
   ```

2. In Claude, go to **Customize → Skills**, click **+**, choose **Create skill → Upload a skill**, and select `webex-ai-agent-authoring.zip`.
3. Make sure the skill is toggled on.

To make it available to everyone in a Team or Enterprise organization, an owner can upload the same ZIP under the organization's skill settings instead. Organization-provisioned skills also load in Claude Code for users signed in with their Claude account.

### Claude Code

Clone straight into a skills folder. The folder name is the skill name.

```bash
# Personal: available in every project
git clone https://github.com/harshrohit/webex-ai-agent-authoring \
  ~/.claude/skills/webex-ai-agent-authoring

# Or project-only: commit it with the repo
git clone https://github.com/harshrohit/webex-ai-agent-authoring \
  .claude/skills/webex-ai-agent-authoring
```

Keep `SKILL.md` directly inside `.../skills/webex-ai-agent-authoring/`; one folder deeper and it won't be found. Changes in an existing skills folder load without a restart; if you created the `skills` folder itself during a session, restart Claude Code.

### OpenAI Codex

```bash
# Personal
git clone https://github.com/harshrohit/webex-ai-agent-authoring \
  ~/.codex/skills/webex-ai-agent-authoring

# Or repository-scoped (shared Agent Skills location)
git clone https://github.com/harshrohit/webex-ai-agent-authoring \
  .agents/skills/webex-ai-agent-authoring
```

Restart Codex if the skill doesn't appear. Codex's docs list both `.codex/skills` and `.agents/skills` locations; if one isn't picked up, check the paths your Codex version scans.

## Use

The skill triggers on its own when you ask for AI Agent Studio work. You can also call it by name: `/webex-ai-agent-authoring` in Claude Code, or `$webex-ai-agent-authoring` in Codex.

Example requests:

- "Build a voice agent for our dental clinic that books, cancels and reschedules appointments."
- "Review the Payment_Agent design and tell me what's missing."
- "Callers say the agent keeps cutting them off. Fix the voice settings on Grace."
- "Create a knowledge base from these FAQs and attach it to the concierge agent."
- "How many AI agents do we have, and which ones look like test agents?"

What it does not do (the MCP can't either): publish agents, bind them in Flow Designer, build flow branches, set handover/feedback flags, or delete anything. The skill ends each build with a list of those manual steps.

## Update

```bash
cd <your skills folder>/webex-ai-agent-authoring && git pull
```

For Claude web and desktop, rebuild the ZIP and upload it again.

## Troubleshooting

| Problem | Fix |
|---|---|
| `Error in user YAML ... mapping values are not allowed` | A frontmatter value contains `: ` and isn't quoted. Keep `description` in double quotes. |
| Upload rejected or skill not found | The ZIP's top folder must be `webex-ai-agent-authoring/` with `SKILL.md` directly inside. Rebuild with the `git archive` command above. |
| Skill never triggers | Check it's enabled, then call it by name once. Make sure the `ai-agent-studio` MCP is connected in that client. |
| Rate-limit errors from the MCP | Expected when tools run in parallel; the skill tells the assistant to call them one at a time. |

## Repository layout

```
webex-ai-agent-authoring/
├── SKILL.md    # the skill (frontmatter + instructions)
└── README.md   # this file; not part of the uploaded skill
```
