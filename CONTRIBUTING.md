# Contributing

Thanks for helping improve the OpenFinance skills. This repo is markdown only: each skill is a folder in [`skills/`](./skills) with a `SKILL.md`.

## Good first contributions

- A skill sent your agent to the wrong route, or an example no longer matches the API's response. Open an issue with the prompt, the call the agent made and the error.
- A trigger phrase that should load a skill but doesn't. Suggest it in the skill's `description`.
- Unclear wording. Small edits are welcome as pull requests.

## Rules for changes

1. Keep every write behind a quote and an explicit "confirm" from the user. Read [SECURITY.md](./SECURITY.md) first.
2. Never add code, scripts, binaries or keys. Skills stay markdown.
3. Funds only move between the user's own wallets. Don't add examples that send to other addresses.
4. Check facts against [docs.openfinance.tech](https://docs.openfinance.tech) before changing routes, fees or limits.

## Testing a change

```bash
npx skills add ./ -y
```

Then ask your agent to do the task the skill covers, and check it quotes first and waits for "confirm".

## Security issues

Don't open a public issue for a way to move funds without confirmation. Report it privately with **Report a vulnerability** on this repository's [Security tab](../../security/advisories/new).
