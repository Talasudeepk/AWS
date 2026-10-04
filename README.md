# Amazon CloudFront Origin Access Control (OAC) for a Private S3 Bucket

> Serve website content to the world through **CloudFront**, while the **S3 bucket stays 100% private**.

| | |
|---|---|
| **Project ID** | 24CC3014-P106 |
| **Team** | S-62-MA (Team 388) |
| **Cloud** | AWS (Region: `us-east-1`, N. Virginia) |
| **Status** | Implemented and tested on a live AWS account |

## Team

| Name | Number |
|---|---|
| Neeli Kundana Veni | 2400030515 |
| T. Deepak Kumar Patro | 2400030499 |
| Grandhi Sunishka | 2400030480 |
| Snehasiva Naga Sree Vakkalagadda | 2400030452 |

---

## 1. The Idea in 30 Seconds

Imagine a **locked warehouse (S3)** and a **security desk (CloudFront)**.

- Visitors never get a key to the warehouse.
- They ask the security desk. The desk has the only key (**OAC**), fetches the file, and hands it over.
- If anyone goes straight to the warehouse door, they are turned away (**Access Denied**).

That is exactly what this project builds.

## 2. Problem Statement

Modern applications must deliver static content (web pages, images, videos, files) **quickly and securely**. Exposing an S3 bucket directly creates problems:

- Public S3 buckets can leak sensitive or important content.
- Direct access to S3 **bypasses central access control**.
- Serving straight from S3 misses the **performance benefits of a global CDN**.
- Stored data should be **encrypted at rest**.
- Organizations need secure access **without making the bucket public**.

## 3. Our Solution

A secure content delivery architecture using **Amazon CloudFront + a private Amazon S3 bucket**:

1. Store static content in a **private S3 bucket**.
2. Make **CloudFront the only public entry point**.
3. Use **Origin Access Control (OAC)** so only CloudFront can read the bucket.
4. Authenticate CloudFront to S3 with **AWS Signature Version 4 (SigV4)**.
5. Encrypt stored objects with **SSE-KMS** (data at rest).
6. Force **HTTPS** for viewers (data in transit).
7. Add **monitoring and auditing** with CloudFront metrics and CloudTrail.

## 4. Architecture

```
                    +---------------------+
                    |   Amazon Route 53   |  DNS (optional custom domain)
                    +----------+----------+
                               | resolves to CloudFront
                               v
+-----------------+   HTTPS   +--------------------+   OAC (SigV4)   +---------------------------+
| Users / Clients |---------->|  Amazon CloudFront |---------------->|   AWS Account             |
| Browsers,       |           |  CDN: edge caching |  signed request |   Amazon S3 (PRIVATE)     |
| Mobile, End     |           |  HTTPS only        |                 |   - Block Public Access   |
| users           |           +----------^---------+                 |   - Bucket policy: only   |
+-----------------+                      |                           |     this CloudFront       |
                              SSL/TLS certificate                  |   - SSE-KMS encryption    |
                          AWS Certificate Manager (ACM)            +---------------------------+

              AWS CloudTrail records account activity across all of the above
```

**Key features**

- OAC with SigV4 authentication
- Bucket remains private (no public access)
- Content delivered over HTTPS through CloudFront
- Custom domain supported with Route 53 and ACM
- Better security and control over who reaches the content

> Route 53 and ACM are part of the full reference architecture. The hands-on demo below uses the default `*.cloudfront.net` domain, which already provides HTTPS.

## 5. AWS Services Used

| Service | Role in this project |
|---|---|
| **Amazon S3** | Private storage for the website files |
| **Amazon CloudFront** | CDN and the only public entry point |
| **Origin Access Control (OAC)** | Lets CloudFront securely read the private bucket using SigV4 |
| **AWS KMS** | SSE-KMS encryption keys for objects at rest |
| **AWS CloudTrail** | Audit trail of who did what in the account |
| **Route 53 / ACM** | Custom domain and certificate (architecture design) |

## 6. Resources Created in the Demo

| Resource | Value |
|---|---|
| S3 bucket | `deepak-cloudfront-oac-demo-2026` (us-east-1) |
| Test file | `index.html` |
| CloudFront distribution | `Private-S3-OAC-Demo` |
| CloudFront domain | `d33lohyabeqhf4.cloudfront.net` |
| Viewer protocol policy | Redirect HTTP to HTTPS |
| Cache policy | `Managed-CachingOptimized` |
| Bucket encryption | SSE-KMS (Bucket Key enabled) |
| CloudTrail trail | `cloudfront-oac-security-trail` (multi-region, logging) |

---

## 7. Implementation Guide

The project was built in two phases and six steps.

### Phase 1: Secure Private Delivery

#### Step 1: Create a private S3 bucket
1. Open **S3 > Create bucket**.
2. Name it (ours: `deepak-cloudfront-oac-demo-2026`), region **us-east-1**.
3. Keep **Block all public access = ON**.
4. Upload `index.html`.

