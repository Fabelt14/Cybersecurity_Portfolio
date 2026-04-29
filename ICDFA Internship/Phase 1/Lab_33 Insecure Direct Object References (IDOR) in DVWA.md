# Insecure Direct Object References (IDOR) in DVWA


## 1. Engagement Overview

An Insecure Direct Object Reference (IDOR) assessment was conducted against
DVWA accessed at `http://127.0.0.1:42001/` on a local Kali Linux environment.
The assessment used the SQL Injection module's User ID lookup page as the test
surface, which exposes user records via a sequential integer `id` parameter in
the URL. Testing was performed while authenticated as a low-privilege user
(`gordonb`) to confirm whether the application enforced ownership checks before
returning records belonging to other accounts. Both manual parameter manipulation
and the real-world implications of automated enumeration were assessed.

---

## 2. Objectives

- Confirm that the DVWA SQL Injection page is vulnerable to IDOR by
  manipulating the `id` URL parameter while authenticated as a low-privilege
  user
- Retrieve user records belonging to accounts other than the currently
  authenticated session
- Assess what personally identifiable information is exposed per record and
  extrapolate the risk to a real production e-commerce context
- Document the access control failure that allows the vulnerability to exist
- Provide specific remediation steps to address the confirmed finding

---

## 3. Scope

**In-Scope:**
- DVWA SQL Injection page at
  `http://127.0.0.1:42001/vulnerabilities/sqli/?id=X&Submit=Submit#`
- `id` GET parameter (integer values 1 through at least 5 confirmed)
- Session authenticated as low-privilege user `gordonb`

**Out-of-Scope:**
- SQL injection exploitation of the same endpoint
- Other DVWA modules not related to IDOR
- Any systems outside the local lab network

**Authorization Statement:**
> This assessment was conducted against DVWA, an intentionally vulnerable web
> application deployed in an isolated local lab environment for security training.
> No production systems or real user accounts were accessed at any point.
> All activities were performed for educational purposes under authorized course
> instruction.

---

## 4. Methodology

### Phase 1 -- Authentication as Low-Privilege User
DVWA was accessed via Firefox and login was performed using the `gordonb`
account credentials. The security level was confirmed as Low. The active session
was established as `gordonb`, a non-administrative user.

### Phase 2 -- Baseline Request Observation
The SQL Injection page was loaded and a user ID was submitted through the form.
The application returned the record for the submitted ID in the page response.
The URL was observed to include the `id` parameter as a GET parameter:
`?id=1&Submit=Submit#`. The response for `id=1` returned `First name: admin,
Surname: admin`, confirming the page queries the database using the supplied
value and returns the result regardless of which user is currently authenticated.

### Phase 3 -- Parameter Manipulation
The `id` value in the URL was manually changed to `5` directly in the browser
address bar. The page was loaded without re-authenticating or modifying any
session-related values. The response was observed and the returned record was
compared against the currently authenticated user to confirm unauthorized
cross-user data access.

### Phase 4 -- Validation
The returned data for `id=5` was confirmed to belong to a different user
(`Bob Smith`), not to the authenticated session (`gordonb`). No access control
error, redirect, or permission denial was returned, confirming the application
performed no ownership verification before serving the record.

### Phase 5 -- Documentation
All URL values, application responses, and screenshots were recorded throughout
the assessment.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | IDOR via Sequential Integer id Parameter | High | /vulnerabilities/sqli/?id= |
| 02 | Missing Authorization Check on User Record Access | High | /vulnerabilities/sqli/?id= |

---

## 6. Detailed Findings

---

### Finding 01 -- IDOR via Sequential Integer id Parameter

#### Severity
High

#### Affected Endpoint
`/vulnerabilities/sqli/?id=X&Submit=Submit#` -- GET, `id` parameter

#### Description
The DVWA SQL Injection page accepts an integer `id` value as a GET parameter
and queries the database for the user record matching that ID. The application
returns the queried record directly in the response with no check to confirm that
the record belongs to the currently authenticated session. While logged in as
`gordonb` (a low-privilege, non-admin account), changing the `id` parameter in
the URL returned records belonging to other user accounts, including the `admin`
account and a user identified as `Bob Smith`.

The vulnerability exists because the application uses a direct database key as
the object reference, the key is exposed in the URL as a predictable sequential
integer, and the backend query executes against any valid ID without verifying
whether the requesting session has authorization to view that record.

#### Proof of Concept

