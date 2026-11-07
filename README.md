# Minimaps plugin for Claude Code

This plugin installs the **minimaps** Agent Skill. With it, Claude Code knows how to add Minimaps static map images to an app:
- choosing between GET `<img>` URLs and POST JSON specs
- signing URLs with a publishable key and keeping secret keys out of URLs
- using the TypeScript or Python SDK when the project has one
- handling ambiguous and unknown place names
- staying within the API's limits

## Install

From the public repository [agilek/minimaps-claude-plugin](https://github.com/agilek/minimaps-claude-plugin):

```bash
claude plugin marketplace add agilek/minimaps-claude-plugin
claude plugin install minimaps@minimaps
```

Or, inside a Claude Code session: `/plugin marketplace add agilek/minimaps-claude-plugin`, then `/plugin install minimaps@minimaps`.

To try it from a local checkout without a marketplace, run `claude --plugin-dir integrations/claude-plugin`.

Other agents that read the Agent Skills format can use the `skills/minimaps` folder directly. Copy it into the agent's skills directory.

## Layout

```text
.claude-plugin/plugin.json        plugin manifest
.claude-plugin/marketplace.json   this folder is also a one-plugin marketplace (source "./")
skills/minimaps/                  the skill: SKILL.md plus reference/*.md, loaded on demand
LICENSE                           MIT
```

The folder is self-contained. It references nothing outside itself, so it can be published as its own repository: copy the folder's contents to the root of the mirror repository.

## Maintainers: do not edit `skills/` here

The Minimaps source repository is private, so marketplace installs need a public mirror of this folder.

`skills/minimaps/` is a generated copy of the canonical `skills/minimaps/` at the root of the Minimaps repository. There, `pnpm docs:gen` regenerates the parameter, error and header tables from the code registries and writes both copies, and `pnpm docs:check` fails CI if either copy is stale.

To publish, run `pnpm plugin:publish` in the source repository. It checks that the generated copy is current, then pushes a snapshot of this folder to [agilek/minimaps-claude-plugin](https://github.com/agilek/minimaps-claude-plugin). Edit only `README.md` and `.claude-plugin/*` here; changes made directly in the public repository are overwritten by the next publish.

## License

MIT, like the Minimaps SDKs. See [LICENSE](LICENSE).
