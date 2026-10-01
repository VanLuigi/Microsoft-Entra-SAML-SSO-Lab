# Microsoft Entra ID SAML SSO Lab

This repository documents a hands-on lab that configures **Microsoft Entra ID** as a SAML Identity Provider (IdP) and the **Microsoft Entra SAML Toolkit** as a SAML Service Provider (SP). It demonstrates a working enterprise SSO flow, explains the core SAML concepts, and shows how to validate and troubleshoot the integration.

The goal is to show:
- How SAML-based SSO works in practice.
- How to configure both the IdP and SP sides.
- How to test an IdP-initiated login.
- How to inspect the SAML assertion.
- Why this matters for real-world security and identity integrations.

---

## Table of Contents

- [Objective](#objective)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [What Is SAML?](#what-is-saml)
- [IdP and SP Roles](#idp-and-sp-roles)
- [Lab Prerequisites](#lab-prerequisites)
- [Configuration Steps](#configuration-steps)
  - [Step 1 — Add the Toolkit App in Entra ID](#step-1--add-the-toolkit-app-in-entra-id)
  - [Step 2 — Get Placeholder SAML Values from Entra](#step-2--get-placeholder-saml-values-from-entra)
  - [Step 3 — Register and Configure the SP (Toolkit)](#step-3--register-and-configure-the-sp-toolkit)
  - [Step 4 — Update Entra with the Toolkit’s Real SP Values](#step-4--update-entra-with-the-toolkits-real-sp-values)
  - [Step 5 — Assign the Test User](#step-5--assign-the-test-user)
  - [Step 6 — Test the SSO Flow (IdP-Initiated)](#step-6--test-the-sso-flow-idp-initiated)
  - [Step 7 — Inspect the SAML Assertion](#step-7--inspect-the-saml-assertion)
- [Troubleshooting](#troubleshooting)
- [Lessons Learned](#lessons-learned)
- [Security Considerations](#security-considerations)
- [Business Value](#business-value)
- [Future Improvements](#future-improvements)

---

## Objective

Demonstrate how Microsoft Entra ID issues a signed SAML assertion to a service provider and how a user can log in once to the IdP and then access the application without re-entering credentials.

This lab focuses on an **IdP-initiated** flow using the Microsoft Access Panel (`myapps.microsoft.com`).

---

## Architecture

```text
User
  |
  v
Microsoft Entra ID
    Identity Provider (IdP)
  |
  | SAML Response (Assertion)
  v
Microsoft Entra SAML Toolkit
    Service Provider (SP)
```

- The **IdP** authenticates the user and issues a signed SAML assertion.
- The **SP** trusts the IdP, validates the assertion, and grants access based on the identity and claims inside it.

---

## Technologies Used

- **Microsoft Entra ID** (formerly Azure AD) as the Identity Provider.
- **Microsoft Entra SAML Toolkit** (gallery app) as the Service Provider.
- **SAML 2.0** for authentication and assertion exchange.
- **SAML-tracer** browser extension to inspect the SAML response.

---

## What Is SAML?

**SAML (Security Assertion Markup Language)** is an open, XML-based standard used to exchange authentication and authorization data between two parties: an **Identity Provider (IdP)** and a **Service Provider (SP)**.

In plain terms: SAML lets a user log in **once** with an IdP (like Microsoft Entra ID), and then get access to one or more applications (SPs) **without re-entering credentials**. This is the foundation of most enterprise **Single Sign-On (SSO)** setups.

The core mechanic is the **SAML assertion** — a digitally signed XML document created by the IdP after it authenticates the user. The SP receives this assertion, validates its signature and conditions (issuer, timestamps, audience), and then trusts the identity and attributes inside it to grant access.

There are two ways a SAML login can start:

- **SP-initiated**: the user tries to access the app directly, and the app redirects them to the IdP to log in.
- **IdP-initiated**: the user starts from the IdP’s app portal (e.g., `myapps.microsoft.com`) and is pushed to the SP already authenticated.

This lab uses the **IdP-initiated** flow.

---

## IdP and SP Roles

- **Identity Provider (IdP)**  
  - Authenticates the user.  
  - Issues the SAML assertion.  
  - Example: Microsoft Entra ID.

- **Service Provider (SP)**  
  - The application the user wants to access.  
  - Trusts the IdP’s assertion instead of authenticating the user itself.  
  - Example: Microsoft Entra SAML Toolkit.

Key SAML terms used in this lab:

| Acronym / Term    | Meaning                                                                                                                                         |
| ----------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| SAML              | XML-based open standard for exchanging authentication/authorization data between IdP and SP.                                                    |
| IdP               | The system that authenticates the user and issues the SAML assertion (e.g., Microsoft Entra ID).                                                |
| SP                | The application or service the user wants to access; trusts the IdP's assertion instead of authenticating the user itself (e.g., SAML Toolkit). |
| SSO               | The end-user experience of logging in once and accessing multiple SPs without re-authenticating.                                                |
| ACS URL           | The SP endpoint that receives and "consumes" the SAML response/assertion posted by the browser.                                                 |
| Entity ID         | A globally unique identifier for the IdP or SP; used to make sure the assertion is intended for the right SP.                                   |
| SAML Response     | Message generated by the IdP containing the assertion, sent back to the SP after authentication.                                                |
| SAML Assertion    | The actual signed XML statement about the user: who they are (authentication assertion) and their attributes (attribute assertion).             |
| NameID            | The value inside the assertion that uniquely identifies the user (e.g., UPN or email) to the SP.                                                |
| Certificate (Raw) | Public certificate from the IdP that the SP uses to verify the assertion's digital signature.                                                   |
| Login URL         | The IdP URL the SP redirects users to for authentication.                                                                                       |
| Logout URL        | The IdP URL used for single logout (SLO) so the session ends on both IdP and SP sides.                                                          |

---

## Lab Prerequisites

- An Azure/Entra tenant with Enterprise Application admin rights (a free trial tenant is sufficient).
- A **test user account** in Entra ID (do not use your own admin/global account for testing).
- A modern browser with the **SAML-tracer** extension installed (optional but recommended).
- Ability to create and configure an Enterprise Application in Entra ID.

> Security note: Do not use production tenants, real user accounts, or sensitive applications for lab work. Use isolated test tenants and test accounts only.

---

## Configuration Steps

### Step 1 — Add the Toolkit App in Entra ID

1. Go to the [Entra admin center](https://entra.microsoft.com).
2. Navigate to **Entra ID → Enterprise applications → New application**.
3. Search for **"Microsoft Entra SAML Toolkit"** and add it to your tenant.

> Note: Older documentation may refer to this as “Azure AD SAML Toolkit”. The current gallery app name is **Microsoft Entra SAML Toolkit**.

---

### Step 2 — Get Placeholder SAML Values from Entra

1. Open the app → **Single sign-on** → select **SAML**.
2. In **Basic SAML Configuration**, note the default values shown under **Patterns**:
   - **Entity ID**: `https://samltoolkit.azurewebsites.net`
   - **Assertion Consumer Service URL**: `https://samltoolkit.azurewebsites.net/SAML/Consume`
   - **Sign-on URL**: `https://samltoolkit.azurewebsites.net/`
3. Under **Section 4 (Set up Microsoft Entra SAML Toolkit)**, note:
   - **Login URL**
   - **Microsoft Entra Identifier**
   - **Logout URL**
4. Under **Section 3 (SAML Certificates)**, download the **Certificate (Raw)**.

These values will be used to configure the SP side (the toolkit website).

---

### Step 3 — Register and Configure the SP (Toolkit)

1. Go to [https://samltoolkit.azurewebsites.net/](https://samltoolkit.azurewebsites.net/).
2. Click the top-right corner to **register a new user**.
   - The username/email **must match** the email/UPN of the Entra test user you plan to sign in with (for example, `testuser@yourtenant.onmicrosoft.com`).
3. Log in with that new account.
4. Go to the **SAML Configuration** page in the top navigation.
5. Enter the values collected in Step 2:
   - Login URL
   - Microsoft Entra Identifier
   - Logout URL
   - Upload/paste the **Raw Certificate** from Entra.
6. Save.

The toolkit will now generate its own:

- **Entity ID**
- **SP-initiated Login URL**
- **Assertion Consumer Service (ACS) URL**

Save these values; you will use them to update Entra ID.

---

### Step 4 — Update Entra with the Toolkit’s Real SP Values

1. Return to the Entra enterprise app → **Single sign-on → SAML → Basic SAML Configuration**.
2. Click **Edit** and replace the placeholder values with the exact values generated by the toolkit:
   - **Identifier (Entity ID)** → toolkit’s Entity ID
   - **Reply URL (Assertion Consumer Service URL)** → toolkit’s ACS URL
   - **Sign-on URL** → toolkit’s SP-initiated SSO URL
3. Save.

At this point, Entra and the toolkit have each other’s correct endpoints and identifiers.

---

### Step 5 — Assign the Test User

1. In the enterprise app, go to **Users and groups**.
2. Click **Add user/group** and assign the same test user whose email matches the toolkit account created in Step 3.
3. Save.

Without this assignment, the user will not see the application in their Access Panel and cannot test the SSO flow.

---

### Step 6 — Test the SSO Flow (IdP-Initiated)

1. Open a new **InPrivate/Incognito browser window** (avoids session conflicts with your admin account).
2. Go to [https://myapps.microsoft.com/](https://myapps.microsoft.com/).
3. Sign in as the **test user** (not your admin account).
4. Find the **Microsoft Entra SAML Toolkit** tile and click it.
5. You will land on a **SAMLLogin** page showing a **Connection Identifier** number and a **Log in** button — this is normal. Click **Log in**.
6. You should now be inside the toolkit application, authenticated as the test user, with no separate password prompt on the toolkit side.

Successful login confirms that:

- Entra authenticated the user.
- Entra issued a SAML assertion.
- The toolkit accepted and validated the assertion.
- The toolkit created a session for the user.

---

### Step 7 — Inspect the SAML Assertion

The toolkit does **not** display a decoded assertion by default. To inspect the actual SAML response:

1. Install the **SAML-tracer** browser extension (available for major browsers).
2. Open the SAML-tracer panel **before** starting login (from `myapps.microsoft.com`, InPrivate window).
3. Repeat the login flow from Step 6.
4. In SAML-tracer, locate the POST request to the toolkit’s **ACS URL** — it carries the base64-encoded `SAMLResponse`.
5. Use SAML-tracer’s decode/XML view to inspect:
   - **Issuer** (should be the Entra Identifier)
   - **NameID** (should match your test user’s email/UPN)
   - **Conditions** (`NotBefore` / `NotOnOrAfter` — assertion validity window)
   - **AttributeStatement** (any claims configured under Entra’s Attributes & Claims)
   - **Signature** block (confirms the assertion is signed and untampered)

This step is valuable for understanding what the SP actually receives and trusts.

---

## Troubleshooting

Common issues and how to investigate them:

| Symptom | Likely cause | Investigation |
|---|---|---|
| User is redirected but not logged in | NameID or account mismatch | Compare the Entra claim with the SP username |
| Invalid audience error | Entity ID mismatch | Compare the assertion audience with the SP Entity ID |
| Reply URL error | Incorrect ACS URL | Compare Entra’s Reply URL with the SP-generated ACS URL |
| Signature validation failure | Wrong or expired certificate | Verify the IdP signing certificate |
| User cannot see the application | Missing assignment | Check Enterprise Application assignments |
| Assertion is rejected due to time | Clock or validity-window issue | Review `NotBefore` and `NotOnOrAfter` |

General tips:

- Always test with a **dedicated test user**, not an admin account.
- Ensure the **email/UPN** in Entra matches the toolkit account exactly.
- Use **InPrivate/Incognito** windows to avoid cached sessions.
- Use **SAML-tracer** to confirm that a SAML response is actually being sent and to inspect its contents.

---

## Lessons Learned

- The toolkit’s own instructions are outdated: they refer to an “Azure AD SAML Toolkit” gallery app, but the current app is named **Microsoft Entra SAML Toolkit** — same app, renamed.
- SAML configuration is **bidirectional**: Entra needs the toolkit’s SP values (Entity ID, ACS URL), and the toolkit needs Entra’s IdP values (Login URL, Identifier, Logout URL, Certificate). Missing either side causes a silent failure (redirect to homepage with no session).
- User identity must **match exactly** between the IdP (Entra) and the SP (toolkit) — the toolkit matches by email/username, so mismatched test accounts will break the flow even if the SAML config is otherwise correct.
- Testing via **Entra’s “Test sign in” button** is not the same as a full round trip in this case — the toolkit’s intended test path is through **myapps.microsoft.com** (IdP-initiated flow).
- The toolkit is a **pass/fail connectivity demo**, not a debugging tool. It confirms SSO works by simply logging you in — it does not render the SAML assertion. Use a browser extension like **SAML-tracer** if you want to see the actual XML assertion, signature, and claims.

---

## Security Considerations

- Use **test tenants and test accounts only**; do not experiment on production identity or application environments.
- Do not publish:
  - Tenant IDs.
  - Real user emails or UPNs.
  - Private keys or sensitive certificates.
  - Full SAML responses containing sensitive claims.
- Treat SAML assertions as security-sensitive objects; they can be used to impersonate users if mishandled.
- Ensure system clocks are synchronized; large time differences can cause assertion validation failures.

---

## Business Value

This lab illustrates capabilities that are directly relevant to enterprise security and identity work:

- **Secure access to SaaS applications** using centralized identity instead of separate passwords.
- **Reduced password fatigue and phishing risk** by minimizing the number of credentials users must manage.
- **Centralized control** over who can access which applications via user and group assignments.
- **Auditable authentication** through a trusted IdP rather than ad-hoc local accounts.
- **Foundation for more advanced identity patterns**, such as:
  - Conditional Access and MFA.
  - Attribute-based access control.
  - Automated provisioning (SCIM).
  - Integration with custom applications via SAML or OIDC.

For employers, this demonstrates:

- Practical SAML and SSO configuration skills.
- Ability to read vendor documentation, identify gaps, and adapt.
- Understanding of how identity integrations affect security and user experience.
- Capability to explain technical concepts to both technical and non-technical stakeholders.

---

## Future Improvements

Possible next steps to extend this lab:

- Add **MFA** in Entra and observe how it affects the SAML flow.
- Configure **custom claims** (e.g., group membership, roles) and consume them in the SP.
- Implement an **SP-initiated** flow and compare it with the IdP-initiated flow.
- Build a **custom SAML SP** (for example, a simple web app) and integrate it with Entra.
- Compare SAML with **OAuth 2.0** and **OpenID Connect** for modern application authentication.
- Document a **troubleshooting playbook** for SAML issues in enterprise environments.

---

## About This Repository

This repository is part of a personal learning journey in identity, access management, and cloud security. It is intended to:

- Demonstrate hands-on lab work.
- Share practical configuration steps and troubleshooting approaches.
- Explain why these technologies matter in real-world security architectures.

Feel free to reuse the structure and explanations for your own labs, and adapt them to your environment.
