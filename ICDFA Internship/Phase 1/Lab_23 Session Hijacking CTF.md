# CTF Challenge - Session Hijacking Test on OWASP Juice Shop

---

## 1. Engagement Overview

A session management assessment was conducted against OWASP Juice Shop running
locally at `http://localhost:3000`. The assessment covered four phases: cookie
attribute inspection, session expiration testing via token replay, JWT payload
predictability analysis, and session fixation testing. Firefox Developer Tools
were used for cookie inspection and manual storage manipulation. Burp Suite was
used to intercept, save, and replay authenticated requests. The session token
was decoded using `jwt.io` to inspect payload contents. Testing confirmed that
while the application successfully resists session fixation, three session
management weaknesses were present and exploitable.

---

## 2. Objectives

- Inspect the session token cookie for the presence and configuration of the
  `HttpOnly`, `Secure`, and `SameSite` security attributes
- Confirm whether the server invalidates the session token after a user logs
  out, or whether it remains valid for replay attacks
- Decode the JWT to assess whether the payload contains sensitive information
  that could aid an attacker who intercepts the token
- Test for session fixation by injecting a pre-crafted token before login and
  observing whether the application adopts or replaces it
- Demonstrate account takeover by injecting a captured JWT into a private
  browsing session without credentials

---

## 3. Scope

**In-Scope:**
- OWASP Juice Shop at `http://localhost:3000`
- Session token cookie and JWT stored in browser storage
- Profile endpoint: `http://localhost:3000/profile`
- Cookie attributes: `HttpOnly`, `Secure`, `SameSite`
- JWT payload structure decoded via `jwt.io`

**Out-of-Scope:**
- SQL injection or XSS exploitation against Juice Shop
- Any systems outside the local lab environment
- Network-level man-in-the-middle interception against live traffic

**Authorization Statement:**
> This assessment was conducted against OWASP Juice Shop, an intentionally
> vulnerable application deployed in an isolated local lab environment for
> security training. No production systems or real user accounts were accessed
> at any point. All activities were performed for educational purposes under
> authorized course instruction.

---

## 4. Methodology

### Phase 1 -- Cookie Attribute Inspection
OWASP Juice Shop was accessed via Firefox at `http://localhost:3000`. The
browser Developer Tools were opened and the Storage tab was selected. The
Cookies section was inspected for the `token` cookie. The `HttpOnly`, `Secure`,
and `SameSite` columns were observed for each cookie. All three were confirmed
as `false` or absent for the session token.

### Phase 2 -- Session Expiration Testing (Token Replay)
A login was performed and the profile page at `http://localhost:3000/profile`
was loaded. A username change to "Prime Hacks" was submitted and the resulting
PUT request was intercepted in Burp Suite, saving the full request including
the active JWT in the Cookie header. Logout was performed and the Storage tab
was checked to confirm the token was deleted from the browser. The saved Burp
request was then modified (username changed from "Prime Hacks" to "Prime Bugs")
and forwarded to the server to test whether the now-logged-out token was still
accepted.

The server returned HTTP 302 Found and redirected to `/profile`, confirming
the token remained valid server-side after logout. To demonstrate account
takeover, a private browsing window was opened, the saved JWT was manually
inserted into the storage tab's cookie section, and the profile URL was
navigated to. The profile page loaded with the "Prime Bug" username visible,
confirming full session restoration without credentials.

### Phase 3 -- JWT Payload Predictability
The JWT token value was copied from the Storage tab and decoded using `jwt.io`.
The decoded payload was reviewed for the presence of sensitive fields and
predictable values.

