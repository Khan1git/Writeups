## Privilege Escalation via Insecure Role Assignment in Signup Flow

### Overview

Recently during a penetration testing engagement on a social/content-based platform with dual onboarding flows (standard users and premium creators), I analyzed the authentication and registration logic focusing on role assignment and access control.

The application provided two distinct signup flows:

- Standard user registration flow
- Premium/creator onboarding flow (requiring identity verification, policy acceptance, and subscription/approval process)

At the UI level, both flows appeared properly separated and enforced.

---

### Observation

While intercepting the signup request, I observed that the registration payload included two role-related parameters:

- `type`
- `regtype`

These parameters were not exposed or controlled directly through the frontend for standard users, but were still being sent in the request body.

---

### Issue

The backend trusted these client-supplied parameters to determine the user role and onboarding path.

There was no proper server-side validation ensuring that:

- Standard users could not assign or escalate roles
- Premium/creator onboarding could not be bypassed
- Role assignment strictly followed backend-defined rules

---

### Exploitation

During testing, I intercepted a standard signup request and modified the role-related parameters before forwarding it to the server.

After submitting the modified request:

- Account creation succeeded
- The user was assigned elevated privileges equivalent to a premium/creator account
- The intended onboarding and verification flow was bypassed

This allowed bypassing the full creator onboarding process, including:

- Identity verification requirements  
- Policy acceptance flow  
- Subscription / approval-based activation  

---

### Impact

This issue leads to privilege escalation at the account creation stage.

An attacker can:

- Create premium/creator accounts without approval
- Bypass identity verification and onboarding requirements
- Access restricted dashboard features
- Abuse monetized or privileged functionality

---

### Root Cause

- Backend relies on client-controlled parameters (`type`, `regtype`) for role assignment
- Lack of server-side validation for allowed roles per signup flow
- No enforcement of onboarding integrity on the backend
- Trust placed on request payload instead of server-defined authorization logic

---

### Expected Behavior

- Role assignment should be handled exclusively on the server side
- Client-provided role parameters should be ignored or strictly validated
- Premium/creator access should only be granted after completing verified onboarding
- Signup flows should be enforced server-side, not only in the UI

---

### Recommendation

- Remove role-related parameters from client-controlled signup requests
- Enforce role assignment strictly on the backend
- Validate signup flow origin before assigning roles
- Implement server-side authorization checks for all privileged roles