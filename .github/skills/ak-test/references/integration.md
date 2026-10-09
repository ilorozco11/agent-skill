# Installation and invocation

Copy the entire `ak-test` directory, including references, into the chosen project's agent skill directory. Keep a single canonical source and recopy deliberately on upgrades. Do not overwrite an existing skill without reviewing its differences. This is a prompt skill: it guides an agent; it is not a sandbox, test runner, or enforced CI policy.

| Agent | Project destination | Invocation |
| --- | --- | --- |
| GitHub Copilot | `.github/skills/ak-test/` | Ask “Use ak-test audit” in an agent surface supporting skills |
| Claude Code | `.claude/skills/ak-test/` | `/ak-test create`, `/ak-test audit`, `/ak-test optimize --ultra` |
| Codex | `.agents/skills/ak-test/` | `$ak-test create`, `$ak-test audit`, `$ak-test optimize --ultra` |
| Other agents | Agent's supported skill folder | Ask it to read `SKILL.md` and apply the chosen mode |

For older hosts, check their supported discovery paths. If automatic discovery is unavailable, explicitly point the agent to the installed SKILL.md. A host must provide repository access, shell/test execution, and CI logs to perform those operations; the skill itself grants no permissions or tools. Independent ultra requires subagent support.

From this GitHub repository root, install into a different project (set an actual absolute destination):

```bash
project_dir=/absolute/path/to/your/project
# Choose ONE appropriate destination:
skill_parent="$project_dir/.github/skills"   # Copilot
# skill_parent="$project_dir/.claude/skills" # Claude Code
# skill_parent="$project_dir/.agents/skills" # Codex
mkdir -p "$skill_parent"
if [ -e "$skill_parent/ak-test" ]; then
  printf '%s\n' 'Destination exists; review differences before updating.' >&2
  exit 1
fi
cp -R .github/skills/ak-test "$skill_parent/ak-test"
```

`/ak:test` from the motivating idea is a logical alias, not standard cross-agent syntax. The portable name is `ak-test`. Configure a host-specific command/plugin alias only if desired; do not claim it is installed by copying this skill.

Smoke-check discovery by asking: “Use ak-test audit on a specified test file; do not edit files.” Confirm the report names the mode, inspects actual behavior, and provides evidence. Runtime compatibility must be verified in the installed host; packaging validation alone does not prove it.

Sources for discovery/invocation conventions:
- https://docs.github.com/en/copilot/how-tos/copilot-on-github/customize-copilot/customize-cloud-agent/add-skills
- https://code.claude.com/docs/en/skills
- https://developers.openai.com/codex/skills

The workflow is an original implementation of the user-supplied idea. Do not imply affiliation with or copy an unavailable upstream AK implementation. Do not repeat anecdotal speedups as results of this skill.