### Phase 4 -- Session Fixation Testing
A new cookie named `token` was manually created in the Storage tab before
logging in. The value was a JWT crafted using `jwt.io` and signed with no
algorithm (`alg: none`). Login was then performed with valid credentials.
The Storage tab was checked post-login to observe whether the server adopted
the pre-planted token or replaced it.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Component |
|----|--------------|----------|--------------------|
| 01 | HttpOnly Set to False on Session Token | High | Session Token Cookie |
| 02 | Secure Set to False on Session Token | High | Session Token Cookie |
| 03 | Session Token Not Invalidated Server-Side After Logout | Critical | localhost:3000/profile |
| 04 | Sensitive Data in JWT Payload (Informational) | Low | JWT Token Payload |
| 05 | Session Fixation -- Not Vulnerable (Control Working) | Informational | Login Endpoint |

---

## 6. Detailed Findings

---

### Finding 01 -- HttpOnly Set to False on Session Token

#### Severity
High

#### Affected Component
Session token cookie -- `localhost:3000`

#### Description
Inspection of the OWASP Juice Shop cookie in Firefox Developer Tools confirmed
that the `HttpOnly` attribute is set to `false` on the session token. The
`HttpOnly` attribute instructs the browser to block JavaScript from accessing
the cookie via `document.cookie`. When it is absent, any JavaScript executing
in the page context can read the session token value directly. This is the
mechanism that allows XSS payloads to steal session cookies from a victim's
browser and send them to an attacker-controlled server.

#### Proof of Concept

Firefox Developer Tools Storage tab showing the `token` cookie with `HttpOnly`
set to `false`:



![Firefox Developer Tools - Storage Tab Showing HttpOnly false on Session Token](image.jpg)



Relevant cookie attributes from the Storage tab:
```

Name:      token
Domain:    localhost
HttpOnly:  false
Secure:    false
SameSite:  None
```

Because `HttpOnly` is false, the following JavaScript executed in the page
context would successfully read and expose the session token:
```javascript
console.log(document.cookie);
// Returns: token=eyJ0eXAiOi...
```

#### Impact
Any XSS vulnerability on the Juice Shop platform would allow an attacker to
execute `document.cookie` in the victim's browser and exfiltrate the session
token to an external server. The attacker can then use the token to impersonate
the victim's session without knowing the account password. Given that XSS
vulnerabilities exist in OWASP Juice Shop by design, the combination of XSS
and an accessible session cookie is a direct path to account takeover for any
logged-in user who views a page containing a stored XSS payload.

#### Remediation
Set the `HttpOnly` flag to `true` on the session token cookie at the point
the server issues it. In a Node.js/Express application:
```javascript
res.cookie('token', jwtValue, {
    httpOnly: true,
    secure: true,
    sameSite: 'Strict'
});
```
This prevents any JavaScript from reading the cookie regardless of what XSS
payloads execute on the page.

---

### Finding 02 -- Secure Set to False on Session Token

#### Severity
High

#### Affected Component
Session token cookie -- `localhost:3000`

#### Description
The `Secure` attribute was confirmed as `false` on the session token from the
same Storage tab inspection. The `Secure` attribute instructs the browser to
transmit the cookie only over HTTPS connections. Without it, the browser
includes the session token in HTTP requests. OWASP Juice Shop was accessed
over an unencrypted HTTP connection throughout this lab (`http://localhost:3000`),
meaning every request containing the session cookie was transmitted in
cleartext. A network-level attacker positioned between the client and server
could capture the session token from the unencrypted HTTP traffic and replay
it to impersonate the session.

#### Proof of Concept

The Storage tab confirmed the `Secure` attribute as `false` alongside `HttpOnly`
on the same token row:



![Firefox Developer Tools - Secure Attribute false on Session Token](image.jpg)



Because the application was accessed over HTTP, the request containing the
session token was transmitted in cleartext on every page load. A Wireshark
capture on the local interface would show the `Cookie: token=eyJ...` header
in plaintext in every request.

#### Impact
In a production deployment where the application is accessible over a network,
a passive observer on the same network segment (LAN, public Wi-Fi) running
a packet capture tool could collect valid session tokens from HTTP traffic
without injecting any payload or interacting with the application. Captured
tokens could then be replayed for account access. The absence of the `Secure`
attribute removes the browser's enforcement of encrypted transmission as a
prerequisite for sending the cookie.

