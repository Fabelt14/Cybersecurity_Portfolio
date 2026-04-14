XSS Vulnerabilities in DVWA

## 1. Engagement Overview

A cross-site scripting assessment was conducted against DVWA hosted locally at
`http://192.168.92.3/dvwa/index.php`, accessed via Firefox on Kali Linux. The
assessment tested three XSS vulnerability types across multiple security levels:
Reflected XSS, Stored XSS, and DOM-based XSS. Each security level was tested
separately to observe how the application's filtering behavior changed and to
determine whether those filters could be bypassed. Security level was adjusted
between tests using the DVWA setup panel.

---

## 2. Objectives

- Identify input fields across DVWA's XSS modules that do not sanitize
  user-supplied HTML or JavaScript
- Confirm exploitability of Reflected, Stored, and DOM-based XSS at each
  security level
- Bypass client-side input length restrictions and server-side tag blocklists
  to demonstrate that surface-level filters do not constitute adequate protection
- Confirm that the high security level's HTML entity encoding successfully
  neutralizes injection attempts
- Provide specific remediation recommendations for each confirmed finding

---

## 3. Scope

**In-Scope:**
- DVWA at `http://192.168.92.3/dvwa/index.php`
- XSS (Reflected) module
- XSS (Stored) module
- XSS (DOM) module
- Security levels tested: Low, Medium, High

**Out-of-Scope:**
- Other DVWA vulnerability modules not related to XSS
- Any systems outside the local lab network
- Session hijacking or real credential theft against live users

**Authorization Statement:**
> This assessment was conducted against DVWA, an intentionally vulnerable web
> application deployed in an isolated local lab environment for security training.
> No production systems or real user accounts were accessed at any point. All
> activities were performed for educational purposes under authorized course
> instruction.

---

## 4. Methodology

### Phase 1 — Reconnaissance
DVWA was accessed via Firefox and the XSS Reflected, Stored, and DOM
modules were identified from the left-hand navigation menu. The application's
security level was confirmed as Low for the initial tests.

### Phase 2 — Vulnerability Identification
A harmless HTML tag `<p> Hello DVWA </p>` was submitted into the Reflected
XSS name field. The application rendered the tag as formatted HTML in the
response rather than displaying it as escaped text. The browser received `Hello`
followed by a rendered paragraph containing `Hello DVWA`, confirming that
the server passed the input directly into the HTML response without converting
`<` and `>` to `&lt;` and `&gt;`. This confirmed the field was injectable.

### Phase 3 — Exploitation
Payloads were tested progressively across security levels. A script-based
payload was used for Stored XSS to confirm database persistence. An `img`
onerror payload was used for medium-level Stored XSS after the client-side
maxlength attribute was raised from 10 to 100 via the browser's developer
tools. A case-modified script tag was used to bypass the medium-level
blocklist on the Reflected XSS page. A JavaScript anchor tag was used as a
custom payload on the Reflected XSS low-level page.

### Phase 4 — Validation
Each payload's execution was confirmed by observing the resulting alert dialog
or rendered HTML. The high-level Stored XSS attempt was validated by
inspecting the stored output in the page source and confirming the tags were
encoded rather than executed.

### Phase 5 — Documentation
All payloads, application responses, and screenshots were recorded throughout
the assessment.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | Stored XSS - Script Tag Execution (Low) | High | /dvwa/vulnerabilities/xss_s/ |
| 02 | Reflected XSS - Script Execution via URL (Low) | High | /dvwa/vulnerabilities/xss_r/ |
| 03 | Stored XSS - img onerror Bypass (Medium) | High | /dvwa/vulnerabilities/xss_s/ |
| 04 | Reflected XSS - Case-Sensitive Blocklist Bypass (Medium) | High | /dvwa/vulnerabilities/xss_r/ |
| 05 | Reflected XSS - JavaScript Anchor Tag (Low) | Medium | /dvwa/vulnerabilities/xss_r/ |
| 06 | Client-Side maxlength Bypassed via Developer Tools | Medium | /dvwa/vulnerabilities/xss_s/ |
| 07 | High-Level HTML Entity Encoding - Attack Neutralized | Informational | /dvwa/vulnerabilities/xss_s/ |

---

## 6. Detailed Findings

---

### Finding 01 -- Stored XSS via Script Tag (Low Security)

