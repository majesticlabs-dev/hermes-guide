# Model Routing for Hermes

Choose routing for the user's workload and approved budget, not from an old fleet's model list. The earlier Beelink/Kimi/GLM/GPT-5.5 lane table was deployment-specific and is no longer an active setup recommendation. It did not establish universal Hermes defaults or current provider availability.

## Default delegation

With `delegation.backend: default`, children inherit the parent's model/provider unless the profile config overrides them:

```yaml
delegation:
  backend: default
  provider: <approved-provider>
  model: <verified-model-id>
```

This is a documentation template, not runnable configuration. Resolve the exact provider/model identifiers and obtain approval before changing cost or routing settings. Leaving the overrides unset inherits the parent; naming a lane in a prompt does not configure it.

The default backend has no per-task raw provider/model override. This restriction is **not** true of every delegation strategy.

## Optional per-task routing

Current Hermes supports delegation-provider plugins. The official `advanced-delegation` plugin supplies:

- `model_map`: the agent chooses a named model option configured by the operator.
- `profile_routes`: the agent chooses an allowed profile, with that profile's model, credentials, skills, and persona.

These use `delegation.backend` and `delegation.backends.<backend>`. While a strategy is active, its routes replace the default `delegation.model` / `delegation.provider` selection. Inspect the installed plugin and exposed tool schema before using strategy-specific fields; do not invent arguments to `delegate_task`.

Profile routes grant access to another profile's credentials and terminal policy. Enable only explicitly approved profiles. Missing plugins or invalid routes are errors, not permission to silently switch providers.

See [official delegation documentation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation#delegation-strategy-plugins) for installation and configuration appropriate to your version.

## Separate profile or child process

An independently configured profile is another supported option:

```bash
hermes profile list
hermes profile show <profile>
hermes -p <profile> chat -q '<self-contained task>'
```

A one-shot process can select an approved provider/model explicitly:

```bash
hermes chat --provider <approved-provider> -m <verified-model-id> -q '<task>'
```

These are templates. Discover real values before a live handoff. Child processes and profile invocations can spend money and perform side effects; they are not read-only routing probes.

## Cost and verification rules

1. Prefer deterministic scripts or `no_agent` jobs when no judgment is needed.
2. Follow the user's active model policy. Do not substitute historical cheap/premium lanes or switch providers merely because this Guide mentions them.
3. Keep fallback chains explicit and approved; a primary model pin alone does not establish a hard spend cap.
4. Use bounded, authorized smoke tests for changed routes. Confirm the actual provider/model in runtime evidence, not just the requested label.
5. Start a fresh session when the selected backend changes so its tool schema and routing agree.
6. Back up the affected config, inspect the exact diff, and retain rollback instructions. Check cron pins separately.

A response that merely prints `delegate_task(...)` is not execution. Call the real tool, or report the missing prerequisite. Never use placeholders in a claimed live handoff.

## Sources

- [Subagent Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation)
- [Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)
- [Cron model resolution and pins](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)

The [April 2026 model notes](model-tier-rankings.md) are historical evaluation material, not a current routing policy.