#### Remediation
Set the `Secure` flag to `true` on the session token cookie and enforce
HTTPS across the entire application. The `Secure` flag alone is not sufficient
if the application also responds to HTTP; an HTTP Strict-Transport-Security
(HSTS) header must also be added to redirect all HTTP traffic to HTTPS:
```
Strict-Transport-Security: max-age=31536000; includeSubDomains
```
In the cookie configuration:
```javascript
res.cookie('token', jwtValue, {
    httpOnly: true,
    secure: true
});
```

---

### Finding 03 -- Session Token Not Invalidated Server-Side After Logout

#### Severity
Critical

#### Affected Component
`http://localhost:3000/profile` -- Profile update endpoint

#### Description
The OWASP Juice Shop logout mechanism instructs the client browser to delete
the session token cookie from local storage. However, the server does not
maintain a record of invalidated tokens and does not reject tokens issued
before the logout event. Because Juice Shop uses a stateless JWT architecture
without a server-side denylist, a token that was valid at the time of capture
remains valid indefinitely -- or until the JWT's own expiry claim is reached --
regardless of whether the user has logged out.

This was confirmed through a full proof of concept. An authenticated request
was intercepted in Burp Suite, saving the active JWT. The user was logged out
and the Storage tab confirmed the token was cleared from the browser. The saved
request was then modified and forwarded to the server. The server returned
HTTP 302 Found and redirected to `/profile`, validating the request with a
token belonging to a logged-out session.

#### Proof of Concept

Step 1 -- Username change to "Prime Hacks" intercepted while logged in,
showing the active JWT in the Cookie header:



![Burp Suite - Active JWT Intercepted During Username Change Request](image.jpg)



Step 2 -- Logout performed, Storage tab confirming token deleted from browser:



![Firefox Developer Tools - Token Cookie Cleared After Logout](image.jpg)

Step 3 -- Saved Burp request modified (username changed to "Prime Bugs") and
forwarded to the server. Server response confirming the old token was accepted:



![Burp Suite - 302 Found Response Confirming Replayed Token Accepted by Server](image.jpg)



Server response:
```
HTTP/1.1 302 Found
Set-Cookie: token=...
Location: /profile
Found. Redirecting to /profile
```

Step 4 -- Private browsing window opened. Saved JWT manually injected into
the Storage tab cookie field. Profile page navigated to and loaded successfully,
showing username "Prime Bug" without any credentials being provided:



![Private Browsing - Profile Page Loaded with Prime Bug Username via Injected JWT](image.jpg)



#### Impact
Any attacker who captures a valid JWT at any point during an authenticated
session retains the ability to use that token indefinitely after the victim
logs out. If the token was captured via XSS (enabled by Finding 01), network
interception (enabled by Finding 02), or physical access to browser history,
the attacker has persistent access to the account. The victim's logout action
provides no security benefit against an attacker holding the token. The account
can be accessed, profile details changed, and purchases made from any browser
or location using only the stolen token string.

#### Remediation
Implement a server-side JWT denylist that records the `jti` (JWT ID) claim of
every token at the time of logout. On every authenticated request, the server
must check the incoming token's `jti` against the denylist before processing
the request. If a match is found, the request must be rejected with HTTP 401
regardless of the signature's validity:
```javascript
// On logout
jwtDenylist.add(decodedToken.jti);

// On every protected request
if (jwtDenylist.has(decodedToken.jti)) {
    return res.status(401).json({ error: "Token invalidated." });
}
```
Enforce short token expiry times (15 to 30 minutes) combined with refresh
token rotation to limit the window of opportunity for replayed tokens.

---

### Finding 04 -- Sensitive Data in JWT Payload (Informational)

#### Severity

Low

#### Affected Component
JWT token payload -- decoded via `jwt.io`