#### Severity
High

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_s/` -- Name field, POST

#### Description
At low security level, the Stored XSS page accepts input into a Name field and
a Message field. Neither field applies server-side sanitization before storing
the values in the database. A script tag injected into the Name field was saved
to the backend database and served back to every subsequent visitor who
loaded the page, executing without any user interaction beyond visiting the URL.

#### Proof of Concept

Payload injected into the Name field:
```
<script>alert(document.domain)</script>
```

The payload was submitted and the page was refreshed. On reload, the stored
script executed automatically, producing an alert dialog displaying
`192.168.92.3`. The alert fired again on every subsequent page load, confirming
the payload was stored persistently in the database.

![Stored XSS - alert(document.domain) Executing on Page Load](image.jpg)

#### Impact
Stored XSS requires no victim interaction beyond visiting the page. Any
authenticated or unauthenticated user who loads the guestbook page after
the payload is submitted has the script executed in their browser. In a real
application, this could be used to steal session cookies, redirect users to
phishing pages, or log keystrokes for every visitor to the affected page.

#### Remediation
- Apply `htmlspecialchars()` to all user-supplied data before inserting it into
  the database and again before rendering it in the HTML response:
```php
echo htmlspecialchars($name, ENT_QUOTES, 'UTF-8');
```
- Implement a Content Security Policy header to restrict script execution to
  trusted sources:
  `Content-Security-Policy: default-src 'self'; script-src 'self'`

---

### Finding 02 -- Reflected XSS via Script Tag in URL (Low Security)

#### Severity
High

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_r/` -- `name` GET parameter

#### Description
The Reflected XSS page at low security level takes a name value from the URL's
`name` GET parameter and prints it directly into the HTML response. A script
tag placed in the URL parameter was reflected into the page and executed by the
browser. The executed payload displayed an alert dialog, confirming the input
was not sanitized before being written into the response.

#### Proof of Concept

URL with payload injected into the `name` parameter:
```
http://192.168.92.3/dvwa/vulnerabilities/xss_r/?name=<script>alert("Hacked")</script>#
```

The browser received the script tag as part of the HTML response and executed
it, displaying an alert containing `Hacked`.

![Reflected XSS - Alert Dialog Displaying "Hacked" from URL Parameter](image.jpg)

#### Impact
Reflected XSS requires the victim to visit a crafted URL. An attacker distributes
the malicious URL via email, messaging, or social media. When the victim clicks
the link, the script executes in their browser session, where it can access their
session cookies, make authenticated requests on their behalf, or redirect them
to an attacker-controlled page.

#### Remediation
- Encode all user-supplied GET parameters before rendering them in the
  HTML response using `htmlspecialchars()` with `ENT_QUOTES`
- Validate that the `name` parameter contains only expected characters
  (letters and spaces) and reject any input containing angle brackets, quotes,
  or script-related strings

---

### Finding 03 -- Stored XSS via img onerror Payload (Medium Security)

#### Severity
High

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_s/` -- Name field, POST

#### Description
At medium security level, the Name field enforces a client-side `maxlength`
attribute of 10 characters, limiting what can be typed in the browser. This
restriction exists only in the HTML attribute and is not enforced server-side.
Using the browser's developer tools, the `maxlength` attribute was changed
from `10` to `100` directly in the DOM. This bypassed the length restriction,
allowing a longer payload to be submitted. The `<script>` tag was blocked by
the server at medium level, so an `img` tag with an `onerror` event handler
was used instead, which the server did not filter.

#### Proof of Concept

Developer tools used to modify the maxlength attribute in the Name field:

![Developer Tools - maxlength Changed from 10 to 100](image.jpg)

Payload injected after the maxlength was raised:
```
<img src=x onerror=alert(I.hack.you)>
```

The browser attempted to load an image from source `x`. Because `x` is not a
valid image path, the load failed and the `onerror` event handler fired,
executing the alert. The payload was stored in the database and fired for every
subsequent visitor.

![Stored XSS - img onerror Alert Executing via Broken Image Source](image.jpg)

#### Impact
Client-side input length restrictions provide no real security. They can be
bypassed in seconds using any browser's developer tools, with no special
software required. The `img onerror` payload bypassed the medium-level
`<script>` tag filter entirely, confirming that tag-based blocklists are
insufficient as a standalone defense.

#### Remediation
- Enforce input length limits server-side in PHP, not only in the HTML
  attribute:
```php
if (strlen($name) > 10) { die("Input too long"); }
```
- Apply output encoding with `htmlspecialchars()` rather than blocking
  specific tags, which covers all HTML elements including `img`, `body`,
  `svg`, and any future HTML tags with event handler attributes

---

### Finding 04 -- Reflected XSS via Case-Sensitive Blocklist Bypass (Medium Security)

#### Severity
High

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_r/` -- `name` GET parameter

#### Description
At medium security level, the Reflected XSS page blocks the `<script>` tag
using a string-matching filter. The filter only matches the lowercase string
`<script>` and does not account for variations in letter case. Submitting the
same payload with uppercase letters bypassed the filter entirely, and the
browser executed the JavaScript.

#### Proof of Concept

Payload blocked by the medium-level filter:
```
<script>alert('Hacked')</script>
```