DVWA SQL Injection page with `id=1` submitted while authenticated as
`gordonb`, returning the admin account record:



![DVWA SQL Injection Page - id=1 Returns admin Record While Logged in as gordonb](image.jpg)



URL confirming `gordonb` session and `id=1` in the address bar:



![URL Bar - gordonb Session Active, id=1 Returning admin Data](image.jpg)



`id` parameter changed to `5` in the URL directly, returning a different user's
record without any session change or permission prompt:



![DVWA - id=5 Returns Bob Smith Record Without Authorization Error](image.jpg)



URL observed during the unauthorized access:
```

http://127.0.0.1:42001/vulnerabilities/sqli/?id=5&Submit=Submit#
```

Application response:
```
ID: 5
First name: Bob
Surname: Smith
```

The `gordonb` session was active throughout. No error, redirect, or
authentication challenge was returned when accessing `id=5`.

#### Impact
Any authenticated user on the platform can enumerate every user record in the
database by incrementing the `id` value. The DVWA lab returns first name and
surname. In a production application with a comparable architecture, the same
parameter manipulation would return whatever fields the backend query selects,
which typically includes email addresses, phone numbers, physical addresses,
and on e-commerce platforms, payment method details and order history. An
attacker with a simple script incrementing from 1 to the maximum ID could
download the entire user database without exploiting any technical
vulnerability beyond reading a URL parameter.

The First American Financial breach (2019) involved the same pattern at scale.
Sequential document IDs in URLs with no authorization check resulted in
approximately 885 million sensitive mortgage records being accessible to anyone
who incremented the ID. The Parler scrape (2021) was executed using a for loop
counting through sequential post IDs. Both incidents occurred because the
application trusted the client to request only what it was entitled to.

#### Remediation
- Implement a server-side authorization check before returning any record.
  The check must confirm that the object requested belongs to the currently
  authenticated user, not just that the user is logged in:
```php
$result = $db->query(
    "SELECT * FROM users WHERE user_id = ? AND session_user_id = ?",
    [$id, $_SESSION['user_id']]
);
if (!$result) { http_response_code(403); die("Access denied."); }
```

- Replace sequential integer IDs in URLs with non-guessable indirect
  references such as UUIDs or short random tokens mapped server-side to
  the actual database key. This prevents enumeration even if the
  authorization check is somehow bypassed:
```php
// Store UUID-to-ID mapping in session or a lookup table
$uuid = bin2hex(random_bytes(8));
$_SESSION['resource_map'][$uuid] = $actual_db_id;
// URL exposes ?ref=a3f7c2... not ?id=5
```
- Apply the same authorization check to every endpoint that returns
  user-specific data, not just the user lookup page. IDOR is frequently
  found on order history, invoice, and profile update endpoints that share
  the same missing check pattern.

---

### Finding 02 -- Missing Authorization Check on User Record Access

#### Severity
High

#### Affected Endpoint

`/vulnerabilities/sqli/?id=X&Submit=Submit#` -- Backend query logic

#### Description
The root cause of Finding 01 is the absence of an authorization check in the
backend query. The application constructs a SQL query using the `id` parameter
and returns whatever record the database returns. There is no WHERE clause
condition or session comparison to confirm the requested record belongs to the
requesting user. Authentication (confirming the user is logged in) and
authorization (confirming the user may access a specific resource) are separate
concerns. The DVWA SQL Injection page implements authentication via the DVWA
session system but performs no authorization check on the returned object.

#### Proof of Concept

The URL and session state at the time of unauthorized access confirmed both
issues simultaneously: `gordonb` was the active session, `id=5` was the
parameter, and the server returned `Bob Smith` without any access control
decision:

```
Session:  gordonb (low-privilege, non-admin)
Request:  GET /vulnerabilities/sqli/?id=5&Submit=Submit#
Response: ID: 5 / First name: Bob / Surname: Smith
Expected: 403 Forbidden or redirect to gordonb's own record
Actual:   Bob Smith's record returned with HTTP 200
```

#### Impact
The absence of an authorization check means the application cannot distinguish
between a user legitimately accessing their own data and an attacker accessing
someone else's. Every user-facing endpoint that returns data based on a URL
parameter is susceptible to the same issue if the pattern is consistent across
the application. In a real application, the missing check typically spans
multiple endpoints: profile pages, order histories, invoice downloads, and
account settings pages all commonly share the same underlying access control
gap.