#### Description
The JWT token issued by OWASP Juice Shop was decoded using `jwt.io` by pasting
the base64-encoded token value. The payload contained fields including the
user's internal database ID, email address, hashed password value, role
assignment, last login IP address, and profile image path. Base64 encoding is
not encryption. While the cryptographic signature prevents an attacker from
modifying the payload and having the server accept the change, any actor who
intercepts or extracts the token can decode and read all payload fields without
any key or password.

#### Proof of Concept

JWT decoded in `jwt.io` showing payload content:



![jwt.io - Decoded JWT Payload Showing id, email, role, lastLoginIp Fields](image.jpg)

Payload fields observed:
```json
{
  "id": 24,
  "username": "",
  "email": "hackerfabelt@gmail.com",
  "password": "64b1893c79810badd2df0b76617cd9e3",
  "role": "customer",
  "deluxeToken": "",
  "lastLoginIp": "127.0.0.1",
  "profileImage": "/assets/public/images/uploads/default.svg"
}
```

Modifying `id` to `1` or `role` to `admin` and re-signing was tested but
could not be exploited because the server's secret cryptographic key is not
known. The server correctly rejected any token with an invalid signature.

#### Impact
The impact is limited to disclosure rather than token manipulation. Any actor
with access to the token value (via the attack paths confirmed in Findings 01,
02, and 03) can immediately read the victim's email address, internal ID,
and last login IP. The presence of the password hash in the payload is
particularly notable -- while the hash is not the plaintext password, it
provides a direct target for offline cracking attempts.

#### Remediation
Remove all sensitive fields from the JWT payload. The token only needs to carry
the minimum information required to identify the session server-side. A user ID
or session reference is sufficient. Fields including password hash, email, role,
and last login IP should be retrieved server-side on each request using the
session identifier, not embedded in the client-held token:
```javascript
// Minimal JWT payload
const payload = {
    jti: crypto.randomUUID(),
    sub: user.id,
    iat: Math.floor(Date.now() / 1000),
    exp: Math.floor(Date.now() / 1000) + 900
};
```

---

### Finding 05 -- Session Fixation Not Present (Control Working)

#### Severity
Informational

#### Affected Component
Login endpoint -- `http://localhost:3000`

#### Description
A session fixation test was performed by creating a manually crafted JWT in
`jwt.io` signed with no algorithm (`alg: none`) and injecting it into the
Storage tab cookie field before logging in. After logging in with valid
credentials, the Storage tab was checked. The application discarded the
pre-planted token and replaced it with a new, cryptographically signed JWT
generated for the authenticated session. This confirms the application does
not adopt a pre-existing token on login and is not vulnerable to session
fixation.

#### Proof of Concept

Browser Storage tab before login showing the manually injected fake token:



![Firefox Developer Tools - Fake JWT Injected into Storage Before Login](image.jpg)



Browser Storage tab after login showing the fake token replaced by a new
legitimately signed JWT:



![Firefox Developer Tools - New Signed JWT Replacing Fake Token After Login](image.jpg)



#### Impact
Session fixation is not exploitable against this application. The login
mechanism correctly generates a fresh token on each authentication event
and does not honour tokens pre-existing in the client's storage. This is
the expected behavior.

#### Remediation
No action required for this finding. The control is working as intended
and should be maintained in future development.

---

## 7. Attack Chain

```
[Step 1] Cookie Inspection
Session token cookie observed in Firefox Developer Tools
HttpOnly: false / Secure: false / SameSite: None
Token accessible to JavaScript, transmittable over HTTP
        |
        v
[Step 2] Token Capture During Authenticated Session
Login performed, profile page loaded
Username change to "Prime Hacks" submitted at /profile
PUT request intercepted in Burp Suite, JWT saved
        |
        v
[Step 3] Logout and Client-Side Verification
User logged out
Storage tab confirmed: token cookie deleted from browser
Server-side state: token still valid (no denylist)
        |
        v
[Step 4] Token Replay After Logout
Saved Burp request modified: username changed to "Prime Bugs"
Request forwarded with old JWT to /profile endpoint
Server response: HTTP 302 Found, redirected to /profile
Server accepted the token from a logged-out session
        |
        v
[Step 5] Full Account Takeover Demonstrated
Private browsing window opened (no existing session)
Saved JWT manually inserted into Storage tab
Navigated to http://localhost:3000/profile
Profile page loaded: username displayed as "Prime Bug"
Full account access confirmed without credentials
```

