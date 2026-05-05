# SVCRO Website — AWS Deployment Guide

Static site: `index.html` + `admin.html`, deployed to S3 + CloudFront.

---

## 1. Create the S3 Bucket

```bash
# Replace YOUR_BUCKET_NAME — must be globally unique (e.g. svcro-website-prod)
aws s3api create-bucket \
  --bucket YOUR_BUCKET_NAME \
  --region us-east-1
```

Enable static website hosting:

```bash
aws s3 website s3://YOUR_BUCKET_NAME/ \
  --index-document index.html \
  --error-document index.html
```

---

## 2. Set the Bucket Policy (Public Read)

Create `bucket-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR_BUCKET_NAME/*"
    }
  ]
}
```

Apply it:

```bash
aws s3api put-bucket-policy \
  --bucket YOUR_BUCKET_NAME \
  --policy file://bucket-policy.json
```

Also disable the "Block Public Access" setting in the S3 console:  
**S3 → Bucket → Permissions → Block public access → Edit → uncheck all → Save**

---

## 3. Upload Your Files

**Via AWS Console:**  
Drag and drop `index.html` and `admin.html` into the bucket via the S3 Console UI.

**Via AWS CLI:**

```bash
# Upload both HTML files
aws s3 cp index.html s3://YOUR_BUCKET_NAME/index.html --content-type "text/html"
aws s3 cp admin.html s3://YOUR_BUCKET_NAME/admin.html --content-type "text/html"

# If you add a /assets folder later:
aws s3 sync ./assets s3://YOUR_BUCKET_NAME/assets/ --cache-control "max-age=31536000"
```

Verify the site works at:  
`http://YOUR_BUCKET_NAME.s3-website-us-east-1.amazonaws.com`

---

## 4. Create a CloudFront Distribution

In the AWS Console → CloudFront → **Create distribution**:

