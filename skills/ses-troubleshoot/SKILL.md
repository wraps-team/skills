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

Before running diagnostics, gather context:

```bash
# Get Wraps infrastructure overview
wraps email status

# Check AWS connectivity and identity
aws sts get-caller-identity
```

Find the user's domain and region from wraps config or `~/.wraps/connections/` metadata files.

---

## Bounce Diagnosis

### Check 1: Bounce rate from SES account

```bash
aws sesv2 get-account --region <region> \
  --query '{BounceRate: SendingEnabled, EnforcementStatus: EnforcementStatus}'
```

Full account details including reputation:

```bash
aws sesv2 get-account --region <region>
```

Look at the `SendQuota` and reputation fields. Bounce rate thresholds:

| Rate | Status | Action |
|------|--------|--------|
| < 2% | Healthy | Investigate individual cases |
| 2-5% | Warning | Clean list, check recent imports |
| 5-10% | Critical | Stop marketing sends, aggressive cleanup |
| > 10% | Danger | SES will suspend account |

### Check 2: Suppression list

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

### Check 3: DNS authentication issues causing bounces

```bash
wraps email check <domain>
```

This runs SPF, DKIM, DMARC, and blacklist checks. Grade must be B+ or higher.

### Check 4: Sending quota

```bash
aws sesv2 get-account --region <region> --query 'SendQuota'
```

Compare `SentLast24Hours` to `Max24HourSend`. If near the limit, sends are throttled or rejected.

### Bounce Remediation (ordered by impact)

1. **DNS issues** → Fix SPF/DKIM/DMARC records (see Spam Diagnosis for exact records)
2. **Suppression list bloated** → Clean suppression list + fix source data quality
3. **Bounce rate > 5%** → Pause marketing sends, run engagement-based list cleanup
4. **Sending quota hit** → Request limit increase in AWS Console → SES → Account Dashboard
5. **Bad import** → Validate email addresses before importing (use verification service)

---

## Spam Diagnosis

### Check 1: DNS authentication

```bash
wraps email check <domain>
```

Critical checks and what to look for:

**SPF**: Must include `amazonses.com`
```
v=spf1 include:amazonses.com ~all
```

**DKIM**: All 3 CNAME records must resolve
```bash
dig +short CNAME selector1._domainkey.<domain>
dig +short CNAME selector2._domainkey.<domain>
dig +short CNAME selector3._domainkey.<domain>
```

**DMARC**: Should be `quarantine` or `reject` (NOT `none`)
```bash
dig +short TXT _dmarc.<domain>
```

If DMARC is `p=none`, inbox providers may deprioritize emails. Upgrade to:
```
_dmarc.<domain> TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@<domain>"
```

**Blacklists**: Any listings are critical and must be addressed.

### Check 2: DMARC alignment

DMARC alignment fails when the From domain doesn't match the SPF or DKIM signing domain. Check:

- Is the envelope sender (MAIL FROM) the same domain as the header From?
- Custom MAIL FROM domain helps alignment:

```bash
# Check current MAIL FROM config
aws ses get-identity-mail-from-domain-attributes --identities <domain>
```

If not set, configure a custom MAIL FROM domain (e.g., `mail.<domain>`) with:
- MX record: `mail.<domain> MX 10 feedback-smtp.<region>.amazonses.com`
- SPF record: `mail.<domain> TXT "v=spf1 include:amazonses.com ~all"`

### Check 3: Domain and account reputation

```bash
aws sesv2 get-account --region <region>
```

Check `EnforcementStatus` (HEALTHY, PROBATION, SHUTDOWN) and the reputation metrics.

For new domains (< 30 days old), low reputation is expected. Follow IP warming schedule:
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

1. **Fix DNS** → Add/correct SPF, DKIM, and DMARC records
2. **Set up custom MAIL FROM domain** → Improves DMARC alignment
3. **Request blacklist delisting** → Use the removal URL for each listed blacklist
4. **Warm sending gradually** → Follow warming schedule for new/cold domains
5. **Clean content** → Remove spam triggers, add plain text version, reduce link count
6. **Add List-Unsubscribe header** → Required for marketing, improves inbox placement

---

## Send Failure Diagnosis

### Check 1: SES sandbox status

```bash
aws sesv2 get-account --region <region>
```

If `ProductionAccessEnabled` is `false`, the account is in sandbox mode:
- Limited to 200 emails/day
- Can only send to verified email addresses
- **Fix**: Request production access in AWS Console → SES → Account Dashboard

### Check 2: Sending quota

```bash
aws sesv2 get-account --region <region> --query 'SendQuota'
```

If `SentLast24Hours` equals or exceeds `Max24HourSend`, all sends are rejected until the 24-hour window rolls.

### Check 3: Domain verification

```bash
aws sesv2 get-email-identity --email-identity <domain> --region <region>
```

Check `VerifiedForSendingStatus`. If `false`, the domain isn't verified:
- Check DKIM CNAME records are added to DNS
- DNS propagation takes 5-60 minutes
- Verify with: `wraps email verify --domain <domain>`

### Check 4: IAM permissions

```bash
wraps email status
```

The IAM role must have `ses:SendEmail` and `ses:SendRawEmail` permissions. If using Vercel OIDC, verify the trust relationship is configured correctly.

### Check 5: Configuration set

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
3. **Domain not verified** → Add DKIM CNAME records, wait for propagation
4. **Permissions** → Run `wraps email init` to recreate/fix IAM role
5. **Config set missing** → Run `wraps email upgrade` to recreate

---

## Account Health Diagnosis

### Check 1: Account status

```bash
aws sesv2 get-account --region <region>
```

Key fields:
- `EnforcementStatus`: HEALTHY, PROBATION, or SHUTDOWN
- `ReviewDetails`: Any ongoing AWS review
- `ProductionAccessEnabled`: Whether production sending is allowed

### Check 2: Reputation metrics

From the same `get-account` response, check bounce and complaint rates:

| Metric | Healthy | Warning | Critical |
|--------|---------|---------|----------|
| Bounce Rate | < 2% | 2-5% | > 5% |
| Complaint Rate | < 0.1% | 0.1-0.3% | > 0.3% |

### Check 3: CloudWatch alarms

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

### Check 1: Complaint rate

```bash
aws sesv2 get-account --region <region>
```

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
aws sesv2 get-account --region <region> --query 'SendQuota.MaxSendRate'
```

If you're sending faster than `MaxSendRate` (emails per second), SES throttles delivery.

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
- Fix: Set DMARC to quarantine, add visible unsubscribe, clean complaint-sourced suppressions

**Pattern: New domain cold start**
- New domain + high volume immediately → Greylisting + spam placement → Low engagement → Poor reputation → More spam placement
- Fix: Follow warming schedule, start with engaged users, prioritize transactional emails

When reporting findings, always note if multiple issues are related and prioritize the root cause fix.
