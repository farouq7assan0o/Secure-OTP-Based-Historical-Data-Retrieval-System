# Secure OTP-Based Historical Data Retrieval System

**Author:** Farouq Hassan
**Assessment Type:** Secure Design Review + OWASP Mapping
**System Type:** OTP-Based Data Retrieval for Deleted Users
**Focus Areas:** Authentication Security, Access Control, S3 Security, OTP Hardening

---

# Part 1 — System Overview

📄 Source: 

## What Are We Building?

We are building a feature that allows **previously deleted users** to retrieve their historical order data by verifying ownership of their old email or phone number using an OTP (One-Time Password) verification flow.

---

## System Components

The architecture includes:

* Frontend
* Data Request API
* OTP Service
* Deleted Users Database
* Data Generation Service
* S3 Archival Storage
* Email/SMS Delivery Service

---

## System Flow (High-Level)

1. User enters old email/phone.
2. OTP is generated.
3. OTP sent via Email/SMS.
4. User verifies OTP.
5. Archived data fetched.
6. PDF generated.
7. PDF uploaded to S3.
8. Download link emailed to user.

The diagram shown at the bottom of page 1  visually represents this flow, connecting frontend → API → OTP → DB → PDF → S3 → Email delivery.

---

# Part 2 — Threat Modeling (What Can Go Wrong?)

📄 Source: 

Identified risks include:

### 1️⃣ OTP Brute Force

* Weak rate limiting
* Unlimited guessing attempts

### 2️⃣ User Enumeration

* Error messages reveal whether an email/phone exists

### 3️⃣ OTP Spam

* Attackers repeatedly request OTPs to spam victims

### 4️⃣ S3 Object Exposure

* Predictable filenames
* Misconfigured buckets
* Public object access

### 5️⃣ Poor Validation / Insecure Communication

* No TLS enforcement
* Weak database protections
* Unfiltered data responses

---

# Part 3 — Vulnerability Analysis (Detailed Findings)

📄 Source: 

Below are the vulnerabilities identified in the implementation.

---

# 🔴 Vulnerability 1 — OTP Exposed in API Response and Logs

## Description

The OTP is:

* Returned directly in the API response
* Printed in application logs

Anyone with:

* API access
* Log access

Can retrieve the OTP and bypass authentication.

---

## Affected Code

* `handle_otp_request()` in `data_request_api.py`
* `MessagingService` in `utilities.py`

(Referenced on page 1 )

---

## OWASP Mapping

**A05 — Security Misconfiguration**

---

## Fix Implemented

* Removed OTP from API responses
* Removed OTP from logs
* Only send OTP via Email/SMS

### Secure Example

```python
def handle_otp_request(identity):
    otp = generate_otp()
    store_hashed_otp(identity, otp)
    send_via_sms_or_email(identity, otp)

    return {"message": "OTP sent successfully"}
```

---

# 🔴 Vulnerability 2 — OTPs Never Expire

## Description

OTPs were stored without checking timestamps.

This allows:

* Reuse of old OTPs indefinitely

---

## Affected Code

* OTP stored in `utilities.py`
* No timestamp validation

(Referenced on page 1 )

---

## OWASP Mapping

**A07 — Authentication Failures**

---

## Fix Implemented

Added expiration logic (5–10 minutes).

### Secure Implementation Example

```python
from datetime import datetime, timedelta

def verify_otp(stored_time):
    if datetime.now() - stored_time > timedelta(minutes=5):
        return False
    return True
```

Also:

* Delete expired OTPs
* Use single-use tokens

---

# 🔴 Vulnerability 3 — No OTP Attempt Limit

## Description

Unlimited OTP guesses allowed.

Attackers can brute-force 6-digit codes:

```text
000000 → 999999
```

---

## Affected Code

* `handle_data_generation()` in `data_generation_service.py`

(Referenced on page 1 )

---

## OWASP Mapping

**A07 — Authentication Failures**

---

## Fix Implemented

* Added rate limiting
* Added attempt counters
* Block after N failures

### Example

```python
MAX_ATTEMPTS = 5

if user_attempts > MAX_ATTEMPTS:
    lock_account(identity)
```

Also implemented:

* CAPTCHA
* Exponential backoff
* IP-based throttling

---

# 🔴 Vulnerability 4 — Predictable S3 Object Paths

## Description

Files stored using predictable path:

```text
email/<identity>.pdf
```

If attacker knows email format:

```text
email/john@example.com.pdf
```

They can guess URLs and download other users' data.

---

## Affected Code

* `data_generation_service.py`
* S3 upload logic

(Referenced on page 1 )

---

## OWASP Mapping

**A01 — Broken Access Control**

---

## Fix Implemented

* Randomized filenames
* Private bucket configuration
* Presigned URLs with expiration

### Secure S3 Upload Example

```python
import uuid

filename = f"{uuid.uuid4()}.pdf"

s3_client.upload_file(
    file_path,
    "private-bucket",
    filename
)
```

### Generate Presigned URL

```python
url = s3_client.generate_presigned_url(
    'get_object',
    Params={'Bucket': bucket, 'Key': filename},
    ExpiresIn=600
)
```

---

# Part 4 — Security Improvements Implemented

📄 Source: 

### Authentication Controls

* Single-use OTP
* 5-minute expiration
* Hashed OTP storage
* Rate limiting
* CAPTCHA
* Uniform error messages

---

### S3 Hardening

* Private bucket
* Presigned URLs only
* No email/phone in object path
* Lifecycle rules for auto-deletion

---

### Infrastructure Hardening

* TLS enforced
* Strict IAM roles
* Centralized logging
* Monitoring & alerting
* Database access restrictions

---

# Part 5 — Verification & Testing

📄 Source: 

To verify security controls:

### ✔ Test OTP Expiry

* Attempt verification after 10 minutes → should fail

### ✔ Test Rate Limiting

* Attempt >5 incorrect OTPs → account blocked

### ✔ Test User Enumeration

* Ensure error messages are identical

### ✔ Test S3 Permissions

* Direct object URL → should fail
* Presigned URL after expiry → should fail

### ✔ Validate PDF Data Integrity

* Ensure only correct user data included

### ✔ Validate Lifecycle Policies

* Confirm automatic deletion of exported files

---

# Security Architecture Summary

| Area             | Improvement                        |
| ---------------- | ---------------------------------- |
| Authentication   | Expiring, hashed OTPs              |
| Access Control   | Private S3 + presigned URLs        |
| Abuse Prevention | Rate limiting + CAPTCHA            |
| Data Protection  | TLS + strict IAM                   |
| Monitoring       | Logging without sensitive exposure |

---

# Skills Demonstrated

* Secure system design
* Threat modeling
* OWASP Top 10 mapping
* Authentication hardening
* Cloud storage security
* Rate limiting implementation
* Secure logging practices
* S3 lifecycle configuration
* Defensive architecture review

---

# Final Outcome

This session demonstrates:

* Identification of authentication flaws
* Proper OTP lifecycle implementation
* Access control hardening
* Cloud object security configuration
* Secure design validation process
* OWASP-aligned remediation


Tell me how you'd like to present your work next.