| Setting | Value |
|---------|-------|
| Origin domain | `YOUR_BUCKET_NAME.s3-website-us-east-1.amazonaws.com` (the S3 **website** endpoint, NOT the REST endpoint) |
| Protocol | HTTP only (S3 website endpoint doesn't support HTTPS natively) |
| Default root object | `index.html` |
| Viewer protocol policy | **Redirect HTTP to HTTPS** |
| Cache policy | CachingOptimized (or create a custom one with 1-day TTL) |
| Price class | Use only North America and Europe (cheapest) |

**Via CLI (alternative):**

```bash
aws cloudfront create-distribution \
  --origin-domain-name YOUR_BUCKET_NAME.s3-website-us-east-1.amazonaws.com \
  --default-root-object index.html
```

Note your CloudFront domain: `d1234example.cloudfront.net`

---

## 5. Custom Domain via Route 53 + ACM

### 5a. Request an SSL Certificate (MUST be us-east-1)

```bash
aws acm request-certificate \
  --domain-name svcro.com \
  --subject-alternative-names "*.svcro.com" \
  --validation-method DNS \
  --region us-east-1
```

Complete DNS validation by adding the CNAME records ACM provides to your Route 53 hosted zone (ACM will show these in the console).

### 5b. Attach the Certificate to CloudFront

1. Edit your CloudFront distribution
2. Under **Alternate domain names (CNAMEs)** add: `svcro.com`, `www.svcro.com`
3. Under **Custom SSL certificate** select the ACM cert you just validated

### 5c. Create Route 53 DNS Records

In Route 53 → your hosted zone:

- **Record 1:** `svcro.com` → Type A → Alias → CloudFront distribution
- **Record 2:** `www.svcro.com` → Type A → Alias → same CloudFront distribution

DNS propagation typically takes 5–30 minutes.

---

## 6. Admin Panel Security

The admin panel is currently protected only by a client-side password check — **do not treat this as real security.** Options to harden it:

| Approach | Effort | Security |
|----------|--------|----------|
| **Obscure path** — rename `admin.html` to a random UUID like `admin-a7f3c9.html` | Low | Low (security through obscurity) |
| **CloudFront + Lambda@Edge** — check a header/cookie before serving the file | Medium | Medium |
| **Amazon Cognito** — full auth flow with user pool, hosted UI, JWT tokens | High | High |
| **Pre-signed S3 URL** — generate a time-limited URL server-side | Medium | Medium |

For a production launch, the **Cognito** option is recommended. It integrates directly with CloudFront via a Lambda@Edge authorizer.

---

## 7. Pushing Updates + Cache Invalidation

After editing `index.html` or `admin.html`, re-upload and invalidate the CDN cache:

```bash
# Upload updated files
aws s3 cp index.html s3://YOUR_BUCKET_NAME/index.html --content-type "text/html"
aws s3 cp admin.html s3://YOUR_BUCKET_NAME/admin.html --content-type "text/html"

# Invalidate CloudFront cache (replace E1EXAMPLE with your Distribution ID)
aws cloudfront create-invalidation \
  --distribution-id E1EXAMPLE \
  --paths "/*"
```

Invalidations are free for the first 1,000 paths/month, then $0.005 per path.

---

## 8. Path to Data Persistence

Currently all admin data lives in JavaScript arrays (in-memory only). To persist changes:

### Option A — S3 JSON File (Simple)
- Store product/collection data as `data.json` in the S3 bucket
- Admin reads it on load via `fetch('/data.json')`
- Admin writes via a Lambda function that calls `s3.putObject()`
- Pros: zero database costs, simple. Cons: no concurrent edit safety

### Option B — DynamoDB + Lambda + API Gateway (Scalable)

```
Admin Panel → API Gateway (REST) → Lambda → DynamoDB
```

1. Create DynamoDB tables: `Products`, `Collections`, `Press`
2. Write Lambda functions for CRUD operations (Node.js or Python)
3. Create API Gateway routes: `GET /products`, `POST /products`, `PUT /products/{id}`, `DELETE /products/{id}`
4. Replace the in-memory JS arrays in `admin.html` with `fetch()` calls to the API
5. Secure the API with an API key or Cognito authorizer

This is the recommended path for a real production store.

---

## 9. Cost Estimate

Assumes ~10,000 visitors/month, ~50 GB CloudFront data transfer.

| Service | Monthly Cost |
|---------|-------------|
| S3 storage (< 1 GB) | ~$0.02 |
| S3 requests | ~$0.05 |
| CloudFront data transfer (50 GB) | ~$0.85 |
| CloudFront requests (1M) | ~$0.01 |
| Route 53 hosted zone | $0.50 |
| ACM SSL certificate | Free |
| Lambda (if adding backend) | Free tier (1M req/mo) |
| DynamoDB (if adding backend) | Free tier (25 GB) |
| **Estimated Total** | **~$1.43/month** |

---

## 10. Launch Checklist

- [ ] S3 bucket created and public access configured
- [ ] `index.html` and `admin.html` uploaded to S3
- [ ] Static website hosting enabled on S3
- [ ] CloudFront distribution created with S3 website endpoint as origin
- [ ] HTTPS redirect configured in CloudFront
- [ ] ACM certificate requested in `us-east-1` and validated via DNS
- [ ] Certificate attached to CloudFront distribution
- [ ] Custom domain added as CloudFront alternate domain name
- [ ] Route 53 A records pointing to CloudFront (apex + www)
- [ ] Site loads correctly at `https://svcro.com`
- [ ] Admin panel accessible (consider obscuring the path before launch)
- [ ] CloudFront cache invalidation tested after a content update
- [ ] Google Analytics ID added to `index.html` `<head>` (replace placeholder in Settings panel)
- [ ] Klaviyo newsletter form wired up (replace `handleNewsletter()` stub in `index.html`)
- [ ] Social links in footer updated to real URLs
- [ ] Open Graph / Twitter meta tags added to `<head>` for social sharing
- [ ] Favicon created and linked

---

## Quick Reference

```bash
# Full deploy from scratch
BUCKET=svcro-website-prod
DIST_ID=E1EXAMPLE

aws s3 cp index.html s3://$BUCKET/ --content-type "text/html"
aws s3 cp admin.html s3://$BUCKET/ --content-type "text/html"
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```
