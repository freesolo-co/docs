# Freesolo docs

[Mintlify](https://mintlify.com) documentation site for Freesolo. It covers
**Flash** (the managed training + serving CLI), the Freesolo platform dashboard,
and the separate `freesolo` agent CLI reference.

This repo IS the Mintlify deployment source (the site builds from the repo root).
The pages were moved here from the `freesolo-co/freesolo` monorepo (`docs/`) so the
Mintlify project deploys the right content.

- **Live site:** https://docs.freesolo.co
- **Mintlify project id:** `6a361f42ca07eb683ee6b96d`
- **Config:** [`docs.json`](docs.json) (navigation, theme, name)

## Preview locally

```bash
npm i -g mint     # the Mintlify CLI
mint dev          # from the repo root; serves http://localhost:3000
```

`mint dev` hot-reloads as you edit. Run `mint broken-links` to check internal links.

## Layout

```
docs.json                 # site config + navigation
index.mdx                 # Flash overview
quickstart.mdx            # install -> login -> train -> deploy -> chat
guides/
  training.mdx
  environments.mdx
  datasets.mdx
  deploy-and-chat.mdx
reference/
  models.mdx              # supported Flash model catalog and serving prices
  cli.mdx                 # every flash command + flag
  freesolo-agent-cli.mdx  # the separate freesolo command
  configuration.mdx       # every config TOML field
  cost-model.mdx          # Flash cost and billing guidance
  troubleshooting.mdx
```

## Editing

- Pages are `.mdx` with YAML frontmatter (`title`, `description`).
- Add a new page by creating the `.mdx` file and adding its path (without the
  extension) to the right group in `docs.json` `navigation.groups`.
- Keep content accurate to
  [flash](https://github.com/freesolo-co/flash) and
  [freesolo](https://github.com/freesolo-co/freesolo); when a command, config
  field, dashboard behavior, or billing rule changes, update the matching page
  here.
