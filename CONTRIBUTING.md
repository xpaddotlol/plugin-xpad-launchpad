# Contributing

Thanks for your interest in improving `xpad-launchpad`.

## Reporting issues

Open an issue at the source repo describing:
- What command or flow you ran.
- Expected vs actual behavior.
- Chain ID (should be `196` for X Layer mainnet), RPC used, and the tx hash if a transaction reverted.

## Pull requests

1. Fork and create a topic branch.
2. Keep changes focused — one fix or feature per PR.
3. If you add or change a command in `SKILL.md`, verify it maps to a real function in the bundled ABIs under `reference/abi/`. Do not invent function signatures.
4. Run `python3 -c "import yaml; yaml.safe_load(open('plugin.yaml'))"` and `python3 -m json.tool < .claude-plugin/plugin.json` to confirm manifests parse.
5. Keep `plugin.yaml` and `.claude-plugin/plugin.json` metadata in sync (name, version, description, license).

## Security

If you believe you have found a security issue, **do not** open a public issue. Contact the maintainers privately.