---

## 8. Tools Used

- Kali Linux (lab environment)
- Firefox with Developer Tools (cookie inspection and manual storage manipulation)
- Burp Suite (request interception, JWT capture, and replay testing)
- `jwt.io` (JWT decoding and session fixation token crafting)
- OWASP Juice Shop (intentionally vulnerable target application)

---

## 9. Challenges Encountered

- **Locating the correct request to intercept for the session expiration test:**
  The profile page makes multiple requests on load. Identifying the specific
  PUT request carrying the username change -- rather than the GET requests for
  page assets -- required filtering Burp's HTTP history by method and endpoint.
  Filtering on `PUT /profile` isolated the correct request containing the active
  JWT.
- **JWT replay returning 302 instead of 200:** The replayed request returned
  an HTTP 302 redirect rather than a direct 200 response with profile data.
  This initially appeared ambiguous. Checking the `Location` header confirmed
  the redirect destination was `/profile`, not a logout or error page, confirming
  the server accepted the token and processed the request successfully.
- **Session fixation test required pre-login timing:** The fake token needed
  to be in storage before the login request was submitted. If login was performed
  before the fake token was created, the test would have been invalid. The Storage
  tab was set up with the fake token first, then the login form was submitted in
  the same browser tab without refreshing or navigating away.

---

## 10. Key Takeaways

- **Logout without server-side invalidation is not logout:** From the server's
  perspective, deleting the cookie from the browser changes nothing. The token
  remains valid. An attacker who captured the token before the user logged out
  retains full session access indefinitely. Client-side cookie deletion is a
  UI action, not a security control. The only reliable logout mechanism is a
  server-side denylist that rejects the token on all future requests.
- **HttpOnly and Secure are two independent controls for different attack paths:**
  HttpOnly blocks XSS-based cookie theft. Secure blocks network-level interception.
  Both must be set because the attack path changes depending on the attacker's
  position. An attacker on the same network does not need XSS. An attacker who
  has found an XSS does not need network access. Each missing flag opens a
  separate attack channel.
- **JWT payload confidentiality is a separate concern from JWT integrity:** The
  cryptographic signature ensures the token cannot be modified without the server
  key. It does not hide the payload contents from anyone who holds the token.
  Developers who treat base64 encoding as a form of encryption and embed
  sensitive fields in JWT payloads are exposing that data to any party who
  intercepts the token. Minimizing payload content is a design requirement, not
  an optional hardening step.
- **Session fixation resistance requires active token replacement on login:**
  Juice Shop correctly replaced the pre-planted token with a fresh one at login.
  This is the right behavior and it works because the application generates a
  new token server-side rather than adopting whatever token the client presents.
  Applications that read a pre-existing session identifier and use it as the
  authenticated session without regenerating it are vulnerable to fixation.
- **Private browsing provides no protection against token injection:** The
  account takeover proof of concept was performed in a private browsing window
  specifically to confirm no residual browser state contributed to the result.
  Private browsing cleared all existing cookies and storage. The session was
  restored purely from the manually injected token, confirming the attack works
  from any browser, device, or location where the attacker can paste the
  token value.

---

## 11. Disclaimer

> This assessment was conducted exclusively against OWASP Juice Shop, an
> intentionally vulnerable web application deployed in an isolated local lab
> environment at `http://localhost:3000`. No production systems, real user
> accounts, or public-facing infrastructure were accessed at any point.
> All session tokens used during this assessment belonged to test accounts
> created within the local Juice Shop instance. All techniques described in
> this report must only be applied against systems for which explicit written
> authorization has been obtained.
