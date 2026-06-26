# Skill Bundles

Skill bundles let you turn a repeatable cluster of Hermes skills into one slash command. Use them when the same task type repeatedly needs several skills loaded together, for example backend debugging, content intake, or recurring research workflows.

## Why use bundles

Without a bundle, you manually load each skill:

```text
/skill systematic-debugging
/skill test-fix-loop
/skill github-operations
```

With a bundle, you load the whole set:

```text
/backend-work
```

That reduces friction and makes recurring workflows more consistent.

## Create a bundle

First confirm the exact installed skill names:

```bash
hermes skills list
```

Then create the bundle:

```bash
hermes bundles create backend-work \
  --skill systematic-debugging \
  --skill test-fix-loop \
  --skill github-operations \
  --description "Backend debugging, test fixing, and GitHub workflow"
```

The bundle name becomes the slash command: `/backend-work`.

## Useful commands

```bash
hermes bundles list              # list installed bundles
hermes bundles show backend-work # inspect one bundle
hermes bundles reload            # rescan the bundles directory
hermes bundles delete backend-work
```

Bundles are stored under:

```text
~/.hermes/skill-bundles/
```

## Good bundle candidates

Create bundles for stable workflows, not one-off tasks:

- `backend-work` — debugging, test-fix loop, GitHub workflow.
- `content-intake` — X/article extraction, source analysis, KB capture.
- `research-brief` — web research, source synthesis, reasoning verification.
- `docs-update` — Hermes docs/Guide editing, write-verify, GitHub operations.

## Guardrails

- Use real installed skill names; social posts may use illustrative names that are not present locally.
- Keep bundles narrow. If a bundle loads ten unrelated skills, it becomes context bloat with a slash command.
- Prefer bundles for repeated workflows. For a single task, load only the exact skills needed.
- After creating or editing bundles used from a gateway session, use `/reset` or reload skills if the command list looks stale.
