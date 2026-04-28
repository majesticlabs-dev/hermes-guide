# Installing Hermes skills from a URL

You do not need to clone a repo just to install a Hermes skill.

Hermes can install a skill directly from any URL that points to a Markdown file:

```bash
hermes skills install https://example.com/path/to/SKILL.md
```

This works with Gists, raw GitHub files, your own server, shared drives, or any other URL that ends in `.md`.

Hermes runs its security scan during install, the same way it does for other skill installs.

## Updating URL-installed skills

When a skill was installed from a URL, refresh it with:

```bash
hermes skills update
```

Hermes re-fetches the skill from the original URL.

## Slash command version

The same pattern works from chat surfaces that support Hermes slash commands, including the CLI and Telegram:

```text
/skills install https://example.com/path/to/SKILL.md
```

## When to use this

Use URL install when you want to share a single skill quickly without publishing a whole skill pack or asking someone to clone a repository.

Good examples:

- a one-off internal SOP skill
- a Gist-hosted experimental skill
- a skill shared from a private server
- a lightweight community skill before it graduates into a pack

## Source

Captured from Tony Simons' Hermes tip:

https://x.com/tonysimons_/status/2048760142305304676
