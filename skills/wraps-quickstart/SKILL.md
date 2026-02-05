---
name: wraps-quickstart
description: Get started with Wraps email infrastructure. Guides through deploying infrastructure, verifying a domain, installing the SDK, and sending the first email. Adapts to project framework and hosting provider.
---

# Wraps Quick Start

Zero-to-sending setup guide for Wraps email infrastructure. Detects project context and adapts instructions to the user's framework and hosting provider.

## How to Use This Skill

Walk the user through the full setup flow. Don't dump everything at once. Progress step by step, adapting to what you detect in their project.

## Step 1: Detect Project Context

Before starting, understand the project:

```bash
# Framework detection
cat package.json | grep -E '"(next|express|fastify|remix|astro|hono|nuxt)"'

# Hosting detection
ls vercel.json railway.json Dockerfile fly.toml render.yaml 2>/dev/null

# Existing email setup
cat package.json | grep -E '"(nodemailer|@sendgrid|resend|postmark|@aws-sdk/client-ses)"'

# AWS credentials
aws sts get-caller-identity
```

Adapt guidance based on findings:
- **Next.js + Vercel**: Recommend OIDC authentication (zero credentials in env vars)
- **Express/Fastify/Hono**: Standard IAM role or environment variable auth
- **Existing email provider**: Offer migration guidance
- **No AWS credentials**: Guide through `aws configure` first

## Step 2: Prerequisites

### AWS Account

An AWS account with credentials configured. Verify:

```bash
aws sts get-caller-identity
```

If this fails, the user needs to configure AWS credentials:
- **Option A**: `aws configure` with access key/secret
- **Option B**: AWS SSO with `aws sso login`
- **Option C**: Environment variables `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`

### Node.js

Node.js 20 or higher:

```bash
node --version
```

## Step 3: Deploy Infrastructure

```bash
npx @wraps.dev/cli@latest email init
```

The command prompts for:
- **Hosting provider**: Vercel (recommended for OIDC), AWS, Railway, or other
- **AWS region**: us-east-1 recommended for best SES deliverability features
- **Domain**: The domain emails will be sent from (e.g., yourapp.com)
- **Feature preset**: Start with **Production** (~$2-5/mo) for most apps

What gets deployed to the user's AWS account:
- SES configuration set with event tracking
- IAM role with least-privilege permissions
- DynamoDB table for email event history
- Lambda function for event processing
- EventBridge rules for SES events
- SQS dead letter queue

**For Vercel users**: OIDC provider is set up automatically. No AWS credentials need to be stored as environment variables.

## Step 4: Verify Domain

After `email init`, the CLI outputs DNS records to add. Three types:

### DKIM Records (Required)

Add 3 CNAME records provided by the CLI. Example format:
```
selector1._domainkey.yourapp.com CNAME selector1.dkim.amazonses.com
selector2._domainkey.yourapp.com CNAME selector2.dkim.amazonses.com
selector3._domainkey.yourapp.com CNAME selector3.dkim.amazonses.com
```

### DMARC Record (Recommended)

```
_dmarc.yourapp.com TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@yourapp.com"
```

### Verify DNS Propagation

DNS propagation takes 5-60 minutes. Check with:

```bash
npx @wraps.dev/cli@latest email verify --domain yourapp.com
```

Or check DKIM tokens directly:

```bash
npx @wraps.dev/cli@latest email domains get-dkim --domain yourapp.com
```

## Step 5: Install SDK

```bash
# npm
npm install @wraps.dev/email

# pnpm
pnpm add @wraps.dev/email

# yarn
yarn add @wraps.dev/email
```

## Step 6: Send First Email

### Next.js (App Router) - Server Action

```typescript
'use server';

import { WrapsEmail } from '@wraps.dev/email';

const email = new WrapsEmail();

export async function sendWelcomeEmail(to: string, name: string) {
  const result = await email.send({
    from: 'hello@yourapp.com',
    to,
    subject: `Welcome, ${name}!`,
    html: `<h1>Welcome to our app, ${name}!</h1><p>We're glad you're here.</p>`,
  });

  if (!result.success) {
    throw new Error(`Failed to send: ${result.error.message}`);
  }

  return result.data.messageId;
}
```

### Next.js (App Router) - API Route

```typescript
import { WrapsEmail } from '@wraps.dev/email';
import { NextResponse } from 'next/server';

const email = new WrapsEmail();

export async function POST(request: Request) {
  const { to, name } = await request.json();

  const result = await email.send({
    from: 'hello@yourapp.com',
    to,
    subject: `Welcome, ${name}!`,
    html: `<h1>Welcome, ${name}!</h1>`,
  });

  if (!result.success) {
    return NextResponse.json({ error: result.error.message }, { status: 500 });
  }

  return NextResponse.json({ messageId: result.data.messageId });
}
```

### Express / Fastify / Hono

