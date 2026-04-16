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

![User unable to log in on WS-01](assets/WS01_user_unable_to_log_in.png)

---

## Step 1 — Ticket Creation

Since the user could not log into WS-01, the ticket was submitted from DC-01 on the user's behalf via the osTicket web portal.

**Ticket details submitted:**
- Name: Jasmine Rodgers
- Email: jasminerodgers@lab.com
- Help Topic: Report a Problem / Access Issue
- Summary: I can't log in to my account

![Ticket submission form](assets/helpdesk_technician_submits_user_ticket1.png)
![Ticket submission — details](assets/helpdesk_technician_submits_user_ticket2.png)
![Ticket created confirmation](assets/helpdesk_technician_ticket-visible-staff-panel.png)

---

## Step 2 — Ticket Intake and Triage

Ticket located in the osTicket Staff Control Panel under Open Tickets.

**Actions taken:**
- Ticket assigned to Ademola Durodola
- Priority adjusted from **High → Normal**

**Reason for downgrade:** Single user affected, no system-wide impact.

![Ticket assigned and viewed](assets/helpdesk_technician_user_ticket_assigned_and_viewed.png)

---

## Step 3 — Internal Analysis

An internal note was added before taking action to document analyst thinking and maintain an audit trail:

> "User unable to authenticate to WS01 using domain credentials. Multiple failed login attempts reported. Proceeding with the Active Directory password reset and enforcing the password change at the next logon."

![Ticket thread with internal note](assets/helpdesk_technician_user_ticketthread1.png)

---

## Step 4 — Remediation

On DC-01, opened **Active Directory Users and Computers (ADUC)**.

- Located user: Jasmine Rodgers (Finance → Users)
- Right-click → Reset Password
- Set a temporary password meeting domain complexity requirements
- Checked: **User must change password at next logon**
- Checked: **Unlock the user's account**

![Active Directory password reset](assets/helpdesk_technician_user_ticket_password_reset_completed.png)

---

## Step 5 — User Communication

A response was sent to the user via osTicket. The temporary password was **not** included in the ticket — communicated securely via phone.

> "Hello Jasmine, your password has been reset. Please try logging in using the temporary password provided. You will be prompted to create a new password after signing in. If you continue to experience any issues, please let us know. Thank you, IT Support"

An internal note was added to document the credential handling:

> "Temporary password communicated to user via secure method."

![User response sent](assets/helpdesk_technician_user_ticket_responded_to.png)

---

## Step 6 — Validation and Closure

User successfully logged into WS-01 using the temporary password and completed the forced password change. Confirmation received via phone call.

Internal note added:

> "User confirmed successful login and account access restored via phone call. No further issues reported."

Ticket status set to **Closed**.

![Ticket thread closed](assets/helpdesk_technician_user_ticketthread2.png)

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
