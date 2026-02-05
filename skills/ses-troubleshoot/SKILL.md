---
name: ses-troubleshoot
description: Interactive SES deliverability troubleshooting. Diagnoses bounces, spam placement, send failures, account health, and complaint issues with targeted AWS CLI commands and ordered remediation steps.
---

# SES Deliverability Troubleshooter

Interactive diagnostician for AWS SES email delivery problems. When a user reports an email issue, identify the symptom, run targeted diagnostics, correlate findings, and provide ordered remediation.

## How to Use This Skill

You are an interactive diagnostician. Do NOT dump all sections at the user. Instead:

1. Identify the symptom (ask if unclear)
2. Run the diagnostic commands for that symptom
3. Interpret the results using the thresholds below
4. Correlate multiple findings (problems are often related)
5. Provide ordered remediation steps (most impactful first)

## Step 1: Identify the Symptom

Ask which problem they're experiencing, then follow the matching branch:

- Emails bouncing → [Bounce Diagnosis](#bounce-diagnosis)
- Emails going to spam/junk → [Spam Diagnosis](#spam-diagnosis)
- Emails not sending at all → [Send Failure Diagnosis](#send-failure-diagnosis)
- SES account under review/paused → [Account Health Diagnosis](#account-health-diagnosis)
- High complaint rate → [Complaint Diagnosis](#complaint-diagnosis)
- Delivery delays → [Delay Diagnosis](#delay-diagnosis)

## Step 2: Gather Context

Before running diagnostics, gather the user's domain and region from Wraps metadata:

```bash
# Read connection metadata to get domain, region, and config
cat ~/.wraps/connections/*.json
```

**Metadata JSON paths:**
- Region: `.region` (top-level)
- Domain: `.services.email.config.domain`
- Provider: `.provider`
- Account ID: `.accountId`
- Preset: `.services.email.preset`
- MAIL FROM subdomain: `.services.email.config.mailFromSubdomain`

Then check AWS connectivity:

```bash
aws sts get-caller-identity
```

---

## Primary Diagnostic Tool: `email check`

The Wraps CLI includes a comprehensive deliverability checker. **Always use `--json` for structured output** that's easy to interpret programmatically:

```bash
npx @wraps.dev/cli email check <domain> --json
```

This single command checks **all** of the following:
- **SPF**: Record, validity, DNS lookup count (max 10), `all` mechanism
- **DKIM**: Automatically finds correct SES token-based selectors (NOT `selector1/2/3`), key type and size
- **DMARC**: Policy, alignment, reporting configuration
- **MX Records**: Existence, resolution, redundancy
- **Mail Server TLS**: STARTTLS support, TLS version on each MX server
- **Reverse DNS**: PTR records for all MX IPs
- **IPv6**: MX AAAA records and SPF IPv6 coverage
- **Blacklists**: 12+ domain blacklists, 30+ IP blacklists per MX IP (165+ total checks)
- **Domain Age**: Registration date, registrar, RDAP data
- **DNSSEC, CAA, BIMI, MTA-STS, TLS-RPT**: Advanced security checks
- **Score**: 0-100 score with grade (A/B/C/D/F), deductions, and bonuses

**Key JSON paths in the response:**
- Overall: `.score.grade`, `.score.finalScore`, `.score.deductions[]`
- SPF: `.spf.exists`, `.spf.valid`, `.spf.record`, `.spf.lookupCount`, `.spf.allMechanism`
- DKIM: `.dkim.found`, `.dkim.selectors[].valid`, `.dkim.selectors[].keyBits`
- DMARC: `.dmarc.exists`, `.dmarc.valid`, `.dmarc.policy`, `.dmarc.reportingEnabled`
- MX: `.mx.exists`, `.mx.records[]`, `.mx.hasRedundancy`
- Blacklists: `.blacklist.overallClean`, `.blacklist.ipChecks.listed[]`, `.blacklist.domainChecks.listed[]`
- Domain age: `.domainAge.ageInDays`, `.domainAge.createdAt`

**Grade thresholds:**
- A (90-100): Excellent, all best practices followed
- B (75-89): Good, minor improvements possible
- C (60-74): Fair, notable issues to address
- D (40-59): Poor, significant problems
- F (0-39): Failing, critical issues

Use this command as the **first diagnostic step** for any symptom involving DNS, authentication, or reputation. It replaces the need for individual `dig`, `nslookup`, or manual DNS commands.

---

## SES Account Diagnostics

For account-level checks, run `get-account` **once** and interpret multiple fields:

```bash
aws sesv2 get-account --region <region>
```

**Key fields to inspect from the single response:**

| Field | What it tells you |
|-------|-------------------|
| `SendingEnabled` | Whether the account can send at all |
| `ProductionAccessEnabled` | `false` = sandbox mode (200/day, verified recipients only) |
| `EnforcementStatus` | HEALTHY, PROBATION, or SHUTDOWN |
| `Details.ReviewDetails.Status` | PENDING, GRANTED, DENIED, or FAILED |
| `SendQuota.Max24HourSend` | Daily sending limit |
| `SendQuota.SentLast24Hours` | Emails sent in rolling 24h window |
| `SendQuota.MaxSendRate` | Max emails per second |
| `SuppressionAttributes.SuppressedReasons` | What gets auto-suppressed |

**Important:** `get-account` does NOT return bounce/complaint rate percentages directly. Those are available in the SES console's Reputation Dashboard or via Virtual Deliverability Manager metrics. The thresholds below are for interpreting those rates when the user provides them or when checking the console.

---

## Bounce Diagnosis

### Check 1: Run email check

```bash
npx @wraps.dev/cli email check <domain> --json
```

Look at `.score.grade` and `.score.deductions[]` for DNS/authentication issues that could cause bounces.

### Check 2: SES account status

```bash
aws sesv2 get-account --region <region>
```

Check `EnforcementStatus` and compare `SentLast24Hours` to `Max24HourSend`. If near the limit, sends are throttled or rejected.

Bounce rate thresholds (from SES console Reputation Dashboard):

| Rate | Status | Action |
|------|--------|--------|
| < 2% | Healthy | Investigate individual cases |
| 2-5% | Warning | Clean list, check recent imports |
| 5-10% | Critical | Stop marketing sends, aggressive cleanup |
| > 10% | Danger | SES will suspend account |

### Check 3: Suppression list

```bash
# List all suppressed addresses
aws sesv2 list-suppressed-destinations --region <region>

# Filter by bounce vs complaint
aws sesv2 list-suppressed-destinations --region <region> --reasons BOUNCE
aws sesv2 list-suppressed-destinations --region <region> --reasons COMPLAINT
```

If suppression list is large, it often indicates a bad import or list hygiene issue. To remove a false positive:

```bash
aws sesv2 delete-suppressed-destination \
  --email-address user@example.com --region <region>
```

### Bounce Remediation (ordered by impact)

1. **DNS issues** → Fix SPF/DKIM/DMARC records based on `email check` findings
2. **Suppression list bloated** → Clean suppression list + fix source data quality
3. **Bounce rate > 5%** → Pause marketing sends, run engagement-based list cleanup
4. **Sending quota hit** → Request limit increase in AWS Console → SES → Account Dashboard
5. **Bad import** → Validate email addresses before importing (use verification service)

---

## Spam Diagnosis

### Check 1: Comprehensive email check

```bash
npx @wraps.dev/cli email check <domain> --json
```

Interpret the JSON response - key things to look for:

**SPF** (`.spf`): Must include `amazonses.com`. Check `.spf.allMechanism` — should be `-` (hard fail). `~` (soft fail) is acceptable but weaker.

**DKIM** (`.dkim`): At least one valid selector with 2048-bit RSA key. The CLI automatically checks the correct SES token-based selectors (e.g., `bsksdtrd66emmvw6frf33yopnumfs33r._domainkey`), NOT the generic `selector1/2/3` names.

**DMARC** (`.dmarc`): Policy should be `quarantine` or `reject` (NOT `none`). If `.dmarc.policy` is `none`, inbox providers may deprioritize emails. Recommended record:
```
_dmarc.<domain> TXT "v=DMARC1; p=reject; rua=mailto:dmarc@<domain>"
```

**Blacklists** (`.blacklist`): Check `.blacklist.overallClean`. If `false`, inspect `.blacklist.ipChecks.listed[]` and `.blacklist.domainChecks.listed[]`. Note the `priority` field — `high` priority listings are critical, `low` priority (like `cbl.anti-spam.org.cn`) are less impactful but should still be addressed.

**Domain Age** (`.domainAge.ageInDays`): Domains < 30 days old have inherently low reputation.

### Check 2: DMARC alignment and MAIL FROM

Check the MAIL FROM configuration from the SES identity:

```bash
aws sesv2 get-email-identity --email-identity <domain> --region <region>
```

Look at `MailFromAttributes`:
- `MailFromDomain`: Should be a subdomain like `mail.<domain>`
- `MailFromDomainStatus`: Should be `SUCCESS`

If MAIL FROM is not configured, set it up with these DNS records:
- MX record: `mail.<domain> MX 10 feedback-smtp.<region>.amazonses.com`
- SPF record: `mail.<domain> TXT "v=spf1 include:amazonses.com ~all"`

### Check 3: Domain and account reputation

```bash
aws sesv2 get-account --region <region>
```

Check `EnforcementStatus` (HEALTHY, PROBATION, SHUTDOWN).

For new domains (< 30 days old), low reputation is expected. Follow warming schedule:
- Days 1-2: 200 emails
- Days 3-7: 500-1,000
- Days 8-14: 5,000
- Days 15-30: 10,000-25,000

### Check 4: Content analysis

Ask the user for a recent email's HTML or subject line. Check for:

- **Spam trigger words**: FREE, WINNER, ACT NOW, LIMITED TIME, URGENT, $$
- **Image-to-text ratio**: Should be < 40% images
- **URL shorteners**: bit.ly, tinyurl are heavily penalized by spam filters
- **Link count**: More than 5-7 links looks spammy
- **Missing plain text**: HTML-only emails are flagged by some filters
- **Missing unsubscribe**: Required for marketing emails

### Spam Remediation (ordered by impact)

1. **Fix DNS** → Address all issues from `email check` deductions
2. **Set up custom MAIL FROM domain** → Improves DMARC alignment
3. **Request blacklist delisting** → Use the `delistUrl` from listed entries, or visit the blacklist's website
4. **Warm sending gradually** → Follow warming schedule for new/cold domains
5. **Clean content** → Remove spam triggers, add plain text version, reduce link count
6. **Add List-Unsubscribe header** → Required for marketing, improves inbox placement

---

## Send Failure Diagnosis

### Check 1: SES account status (sandbox, quota, enforcement)

```bash
aws sesv2 get-account --region <region>
```

Check all of these from the single response:
- `ProductionAccessEnabled`: If `false`, account is in sandbox mode (200 emails/day, verified recipients only). **Fix**: Request production access in AWS Console → SES → Account Dashboard.
- `SendQuota`: If `SentLast24Hours` equals or exceeds `Max24HourSend`, all sends are rejected until the 24-hour window rolls.
- `SendingEnabled`: If `false`, sending is disabled entirely.
- `EnforcementStatus`: If not HEALTHY, account may be restricted.

### Check 2: Domain verification

```bash
aws sesv2 get-email-identity --email-identity <domain> --region <region>
```

Check `VerifiedForSendingStatus`. If `false`, the domain isn't verified:
- Run `npx @wraps.dev/cli email check <domain> --json` to see DKIM status
- DNS propagation takes 5-60 minutes
- Verify with: `npx @wraps.dev/cli email verify --domain <domain>`

### Check 3: IAM permissions

```bash
npx @wraps.dev/cli email status
```

The IAM role must have `ses:SendEmail` and `ses:SendRawEmail` permissions. If using Vercel OIDC, verify the trust relationship is configured correctly.

### Check 4: Configuration set

```bash
aws sesv2 get-configuration-set \
  --configuration-set-name wraps-email-tracking --region <region>
```

If this returns a 404, the configuration set is missing. Check event destinations:

```bash
aws sesv2 get-configuration-set-event-destinations \
  --configuration-set-name wraps-email-tracking --region <region>
```

### Send Failure Remediation (ordered)

1. **Sandbox** → Request production access via AWS Console
2. **Quota hit** → Request limit increase, or spread sends over time
3. **Domain not verified** → Check DKIM records with `email check`, wait for propagation
4. **Permissions** → Run `npx @wraps.dev/cli email init` to recreate/fix IAM role
5. **Config set missing** → Run `npx @wraps.dev/cli email upgrade` to recreate

---

## Account Health Diagnosis

### Check 1: Account status

```bash
aws sesv2 get-account --region <region>
```

Key fields:
- `EnforcementStatus`: HEALTHY, PROBATION, or SHUTDOWN
- `Details.ReviewDetails`: Any ongoing AWS review
- `ProductionAccessEnabled`: Whether production sending is allowed

### Check 2: Reputation metrics

From the same `get-account` response, note the enforcement status. For actual bounce/complaint rate percentages, check the SES console Reputation Dashboard or Virtual Deliverability Manager.

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Bounce Rate | < 2% | 2-5% | > 5% |
| Complaint Rate | < 0.1% | 0.1-0.3% | > 0.3% |

### Check 3: Email check for DNS/reputation issues

```bash
npx @wraps.dev/cli email check <domain> --json
```

Check for blacklist listings, domain age, and authentication issues that could be contributing to account health problems.

### Check 4: CloudWatch alarms

```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix wraps --region <region>
```

Check if any alarms are in ALARM state.

### Account Health Remediation

**If PROBATION:**
1. Immediately reduce sending volume
2. Pause all marketing/bulk sends
3. Clean contact lists aggressively (remove anyone not engaged in 90 days)
4. Fix the root cause (bad import, spam content, missing unsubscribe)
5. Monitor daily until status returns to HEALTHY

**If SHUTDOWN:**
1. Appeal via AWS Support case
2. Document: what caused the issue, steps taken, prevention plan
3. Demonstrate compliance improvements before requesting reinstatement

---

## Complaint Diagnosis

### Check 1: Account status and complaint rate

```bash
aws sesv2 get-account --region <region>
```

Check `EnforcementStatus`. For actual complaint rate, check SES console Reputation Dashboard.

Complaint rate thresholds:

| Rate | Status | Action |
|------|--------|--------|
| < 0.1% | Healthy | Continue normally |
| 0.1-0.3% | Warning | Review content and targeting |
| 0.3-0.5% | Critical | Pause marketing emails |
| > 0.5% | Danger | SES may suspend account |

### Check 2: Suppression list (complaint-sourced)

```bash
aws sesv2 list-suppressed-destinations --region <region> --reasons COMPLAINT
```

High complaint suppression count indicates content or targeting problems.

### Check 3: Unsubscribe mechanism

Verify emails include:
- Visible unsubscribe link in email body
- `List-Unsubscribe` header (RFC 8058)
- `List-Unsubscribe-Post: List-Unsubscribe=One-Click` header
- One-click unsubscribe that works without login

### Complaint Remediation (ordered)

1. **Add 1-click unsubscribe** → Every marketing email must have List-Unsubscribe header
2. **Add preference center** → Let users choose frequency and topics
3. **Review content relevance** → Only send what users signed up for
4. **Segment by engagement** → Only send marketing to contacts who opened in last 90 days
5. **Verify consent** → Was the list properly opted-in? Double opt-in prevents complaints

---

## Delay Diagnosis

### Check 1: SES sending rate

```bash
aws sesv2 get-account --region <region>
```

Check `SendQuota.MaxSendRate`. If you're sending faster than this (emails per second), SES throttles delivery.

### Check 2: Delivery delay events

If Wraps event tracking is enabled, check for DELIVERY_DELAY events in the DynamoDB table. Common causes:
- Recipient mail server is throttling (greylisting)
- DNS resolution delays
- Recipient server temporarily unavailable

### Check 3: Sending pattern

Are sends happening in bursts? Large bursts trigger:
- SES throttling (rate limit enforcement)
- Recipient server greylisting (new sender suspicion)
- Temporary deferrals at high-volume recipients (Gmail, Outlook)

### Delay Remediation

1. **Bursting** → Spread sends over time with batch delays (1,000/batch, 60s between)
2. **Greylisting** → Expected for new domains, resolves as reputation builds
3. **Rate limit** → Request MaxSendRate increase via AWS Console
4. **Persistent delays to specific providers** → Check if that provider is rate-limiting your IP/domain

---

## Correlating Multiple Issues

Problems are often related. Common patterns:

**Pattern: Bad import cascade**
- Imported unvalidated email list → High bounce rate → SES suppression list grows → Account enters probation
- Fix: Clean list, remove suppressions from bad import, validate before importing

**Pattern: Missing authentication cascade**
- No DMARC or DMARC=none → Emails hit spam → Users don't see unsubscribe → They mark as spam → Complaint rate rises → Account reputation drops
- Fix: Set DMARC to quarantine/reject, add visible unsubscribe, clean complaint-sourced suppressions

**Pattern: New domain cold start**
- New domain + high volume immediately → Greylisting + spam placement → Low engagement → Poor reputation → More spam placement
- Fix: Follow warming schedule, start with engaged users, prioritize transactional emails

When reporting findings, always note if multiple issues are related and prioritize the root cause fix.
