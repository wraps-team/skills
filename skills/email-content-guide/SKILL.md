---
name: email-content-guide
description: Best practices for email content that avoids spam filters. Apply these rules when writing email copy, building templates, or debugging spam placement issues.
---

# Email Content Guide

Rules and constraints for writing email content that lands in the inbox. Apply these every time you generate email copy, build a template, or help debug spam placement.

## How to Use This Skill

These are CONSTRAINTS, not suggestions. Apply them automatically when writing any email content. Do not wait for the user to ask about deliverability.

## Subject Lines

- Under 50 characters (40 is ideal for mobile preview)
- No ALL CAPS words
- No excessive punctuation (!! or ???)
- No misleading Re: or Fwd: prefixes
- Be specific about what's inside the email
- Personalize when data is available (e.g., include first name or company)
- Front-load the important words (mobile truncates after ~35 chars)

### Spam Trigger Words to Avoid in Subjects

**Financial**: free, bonus, cash, earn, income, investment, profit, money, discount, save, lowest price, bargain, cheap
**Urgency**: act now, limited time, expires, hurry, don't miss, last chance, only today, deadline, while supplies last
**Claims**: guaranteed, no obligation, risk-free, 100%, promise, winner, selected, congratulations, you've been chosen
**Pressure**: order now, buy now, click here, call now, apply now, sign up free, subscribe now
**Symbols**: $$$, excessive exclamation marks (!!!), all caps words, emoji overuse in subject

## Preheader Text

- Always include preheader/preview text (40-130 characters)
- Complement the subject line, do not repeat it
- If left empty, email clients show the first line of body text (often looks broken)
- Use it to add context that helps the reader decide to open

## Email Body

### Structure

- **Single primary CTA**: One clear action per email. Secondary links are fine in navigation, but the main message drives one action.
- **Inverted pyramid**: Most important content first, supporting details below, CTA at the bottom.
- **Scannable**: Use headings, short paragraphs (2-3 sentences), bullet points for lists.
- **Width**: 600px max for content area. Mobile-first.

### Content Rules

- **Plain text version**: ALWAYS include alongside HTML. HTML-only emails are flagged by some filters.
- **Text-to-image ratio**: At least 60% text, maximum 40% images. Image-only emails are heavily penalized.
- **No URL shorteners**: Never use bit.ly, tinyurl, t.co, etc. Use full domain links. Shorteners are associated with phishing.
- **Link count**: Maximum 5-7 links total (including footer). High link count looks spammy.
- **No JavaScript**: Stripped by all email clients. Never include script tags.
- **Images**: Always include alt text. Host on your domain or a CDN you control. Don't embed base64 images.
- **No invisible text**: White text on white background is a spam signal.

### Personalization

- Use real data: `{{firstName}}`, `{{companyName}}`, `{{planName}}`
- Fallback values: Always provide defaults for missing data (`{{firstName | "there"}}`)
- Don't over-personalize: Using too many personal data points in one email feels invasive

## Unsubscribe (Required for Marketing)

Every marketing or promotional email MUST include:

1. **Visible unsubscribe link** in the email body (typically in footer)
2. **List-Unsubscribe header** (RFC 8058):
   ```
   List-Unsubscribe: <mailto:unsubscribe@yourapp.com>, <https://yourapp.com/unsubscribe?id={{contactId}}>
   List-Unsubscribe-Post: List-Unsubscribe=One-Click
   ```
3. **One-click functionality**: Unsubscribe MUST work without requiring login
4. **Physical mailing address** (CAN-SPAM requirement)

Transactional emails (password reset, order confirmation, etc.) do NOT require unsubscribe but MUST NOT contain promotional content.

## Content by Email Type

### Transactional

Password resets, order confirmations, welcome emails, account notifications.

- Send immediately (no delay, no batching)
- Minimal, functional design (not marketing-style)
- No promotional content (mixing promo into transactional emails degrades sender reputation)
- Clear action: What does the user need to do?
- Include context: What triggered this email (e.g., "You requested a password reset")
- Keep subject line factual: "Your password reset link" not "Don't miss your new password!"

### Marketing

Newsletters, promotions, announcements, product updates.

- Include unsubscribe + email preferences link
- Segment by engagement (don't email users who haven't opened in 90+ days)
- Respect frequency expectations set during signup
- Physical mailing address in footer
- Clear sender identity (From name should be recognizable)
- Subject should set accurate expectations for content

### Automated / Lifecycle

Onboarding sequences, re-engagement, abandoned cart, renewal reminders.

- Reference the trigger event: "You signed up 3 days ago" or "Your trial ends in 7 days"
- Timely: Send at appropriate intervals (don't send 5 onboarding emails in one day)
- Progressive: Each email in a sequence should add new value, not repeat
- Exit conditions: Stop the sequence when the goal is achieved
- Personalize with real behavior data, not generic placeholders

## HTML Email Best Practices

### Layout

- Table-based layouts (flexbox and CSS grid do not work reliably in Outlook, Gmail, or Yahoo)
- Inline CSS or use a framework like React Email that handles inlining
- `<td>` for layout cells, `<table>` for structure
- Width attributes on tables: `width="600"` for outer container

### Compatibility

- Test in: Gmail (web + mobile), Outlook (desktop + web), Apple Mail, Yahoo Mail
- Dark mode: Include `<meta name="color-scheme" content="light dark">` and test dark mode rendering
- Font stacks: System fonts or web-safe only. Custom web fonts don't work in most email clients:
  ```css
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Helvetica, Arial, sans-serif;
  ```

### Responsive

- Media queries for mobile (not all clients support them, so design mobile-first)
- Minimum tap target: 44x44px for buttons and links on mobile
- Single-column layout for widths under 480px

## Accessibility

- Minimum 14px font size for body text (16px preferred)
- Color contrast ratio: 4.5:1 minimum for text
- Semantic HTML: Use `<h1>`-`<h3>`, `<p>`, `<ul>/<ol>` appropriately
- Alt text on every image (describe the content, not "image" or "photo")
- Don't convey information only through color (add text labels)
- Descriptive link text: "Read the full report" not "Click here"
- Language attribute: `<html lang="en">` on the root element
- Role attribute: `<table role="presentation">` for layout tables

## Compliance Quick Reference

### CAN-SPAM (US)

1. Accurate From/Reply-To headers
2. Non-deceptive subject lines
3. Physical postal address
4. Working unsubscribe mechanism
5. Honor opt-outs within 10 business days

### GDPR (EU)

1. Explicit opt-in consent for marketing
2. Right to access data on request
3. Right to erasure on request
4. Record of consent
5. Lawful basis documented

### CASL (Canada)

1. Express consent required (written/verbal)
2. Implied consent limited to existing relationships
3. Honor unsubscribe within 10 days
4. Clear sender identification

## Pre-Send Checklist

Apply before sending or generating any email:

- [ ] Subject line under 50 chars, no spam trigger words, no ALL CAPS
- [ ] Preheader text included and complements subject
- [ ] Plain text version present alongside HTML
- [ ] Single clear primary CTA
- [ ] Links use full URLs (no shorteners), 7 or fewer total
- [ ] Images have alt text, hosted on your domain
- [ ] Unsubscribe link visible and working (marketing emails)
- [ ] Physical address included (marketing emails)
- [ ] List-Unsubscribe header set (marketing emails)
- [ ] Mobile preview looks good at 320px width
- [ ] Dark mode doesn't break layout or hide content
- [ ] No spam trigger words in subject or body
- [ ] Text-to-image ratio is at least 60/40