#### Step 2: Create a CloudFront distribution with OAC
1. Open **CloudFront > Create distribution**.
2. Choose the S3 bucket as the **origin**.
3. Under origin access, select **Origin access control** and create a new OAC (signing: *Sign requests*).
4. Create the distribution. CloudFront shows *"Successfully created new distribution"* and then deploys.

#### Step 3: Apply and check the bucket policy
CloudFront generates a policy that trusts only your distribution. Copy it into **S3 > Permissions > Bucket policy**:

```json
{
  "Version": "2008-10-17",
  "Id": "PolicyForCloudFrontPrivateContent",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": { "Service": "cloudfront.amazonaws.com" },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::deepak-cloudfront-oac-demo-2026/*",
      "Condition": {
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>"
        }
      }
    }
  ]
}
```

Why it works: the **Principal** is the CloudFront service, and the **SourceArn condition** limits access to *this one distribution*. Nobody else can read the bucket.

#### Test (the proof that OAC works)

| Test | URL | Result |
|---|---|---|
| Direct S3 URL | `http://deepak-cloudfront-oac-demo-2026.s3.us-east-1.amazonaws.com/index.html` | **AccessDenied** (XML error) |
| CloudFront URL | `http://d33lohyabeqhf4.cloudfront.net/index.html` | **Content delivered** |

Same file, only one allowed path. **Phase 1 completed.**

### Phase 2: Hardening and Visibility

#### Step 4: Enforce HTTPS
1. CloudFront > your distribution > **Behaviors** > edit the **Default (*)** behavior.
2. Set **Viewer protocol policy = Redirect HTTP to HTTPS**.
3. Save. The console shows *"The default cache behavior was updated."*
4. Use the secure link: `https://d33lohyabeqhf4.cloudfront.net/index.html`

#### Step 5: Encrypt with SSE-KMS (data at rest)
1. S3 > your bucket > **Properties > Default encryption > Edit**.
2. Choose **SSE-KMS** and select your AWS KMS key.
3. Keep **Bucket Key = Enabled** (reduces KMS request costs).
4. Save. The console shows *"Successfully edited default encryption."*

> **Important lesson:** with SSE-KMS, the **KMS key policy must allow the CloudFront service principal to decrypt**, otherwise requests fail with **403**. This works with OAC. The legacy OAI did not support SSE-KMS.
> Default encryption applies to **new** objects. Re-upload existing files to encrypt them.

#### Step 6: Monitor and audit
- **CloudFront monitoring:** CloudFront > **Monitoring** shows requests, data transfer, and error rates for the distribution.
- **CloudTrail:** create a trail (ours: `cloudfront-oac-security-trail`, multi-region, status *Logging*). **Event history** lists what happened, for example `StartLogging`, `PutEventSelectors`, `PutKeyPolicy`, `PutBucketEncryption`, `UpdateTrail`.

---

## 8. Final Checklist (All Verified)

- [x] Private S3 bucket with Block Public Access
- [x] CloudFront distribution
- [x] Origin Access Control (OAC)
- [x] SigV4 signing between CloudFront and S3
- [x] S3 bucket policy restricted to our CloudFront distribution
- [x] HTTPS redirect
- [x] SSE-KMS encryption for S3 objects
- [x] CloudFront monitoring (successful and error requests)
- [x] CloudTrail logging
- [x] CloudTrail SSE-KMS encryption
- [x] CloudTrail Event History showing AWS activity

## 9. Why OAC and Not the Old OAI?

| | Legacy OAI | **OAC (used here)** |
|---|---|---|
| SSE-KMS support | No | **Yes** |
| Request signing | Limited | **SigV4** |
| AWS recommendation | Legacy | **Current best practice** |

## 10. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 403 on the CloudFront URL | Bucket policy missing or wrong distribution ARN | Re-copy the policy CloudFront generated |
| 403 right after setup | Distribution still deploying or change not propagated | Wait a few minutes |
| 403 only on KMS-encrypted objects | KMS key policy does not allow CloudFront | Grant `kms:Decrypt` to the CloudFront service principal |
| Old content or errors still showing | Cached by CloudFront | Create an **invalidation** (`/*`) |
| Direct S3 URL works | Bucket is public | Turn **Block Public Access** back on |

## 11. Clean Up (Avoid Charges)

1. Disable and delete the CloudFront distribution.
2. Empty and delete the S3 bucket (and the CloudTrail log bucket).
3. Delete the CloudTrail trail.
4. Schedule deletion of the KMS key (customer-managed keys can incur monthly charges).

## 12. Demonstration Evidence

The project PDF contains console screenshots for every step: S3 bucket list, distribution created, bucket policy, AccessDenied on the direct S3 URL, behavior set to Redirect HTTP to HTTPS, default encryption with SSE-KMS, CloudFront monitoring, the CloudTrail trail, and CloudTrail Event History.

## 13. Key Takeaways

- Keep the origin **private**; let **CloudFront** be the only door.
- **OAC + SigV4** proves to S3 that a request really came from your distribution.
- **HTTPS** protects data in transit; **SSE-KMS** protects data at rest.
- **CloudTrail** gives an audit history of who did what.

---

*Project ID: 24CC3014-P106 | Team S-62-MA*
