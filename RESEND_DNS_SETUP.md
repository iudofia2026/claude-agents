# Resend DNS Configuration for Non-Gmail Domains

## Overview

To send emails to **non-Gmail domains** (Yahoo, Outlook, corporate emails, etc.), you must verify your domain in Resend. Without domain verification, Resend operates in **sandbox mode**, which only allows sending to the account creator's email address.

## Quick Start: Verify Your Domain

### Step 1: Add Domain in Resend Dashboard

1. Go to [Resend Dashboard](https://resend.com/domains)
2. Click **"Add Domain"**
3. Enter your domain: `updates.lamcpainting.com` (or your preferred subdomain)
4. Resend will generate **unique DNS records** for your domain

### Step 2: Get Your DNS Records from Resend

**Important**: The DNS records below are examples. You must get the **actual records** from your Resend dashboard, as they are unique to your account.

In the Resend dashboard, you'll see records like:

#### DKIM Record
```
Type: TXT
Host: resend._domainkey.updates (or similar)
Value: [Unique key from Resend dashboard]
```

#### SPF Record
```
Type: TXT
Host: updates (or @ for root domain)
Value: v=spf1 include:amazonses.com ~all
```

#### MX Record (for bounce handling)
```
Type: MX
Host: send.updates (or similar)
Value: feedback-smtp.us-east-1.amazonses.com
Priority: 10
```

#### DMARC Record (Recommended)
```
Type: TXT
Host: _dmarc.updates (or _dmarc for root domain)
Value: v=DMARC1; p=none; rua=mailto:admin@lamcpainting.com
```

### Step 3: Add DNS Records to Your Domain Provider

1. Log in to your DNS provider (where `lamcpainting.com` is hosted)
2. Add each DNS record **exactly as shown** in the Resend dashboard
3. **Important**: 
   - For subdomains like `updates.lamcpainting.com`, use `updates` as the host
   - For root domain, use `@` or leave host blank
   - Wait 5-10 minutes after adding records

### Step 4: Verify Domain in Resend

1. Return to Resend dashboard → Domains
2. Click **"Verify"** next to your domain
3. Resend will check all DNS records
4. Status should change from **"Pending"** to **"Verified"** (green checkmark)

### Step 5: Update Environment Variables

Once verified, update your environment variables:

```bash
# Production (use your verified domain)
FROM_EMAIL="LAMC Painting <noreply@updates.lamcpainting.com>"
ADMIN_EMAIL="brian.dibassinga@yale.edu"

# OR multiple admin emails
ADMIN_EMAIL="brian.dibassinga@yale.edu,isiah.udofia@yale.edu"
```

**Important**: Make sure `FROM_EMAIL` uses the **verified domain** from Resend.

## Common Issues & Solutions

### ❌ "Emails only work for Gmail addresses"

**Problem**: You're in sandbox mode. Resend sandbox only allows sending to the account creator's email.

**Solution**: 
1. Verify your domain in Resend (follow steps above)
2. Ensure `FROM_EMAIL` uses your verified domain
3. Check domain status in Resend dashboard (must show "Verified")

### ❌ "Domain verification failed"

**Problem**: DNS records not properly configured.

**Solution**:
1. Double-check DNS records match **exactly** what Resend shows
2. Use DNS checker tools:
   - [MXToolbox](https://mxtoolbox.com/)
   - [DNS Checker](https://dnschecker.org/)
3. Wait 24-48 hours for DNS propagation
4. Verify each record individually in Resend dashboard

### ❌ "DNS records not found"

**Problem**: Records not propagated or incorrect host names.

**Solution**:
- For subdomain `updates.lamcpainting.com`:
  - DKIM: `resend._domainkey.updates`
  - SPF: `updates` (not `@`)
- For root domain `lamcpainting.com`:
  - DKIM: `resend._domainkey` or `resend._domainkey.@`
  - SPF: `@` or leave blank

### ❌ "Still using sandbox domain"

**Problem**: Environment variable still points to sandbox.

**Solution**:
1. Check `FROM_EMAIL` in your environment variables
2. Should be: `noreply@updates.lamcpainting.com` (your verified domain)
3. Should NOT be: `onboarding@resend.dev` (sandbox domain)
4. Redeploy your application after updating env vars

## Testing Domain Verification

Run the test script to check your domain status:

```bash
node test-resend-api.js
```

Look for:
- ✅ "Domain verification: [your-domain]" - Verified
- ❌ "Sandbox mode detected" - Still in sandbox
- ⏳ "Pending DNS verification" - DNS not propagated yet

## Production Checklist

- [ ] Domain added in Resend dashboard
- [ ] All DNS records added to domain provider
- [ ] DNS records verified in Resend (status = "Verified")
- [ ] `FROM_EMAIL` uses verified domain
- [ ] Test email sent to non-Gmail address (Yahoo, Outlook, etc.)
- [ ] Email received successfully

## Additional Resources

- [Resend Domain Verification Docs](https://resend.com/docs/dashboard/domains/introduction)
- [Resend DNS Records Guide](https://resend.com/docs/dashboard/domains/dns-records)
- [Email Deliverability Best Practices](https://resend.com/docs/dashboard/domains/deliverability)

---

**Note**: Once your domain is verified, you can send emails to **any email provider** (Gmail, Yahoo, Outlook, corporate emails, etc.). Sandbox mode is only for testing and only works with the account creator's email.