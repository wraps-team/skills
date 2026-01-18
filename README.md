# Wraps.dev Skills

Agent skills for building with [Wraps.dev](https://wraps.dev) — modern email and SMS infrastructure for AWS.

## Installation

```bash
npx add-skill wraps-team/skills
```

Or add individual skills:

```bash
npx add-skill wraps-team/skills/wraps-email
npx add-skill wraps-team/skills/wraps-sms
npx add-skill wraps-team/skills/wraps-cli
npx add-skill wraps-team/skills/aws-ses-best-practices
```

## Available Skills

| Skill | Description |
|-------|-------------|
| [wraps-email](./skills/wraps-email) | TypeScript SDK for sending emails via AWS SES |
| [wraps-sms](./skills/wraps-sms) | TypeScript SDK for sending SMS via AWS End User Messaging |
| [wraps-cli](./skills/wraps-cli) | CLI for deploying email/SMS infrastructure to your AWS account |
| [aws-ses-best-practices](./skills/aws-ses-best-practices) | Deliverability, warming, compliance, and AWS CLI diagnostics |

## What are Skills?

Skills are structured instructions that help AI coding agents work more effectively with specific libraries and frameworks. They provide:

- **Accurate patterns** — Real code examples from the actual SDK
- **Best practices** — Patterns that prevent common mistakes
- **Current documentation** — Always up-to-date with the latest API

## Usage with Claude Code

Once installed, skills are automatically available to Claude Code. The agent will use them when working on relevant tasks.

Example prompts:
- "Send a welcome email using Wraps"
- "Set up email infrastructure with the Wraps CLI"
- "Send an OTP via SMS"
- "Check my SES configuration with AWS CLI"
- "Help me improve email deliverability"

## Links

- [Wraps.dev](https://wraps.dev) — Main website
- [@wraps.dev/email](https://www.npmjs.com/package/@wraps.dev/email) — Email SDK
- [@wraps.dev/sms](https://www.npmjs.com/package/@wraps.dev/sms) — SMS SDK
- [@wraps.dev/cli](https://www.npmjs.com/package/@wraps.dev/cli) — CLI
- [GitHub](https://github.com/wraps-team/wraps) — Source code

## License

MIT
