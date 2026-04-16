# T001 — Password Reset Request

## Ticket Details

| Field | Details |
|---|---|
| Ticket ID | #530759 |
| Date | 2026-04-16 |
| Submitted via | osTicket (web portal) |
| Priority | Normal (downgraded from High) |
| Status | Closed |
| Assigned To | Ademola Durodola |
| Environment | DC-01 (Windows Server 2022) · WS-01 (Windows 10 Pro) · INFRA-01 (osTicket) |

---

## User Report

User **Jasmine Rodgers** (`jasminerodgers@lab.com`) was unable to log into WS-01 using her domain credentials (`lab\jasmine`).

> "Hi, I am unable to sign in to my computer with my usual password. It says my credentials are incorrect. I may have entered it wrong a few times. Can someone help me reset my password so I can log back in?"

![User unable to log in on WS-01] <img width="1009" height="556" alt="image" src="https://github.com/user-attachments/assets/9b535621-b189-4e81-9ccf-eb912954f30f" />

---

## Step 1 — Ticket Creation

Since the user could not log into WS-01, the ticket was submitted from DC-01 on the user's behalf via the osTicket web portal.

**Ticket details submitted:**
- Name: Jasmine Rodgers
- Email: jasminerodgers@lab.com
- Help Topic: Report a Problem / Access Issue
- Summary: I can't log in to my account

![Ticket submission form] <img width="898" height="539" alt="image" src="https://github.com/user-attachments/assets/079194f2-2485-461e-a5b4-798a6ade90d0" />

![Ticket submission — details] <img width="865" height="537" alt="image" src="https://github.com/user-attachments/assets/4a7005e5-a4c0-4857-9cf9-07aaff885bbc" />

![Ticket created confirmation] <img width="897" height="344" alt="image" src="https://github.com/user-attachments/assets/8b3f5d7c-50c0-4375-b7fe-0d63bf47b766" />


---

## Step 2 — Ticket Intake and Triage

Ticket located in the osTicket Staff Control Panel under Open Tickets.

**Actions taken:**
- Ticket assigned to Ademola Durodola
- Priority adjusted from **High → Normal**

**Reason for downgrade:** Single user affected, no system-wide impact.

![Ticket assigned and viewed] <img width="1027" height="561" alt="image" src="https://github.com/user-attachments/assets/e94806e6-5b3e-416d-a338-ca17bf3035fa" />


## Step 3 — Internal Analysis

An internal note was added before taking action to document analyst thinking and maintain an audit trail:

> "User unable to authenticate to WS01 using domain credentials. Multiple failed login attempts reported. Proceeding with the Active Directory password reset and enforcing the password change at the next logon."

![Ticket thread with internal note]  <img width="1042" height="621" alt="image" src="https://github.com/user-attachments/assets/a0a7aab1-abf5-4444-8a7b-a56042f8fdb5" />

---

## Step 4 — Remediation

On DC-01, opened **Active Directory Users and Computers (ADUC)**.

- Located user: Jasmine Rodgers (Finance → Users)
- Right-click → Reset Password
- Set a temporary password meeting domain complexity requirements
- Checked: **User must change password at next logon**
- Checked: **Unlock the user's account**

![Active Directory password reset]  <img width="686" height="453" alt="image" src="https://github.com/user-attachments/assets/b146f408-cd7f-4b88-8d93-811a11ecfda9" />


---

## Step 5 — User Communication

A response was sent to the user via osTicket. The temporary password was **not** included in the ticket — communicated securely via phone.

> "Hello Jasmine, your password has been reset. Please try logging in using the temporary password provided. You will be prompted to create a new password after signing in. If you continue to experience any issues, please let us know. Thank you, IT Support"

An internal note was added to document the credential handling:

> "Temporary password communicated to user via secure method."

![User response sent]  <img width="1028" height="595" alt="image" src="https://github.com/user-attachments/assets/d969f100-7d2d-4fa0-b98b-4aa225be0ced" />

---

## Step 6 — Validation and Closure

User successfully logged into WS-01 using the temporary password and completed the forced password change. Confirmation received via phone call.

Internal note added:

> "User confirmed successful login and account access restored via phone call. No further issues reported."

Ticket status set to **Closed**.

![Ticket thread closed] <img width="1036" height="623" alt="image" src="https://github.com/user-attachments/assets/edb93634-b817-48fb-98db-df0d47339052" />

---

## Resolution Summary

Password reset via Active Directory Users and Computers on DC-01. User confirmed successful login on WS-01. Ticket closed after validation.

---

## Key Concepts Demonstrated

**Technical**
- Active Directory user management and password reset
- Account unlock via ADUC
- Domain authentication troubleshooting

**IT Support Workflow**
- Full ticket lifecycle: creation → triage → analysis → remediation → communication → validation → closure
- Priority adjustment with documented reasoning
- Internal notes maintained for audit trail

**Security Practices**
- Temporary password never stored in ticket or sent via email
- Secure credential delivery via phone call
- Forced password change at next logon enforced

---

## Notes

- Always communicate temporary passwords verbally — never via ticket or email
- Priority should reflect actual business impact, not initial user urgency
- Related script: [`Reset-UserPassword.ps1`](../scripts/Reset-UserPassword.ps1)