```typescript
import { WrapsEmail } from '@wraps.dev/email';

const email = new WrapsEmail();

app.post('/api/send-email', async (req, res) => {
  const { to, name } = req.body;

  const result = await email.send({
    from: 'hello@yourapp.com',
    to,
    subject: `Welcome, ${name}!`,
    html: `<h1>Welcome, ${name}!</h1>`,
  });

  if (!result.success) {
    return res.status(500).json({ error: result.error.message });
  }

  res.json({ messageId: result.data.messageId });
});
```

### Standalone Script

```typescript
import { WrapsEmail } from '@wraps.dev/email';

const email = new WrapsEmail();

const result = await email.send({
  from: 'hello@yourapp.com',
  to: 'test@example.com',
  subject: 'Test from Wraps',
  html: '<h1>It works!</h1>',
});

console.log(result.success ? `Sent: ${result.data.messageId}` : `Error: ${result.error.message}`);
```

### Vercel OIDC Authentication

For Vercel deployments, the SDK automatically uses OIDC. No environment variables needed:

```typescript
const email = new WrapsEmail(); // Auto-detects Vercel OIDC
```

For non-Vercel environments, configure credentials:

```typescript
const email = new WrapsEmail({
  region: 'us-east-1',
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});
```

## Step 7: Verify It Worked

1. Check the recipient's inbox (and spam folder)
2. Check infrastructure status:

```bash
npx @wraps.dev/cli@latest email status
```

3. Check SES console for delivery events:

```bash
aws sesv2 get-account --region us-east-1 --query 'SendQuota'
```

## Step 8: Next Steps

Based on the project context, suggest relevant next steps:

**For SaaS apps:**
- Set up transactional templates (welcome, password reset, invoice)
- Configure React Email templates with `@wraps.dev/email` template API
- Add bounce/complaint handling with Wraps event tracking

**For marketing / newsletters:**
- Set up contacts and segments via the API or dashboard
- Configure topics for subscription management
- Create a preference center for subscribers
- Follow IP warming schedule before large sends

**For AI-powered products:**
- Set up named API keys for each agent
- Configure CloudWatch alarms for anomalous sending patterns
- Use event tracking to create agent feedback loops

**For all projects:**
- Visit the Wraps dashboard for analytics and monitoring
- Set up CloudWatch alarms for bounce/complaint rate thresholds
- Add custom MAIL FROM domain for better DMARC alignment

## Common Issues During Setup

### SES Sandbox Mode
New AWS accounts start in sandbox (200 emails/day, verified recipients only). Request production access:
1. AWS Console → SES → Account Dashboard
2. Click "Request Production Access"
3. Describe your use case and volume expectations

### DNS Propagation Delays
DKIM records can take up to 72 hours in rare cases (usually 5-60 minutes). Use `wraps email verify` to check status.

### Vercel OIDC Requires Team Account
Vercel OIDC authentication requires a Vercel team account (not personal). For personal accounts, use environment variable credentials instead.

### AWS Region Considerations
- **us-east-1**: Most SES features, best for North American audiences
- **eu-west-1**: GDPR-friendly, best for European audiences
- **ap-southeast-1**: Best for Asia-Pacific audiences

## Migrating from Other Providers

### From Resend

```typescript
// Before (Resend)
import { Resend } from 'resend';
const resend = new Resend('re_xxx');
await resend.emails.send({ from: '...', to: '...', subject: '...', html: '...' });

// After (Wraps)
import { WrapsEmail } from '@wraps.dev/email';
const email = new WrapsEmail();
await email.send({ from: '...', to: '...', subject: '...', html: '...' });
```

Key differences: No API key needed (uses AWS IAM). Infrastructure in YOUR AWS account.

### From SendGrid

```typescript
// Before (SendGrid)
import sgMail from '@sendgrid/mail';
sgMail.setApiKey(process.env.SENDGRID_API_KEY!);
await sgMail.send({ from: '...', to: '...', subject: '...', html: '...' });

// After (Wraps)
import { WrapsEmail } from '@wraps.dev/email';
const email = new WrapsEmail();
await email.send({ from: '...', to: '...', subject: '...', html: '...' });
```

### From Nodemailer + SES

```typescript
// Before (Nodemailer)
import nodemailer from 'nodemailer';
const transport = nodemailer.createTransport({ SES: new AWS.SES() });
await transport.sendMail({ from: '...', to: '...', subject: '...', html: '...' });

// After (Wraps)
import { WrapsEmail } from '@wraps.dev/email';
const email = new WrapsEmail();
await email.send({ from: '...', to: '...', subject: '...', html: '...' });
```

Wraps adds: event tracking, analytics dashboard, contact management, templates, and workflows on top of the same SES infrastructure.

### DNS Changes When Migrating

When switching from another provider:
1. Keep old provider's DNS records active during transition
2. Add Wraps DKIM CNAME records
3. Update SPF to include `amazonses.com` (if not already present)
4. Test sending with Wraps
5. Remove old provider's DNS records once confirmed working