Payload that bypassed the filter by using uppercase letters:
```
<SCRIPT>alert('Hacked') </SCRIPT>
```

The uppercase version was not matched by the server's blocklist, so it passed
through to the response. The browser parsed `<SCRIPT>` as a valid script tag
and executed the alert.

![Reflected XSS Medium Level - SCRIPT Uppercase Bypass Alert Displayed](image.jpg)

#### Impact
Case-based blocklists are unreliable. HTML tag names are case-insensitive in
browsers, meaning `<SCRIPT>`, `<Script>`, and `<sCrIpT>` all execute
identically. Any filter that checks for an exact lowercase string is bypassed
by the first case variation an attacker tries.

#### Remediation
- Replace blocklist-based filtering with output encoding using
  `htmlspecialchars()`. Encoding converts `<` to `&lt;` regardless of
  letter case, making case variations irrelevant
- If filtering must be used as an additional layer, apply it case-insensitively
  using `stripos()` or `preg_replace()` with the `i` flag in PHP

---

### Finding 05 -- Reflected XSS via JavaScript Anchor Tag (Low Security, Custom Payload)

#### Severity
Medium

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_r/` -- Name field, GET

#### Description
A custom payload using an HTML anchor tag with a `javascript:` URI was
injected into the Reflected XSS name field at low security level. Unlike a script
tag that executes on page load, this payload required the victim to click the
rendered link. The anchor tag was stored in the response and the clickable text
"Free Money" was displayed on the page. Clicking the link triggered the
alert dialog.

#### Proof of Concept

Payload injected into the name field:
```
<a href="javascript:alert('Hacked')">Free Money</a>
```

The application rendered the anchor tag in the response. The page displayed
the text "Hello" followed by a clickable hyperlink reading "Free Money".
Clicking the link executed the JavaScript alert.

![Reflected XSS - Anchor Tag with javascript: URI Rendered and Clicked](image.jpg)

The browser inspector confirmed the raw anchor tag was present in the HTML
without any encoding applied:
```html
<a href="javascript:alert('Hacked')">Free Money</a>
```

#### Impact
This payload type is used in social engineering attacks where the rendered
link looks legitimate but carries a malicious JavaScript action. In a real
application, the `javascript:` URI in the href can execute any JavaScript the
attacker chooses, including cookie theft and session hijacking, triggered by a
single click on a link that appears harmless to the victim.

#### Remediation
- Apply output encoding before rendering user input in the response, which
  converts the `<` and `"` characters and prevents the anchor tag from
  being treated as HTML
- Implement a Content Security Policy that blocks `javascript:` URI
  execution:
  `Content-Security-Policy: default-src 'self'`

---

### Finding 06 -- Client-Side maxlength Restriction Bypassed via Developer Tools

#### Severity
Medium

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_s/` -- Name field HTML attribute

#### Description
The Stored XSS page at medium security level sets `maxlength="10"` on the
Name input field. This attribute is rendered in the HTML and enforced by the
browser, but it has no equivalent check on the server side. The attribute was
changed to `maxlength="100"` using the browser's developer inspector in under
10 seconds, after which payloads of any length could be submitted without
restriction.

#### Proof of Concept

Original HTML attribute in the page source:
```html
<input name="txtName" type="text" size="30" maxlength="10">
```

Modified via developer tools to:
```html
<input name="txtName" type="text" size="30" maxlength="100">
```

After the change, the longer `img onerror` payload was submitted successfully,
as documented in Finding 03.

![Developer Tools - maxlength Attribute Modified from 10 to 100](image.jpg)

#### Impact
Client-side restrictions on input length, format, or content are not security
controls. They can be bypassed by anyone using a browser's built-in developer
tools, a proxy like Burp Suite, or by crafting a direct HTTP request. Any
validation that exists only in the browser can be removed or altered before the
request reaches the server.

#### Remediation
- Validate all input length, format, and content on the server side in PHP
  before processing or storing the submitted value
- Client-side restrictions may remain for user experience, but they must
  never be the only enforcement point

---

### Finding 07 -- High-Level HTML Entity Encoding Neutralizes Attack (Informational)

#### Severity
Informational

#### Affected Endpoint
`/dvwa/vulnerabilities/xss_s/` -- Name and Message fields (High security)

#### Description
At high security level, the same `<body onload=alert('BugBot19')>` payload
was submitted into both the Name and Message fields of the Stored XSS page.
The page loaded normally with no alert dialog. Inspecting the stored output
confirmed that the angle brackets had been converted to HTML entities: `<`
became `&lt;` and `>` became `&gt;`. Because the browser received encoded
text rather than markup, it displayed the characters as visible text rather than
interpreting them as an HTML tag. The attack was neutralized at the storage
and rendering layer.

#### Proof of Concept

Payload submitted at high security level:
```
<body onload=alert('BugBot19')>
```

Output observed in the page after submission:
```
&lt;body onload=alert('BugBot19')&gt;
```

The browser displayed the payload as literal text, confirming the HTML entity
encoding was applied and working correctly.

#### Impact
This is the expected and correct behavior. HTML entity encoding is the
recommended defense against XSS. It does not block the input from being
stored; it ensures that when the stored value is rendered, the browser treats
angle brackets as text characters rather than HTML tag delimiters, preventing
any script from executing.

#### Remediation
The high-level implementation demonstrates the correct approach. The same
encoding must be applied consistently at low and medium security levels to
bring them to the same standard. Applying `htmlspecialchars()` with
`ENT_QUOTES` and `UTF-8` encoding to all user-supplied output fields removes
the vulnerability across all three XSS module types.

---

## 7. Attack Chain

```
[Step 1] Vulnerability Identification
Harmless HTML tag submitted: <p> Hello DVWA </p>
Application renders <p> as formatted HTML rather than escaped text
Confirms input is written into the response without entity encoding
        |
        v
