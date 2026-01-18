# Wraps.dev Skills Repository

This repository contains agent skills for the Wraps.dev platform.

## Repository Structure

```
skills/
├── wraps-email/      # @wraps.dev/email SDK patterns
├── wraps-sms/        # @wraps.dev/sms SDK patterns
├── wraps-cli/        # @wraps.dev/cli deployment patterns
└── aws-ses-best-practices/  # Email deliverability guidelines
```

## Skill Format

Each skill follows the standard format:

- `SKILL.md` — Main instructions for the agent
- `references/` — Optional supporting documentation
- `examples/` — Optional code examples

## Contributing

1. Edit the relevant `SKILL.md` file
2. Test with Claude Code to verify accuracy
3. Submit a PR

## Related Repositories

- [wraps-team/wraps](https://github.com/wraps-team/wraps) — Main platform
- [wraps-team/wraps-js](https://github.com/wraps-team/wraps-js) — JavaScript/TypeScript SDKs