#### Remediation
- Establish a consistent authorization pattern applied at the service layer
  rather than duplicated in individual endpoints. A middleware function that
  confirms resource ownership before any data query executes removes the
  risk of individual endpoints being missed:
```php
function authorizeResourceAccess($resource_id, $session_user_id) {
    $owner = $db->query(
        "SELECT owner_id FROM resources WHERE id = ?", [$resource_id]
    );
    if ($owner['owner_id'] !== $session_user_id) {
        http_response_code(403);
        die("Access denied.");
    }
}
```
- Log all 403 responses from resource access endpoints. Repeated 403s from
  the same authenticated session are a reliable signal of IDOR probing and
  should trigger an alert in any security monitoring system.

---

## 7. Attack Chain

```
[Step 1] Authentication
gordonb logs in to DVWA at low security level
Session established as non-administrative, low-privilege user
        |
        v
[Step 2] Baseline Observation
SQL Injection page loaded, id=1 submitted via form
Response: First name: admin / Surname: admin
URL reveals: ?id=1&Submit=Submit#
id parameter is sequential integer, directly exposed in URL
        |
        v
[Step 3] Parameter Manipulation - id=1
URL modified directly in browser address bar: ?id=1&Submit=Submit#
Application returns admin's record to gordonb session
No authorization error returned
        |
        v
[Step 4] Parameter Manipulation - id=5
URL modified to: ?id=5&Submit=Submit#
Application returns: First name: Bob / Surname: Smith
No session change, no re-authentication, no permission check
Cross-user data access confirmed
        |
        v
[Step 5] Extrapolated Risk
Sequential IDs 1 through N are all accessible via the same pattern
A Python script looping from 1 to 100,000 would download every
user record in the database with no special tooling required
```

---

## 8. Tools Used

- Kali Linux (lab environment)
- Firefox (DVWA access and URL parameter manipulation)
- DVWA (intentionally vulnerable target application)

---

## 9. Challenges Encountered

- **DVWA's SQL Injection page was used as the IDOR test surface:** The
  DVWA installation used in this lab did not include a dedicated IDOR module.
  The SQL Injection page was used because it exposes a user record lookup
  with a sequential `id` GET parameter, which directly demonstrates the IDOR
  pattern. The vulnerability observed is IDOR regardless of the module label,
  because the root issue is the missing authorization check on the `id`
  parameter rather than the SQL construction behind it.
- **Minimal data exposure in DVWA limits real-world impact demonstration:**
  The DVWA SQL Injection page returns only first name and surname per record.
  In a production e-commerce or banking application with the same access
  control gap, the same URL manipulation would return a much broader set of
  fields. The risk assessment in this report extrapolates to that context based
  on the confirmed pattern rather than the specific DVWA field set.

---

## 10. Key Takeaways

- **Authentication and authorization are different controls:** DVWA confirmed
  the session was `gordonb` throughout the test. Being authenticated did not
  prevent access to other users' records because the application checked
  identity but not ownership. Every endpoint that returns data based on a
  parameter must independently verify that the requesting session owns the
  requested object.
- **Sequential integers in URLs are an enumeration invitation:** The `id`
  parameter values 1, 2, 3, 4, 5 require no guessing. Any authenticated
  user who observes the URL pattern can iterate through every valid ID with
  a for loop. Using UUIDs or opaque tokens as indirect references does not
  fix the access control gap, but it removes the enumeration advantage and
  forces an attacker to already know a valid reference before they can
  attempt unauthorized access.
- **The impact scales with what the endpoint returns:** In DVWA, the result
  is a first name and surname. In the First American Financial breach, the
  same pattern returned mortgage documents, bank account numbers, and tax
  records. The vulnerability class is identical. The difference is only what
  the database query returns. Applications that store payment data, health
  records, or legal documents with this pattern face a much higher consequence
  from the same missing check.
- **Logging failed access attempts is a practical detection control:** An
  attacker iterating through IDs will produce a high volume of requests in a
  short window. If the authorization check is in place and returns 403 for
  unauthorized requests, monitoring for repeated 403s from the same session
  provides a reliable alert signal for IDOR probing, even when the attacker
  is authenticated.

---

## 11. Disclaimer

> This assessment was conducted exclusively against DVWA, an intentionally
> vulnerable web application deployed in an isolated local lab environment at
> `http://127.0.0.1:42001/`. No production systems, real user accounts, or
> external databases were accessed at any point. All techniques described in
> this report were performed for educational purposes under authorized course
> instruction. These techniques must only be applied against systems for which
> explicit written authorization has been obtained.