[Step 2] Stored XSS - Low Level
Payload: <script>alert(document.domain)</script>
Stored in database, executes on every page load for all visitors
Zero-click code execution confirmed
        |
        v
[Step 3] Medium Level - Client-Side Restriction Bypassed
maxlength changed from 10 to 100 via developer tools
Payload: <img src=x onerror=alert(I.hack.you)>
img onerror bypasses script tag filter, stored XSS confirmed
        |
        v
[Step 4] Reflected XSS - Blocklist Bypass
Lowercase <script> blocked by medium-level filter
Uppercase <SCRIPT>alert('Hacked') </SCRIPT> bypasses filter
Case-insensitive tag parsing by browser executes payload
        |
        v
[Step 5] High Level - Attack Neutralized
<body onload=alert('BugBot19')> submitted at high security
Output stored as &lt;body onload=alert('BugBot19')&gt;
HTML entity encoding prevents browser from parsing as markup
```

---

## 8. Tools Used

- Kali Linux (lab environment)
- Firefox with built-in Developer Tools (DOM inspection and maxlength modification)
- DVWA v1.x at `http://192.168.92.3/dvwa/` (intentionally vulnerable target)

---

## 9. Challenges Encountered

- **Client-side maxlength blocked medium-level payload submission:** The Name
  field at medium security level only accepted 10 characters, which was too short
  for a functional XSS payload. The restriction existed only in the HTML attribute
  and was removed via the developer inspector before testing continued.
- **Script tag blocked at medium level:** The standard `<script>alert()</script>`
  payload failed at medium security level because the server filtered the exact
  lowercase string. This required switching to an alternative payload type (`img
  onerror`) for the Stored XSS test and a case-modified tag for the Reflected XSS
  test.
- **High-level test produced no alert:** Submitting the `<body onload>` payload
  at high security level produced no visible JavaScript execution. Inspecting the
  stored output was necessary to confirm the encoding was applied rather than
  concluding the payload was silently dropped.

---

## 10. Key Takeaways

- **Blocklists break at the first variation an attacker tries:** The medium-level
  filter blocked `<script>` but not `<SCRIPT>`. That is one keypress of change.
  Real filters must handle every possible HTML tag name, attribute, and encoding
  variation, which is why output encoding is the correct approach rather than
  trying to enumerate what to block.
- **Client-side length limits are not a security control:** The maxlength bypass
  took under 10 seconds using tools every browser ships with by default. Any
  length, format, or content check that lives only in the HTML attribute must be
  duplicated server-side to have any effect.
- **Stored XSS requires no attacker presence after initial injection:** Once the payload is in the database, the attacker does not need to be online or interacting with the application. Every visitor to the page is a victim from that point forward until the payload is removed from the database.
- **HTML entity encoding at high level shows what correct output handling
  looks like:** The contrast between the low and high security levels in this lab
  makes the fix concrete. `htmlspecialchars()` applied to output is the single
  change that moves the application from fully exploitable to correctly defended
  against every payload tested.
- **Custom payloads reveal filter gaps that standard payloads miss:** The
  JavaScript anchor tag payload did not use a `<script>` tag at all. A filter
  that only checks for `<script>` would not have caught it. Testing beyond
  the obvious payloads is necessary to find what a filter actually covers.

---

## 11. Disclaimer

> This assessment was conducted exclusively against DVWA, an intentionally
> vulnerable web application deployed in an isolated local lab environment at
> `http://192.168.92.3`. No production systems, real user accounts, or
> public-facing infrastructure were accessed or affected at any point. All
> techniques and payloads documented in this report were executed in a
> controlled environment for educational purposes under authorized course
> instruction. These techniques must only be used against systems for which
> explicit written authorization has been obtained.
