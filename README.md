# Freesolo docs

[Mintlify](https://mintlify.com) documentation site for Freesolo. Right now it
covers **Flash** (the managed training + serving CLI); more product areas can be
added as new groups in `docs.json`.

This repo IS the Mintlify deployment source (the site builds from the repo root).
The pages were moved here from the `freesolo-co/freesolo` monorepo (`docs/`) so the
Mintlify project deploys the right content.

- **Live site:** https://freesolo.mintlify.app
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
  cli.mdx                 # every flash command + flag
  configuration.mdx       # every config TOML field
```

## Editing

- Pages are `.mdx` with YAML frontmatter (`title`, `description`).
- Add a new page by creating the `.mdx` file and adding its path (without the
  extension) to the right group in `docs.json` `navigation.groups`.
- Keep content accurate to the `flash` CLI; when a command or config field
  changes in [flash](https://github.com/freesolo-co/flash), update the matching
  reference page here.
