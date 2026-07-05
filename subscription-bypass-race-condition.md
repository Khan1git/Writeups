# Business Logic Race Condition -- Subscription Bypass

> **Disclaimer**
>
> This write-up is based on a real vulnerability discovered during a
> professional security assessment. All identifying information has been
> removed to protect the confidentiality of the application and
> organization.

## Overview

During a security assessment of a restaurant management platform, I
identified a **business logic race condition** that allowed users on the
**Free Plan** to bypass subscription restrictions.

The application intended to allow only **one restaurant** per free
account. Premium plans increased this limit, making the restriction a
core part of the platform's licensing model.

By exploiting concurrent requests, I was able to create an **unlimited
number of restaurants** using a free account.

## The Intended Logic

``` text
If user already owns a restaurant
    Reject the request
Else
    Create restaurant
```

Under normal usage, this logic works correctly. The problem appeared
when multiple requests were processed simultaneously.

## Finding the Issue

Whenever I encounter restrictions such as:

-   One account per user
-   One free resource
-   One active subscription
-   Limited free-tier features

I consider whether the implementation safely handles concurrent
requests.

The vulnerable endpoint was:

``` http
POST /api/restaurant
```

Using **Burp Suite Repeater's parallel request feature**, I submitted
multiple identical requests simultaneously.

## What Happened?

Each request independently checked whether the account already owned a
restaurant.

Because every request performed the validation before any of the others
committed their changes, they all concluded that no restaurant existed
yet.

As a result, every request created a new restaurant.

## Root Cause

The validation ("Does this user already have a restaurant?") and the
database insertion ("Create restaurant") were executed separately
without an atomic transaction.

This created a classic **Time-of-Check to Time-of-Use (TOCTOU)** race
condition.

``` text
Request A -> Check -> No restaurant
Request B -> Check -> No restaurant
Request C -> Check -> No restaurant

Request A -> Create restaurant
Request B -> Create restaurant
Request C -> Create restaurant
```

## Impact

-   Subscription restriction bypass
-   Unlimited restaurant creation on the Free Plan
-   Access to premium functionality without payment
-   Potential financial loss
-   Business logic compromise

Every created restaurant functioned normally.

## Remediation

The issue was fixed by making the validation and creation process atomic
using proper database transactions. Once one request successfully
created the first restaurant, concurrent requests correctly detected the
existing restaurant and were rejected.

## Lessons Learned

Business logic vulnerabilities can have significant business impact even
without traditional injection or memory corruption issues.

Whenever an application enforces quotas, subscription limits, inventory
counts, or similar business rules, testing concurrent execution should
be part of the security assessment.

## Skills Demonstrated

-   Business Logic Testing
-   Race Condition Discovery
-   Concurrent Request Testing
-   Burp Suite Repeater
-   Backend Logic Analysis
-   Root Cause Analysis
-   Technical Communication
