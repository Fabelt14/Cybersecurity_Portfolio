# Cross-Site Request Forgery (CSRF) Protection

## Overview

This lab demonstrated CSRF attacks and defenses using DVWA (Damn Vulnerable Web Application). CSRF exploits the browser's automatic cookie-sending behavior to trick authenticated users into performing actions they never intended. I simulated an attack that changed a user's password without their knowledge, then implemented token-based validation to prevent the exploit, and finally tested whether the protection actually blocks unauthorized requests.

## Objectives

- Understand how CSRF attacks exploit browser trust in authenticated sessions
- Simulate a real CSRF attack using malicious HTML hosted on an external domain
- Implement server-side token validation to verify request authenticity
- Test protection effectiveness by attempting attacks with and without valid tokens
- Identify limitations of token-based CSRF defense and complementary mitigations

## Lab Environment

- **Target Application:** DVWA running on localhost (http://127.0.0.1:42001)
- **Attack Server:** Python HTTP server hosting malicious HTML (http://127.0.0.1:8000)
- **Test User:** gordonb with initial password "password123"
- **Interception Tool:** Burp Suite for request analysis

## Tools Used

- DVWA (Damn Vulnerable Web Application)
- Python HTTP server (`python -m http.server 8000`)
- Burp Suite (traffic interception and analysis)
- Text editor (creating malicious HTML payload)
- Browser (Firefox for sending requests)

## Methodology

### Exercise 1: Simulating the CSRF Attack

I started by logging into DVWA as a low-privilege user (gordonb) with the password "password123". The CSRF vulnerability page is at http://127.0.0.1:42001/vulnerabilities/csrf/ and allows users to change their password through a simple form.

![DVWA CSRF vulnerability page](screenshots/dvwa_csrf_page.png)



**Attack vector identification:** When a user submits the password change form, DVWA constructs a GET request with all parameters in the URL:

```
http://127.0.0.1:42001/vulnerabilities/csrf/?password_new=abc123&password_conf=abc123&Change=Change#
```

The critical flaw is that this URL, if visited while the user is authenticated, will execute the password change immediately. The server trusts that if the request includes a valid session cookie, it must have come from the legitimate user clicking the form button. This assumption is wrong.

**Crafting the malicious page:** I created a file called `csrf_attack.html` with minimal HTML:

```html
<html>
<body>
<h1>You've won a prize! Click here!</h1>
<img src="http://127.0.0.1:42001/vulnerabilities/csrf/?password_new=abc123&password_conf=abc123&Change=Change#">
</body>
</html>
```

![HTML source code for CSRF attack](screenshots/csrf_attack_html.png)



**Why this works:** The `<img>` tag tells the browser to load an image from the specified URL. Browsers automatically include cookies for the target domain when requesting any resource, including images. When the victim's browser tries to "load" this image, it sends an authenticated GET request to DVWA with the password change parameters. The server sees a valid session cookie and processes the request, not realizing it came from JavaScript on an attacker-controlled site instead of the user clicking the legitimate form.

**Hosting the attack:** I hosted this HTML file on a local web server using Python's built-in HTTP server:

```bash
python -m http.server 8000
```

This is critical because browsers enforce the same-origin policy. A file opened directly from the filesystem (`file:///home/user/csrf_attack.html`) cannot send authenticated requests to web origins like `http://127.0.0.1:42001`. Hosting it on a real HTTP server makes the browser treat it as a proper web page that can make cross-origin requests.

![Malicious page hosted on localhost port 8000](screenshots/csrf_attack_hosted.png)



**Attack execution:** When I loaded http://127.0.0.1:8000/csrf_attack.html in my browser (while still authenticated to DVWA as gordonb), the page displayed "You've won a prize! Click here!" and the browser attempted to load the image. The image request failed because the URL returns HTML, not an image, but the password change request was already sent and processed.



![CSRF attack page displayed in browser](screenshots/csrf_attack_frontend.png)



I logged out of DVWA and attempted to log back in. The old password "password123" no longer worked. The new password "abc123" (from the attack URL) was accepted. The password change succeeded without the user ever clicking the legitimate password change button.

**Business impact:** This attack allows complete account takeover. An attacker sends a victim a link to http://evil.com/prize.html, which changes the victim's password to something the attacker knows. The victim loses access to their account, and the attacker gains full control. In a banking application, this could be used to change the registered email address or phone number, then reset the password through those channels.

### Exercise 2: Implementing Token-Based Protection

Token-based CSRF protection works by requiring each form submission to include a secret value that attackers cannot predict. The server generates a unique token when the page loads, stores it in the user's session, embeds it in the form as a hidden field, and validates it when the form is submitted.

**Server-side validation logic:** I modified the `low.php` configuration file to add token checking before processing password changes. The validation code checks two things:

![Server-side token validation code in low.php](screenshots/csrf_validation_code.png)



```php
if (isset($_GET['user_token']) && $_GET['user_token'] == $_SESSION['session_token']) {
    // Token matches - this is a legitimate request
    $pass_new = $_GET['password_new'];
    $pass_conf = $_GET['password_conf'];
    // Process password change
} else {
    die('That request didn\'t look correct');
}
```

**Why this works:** The `isset()` function verifies that a token was actually submitted. If the request came from the malicious HTML page, there is no `user_token` parameter at all because the attacker has no way of knowing what value to include. Even if they try to guess, the token is a randomly generated string that changes with each page load.

The second check (`$_GET['user_token'] == $_SESSION['session_token']`) compares the submitted token against the value stored in the user's session on the server. If they match, the server knows this request came from someone who loaded the legitimate form page and submitted it. If they do not match (or if no token was submitted), the request is rejected.

**Form modification:** I added the `tokenField()` function call to the form generation code. This function was already implemented for high and impossible security levels, but low security intentionally omitted it to demonstrate the vulnerability. The function generates HTML like this:

```html
<input type="hidden" name="user_token" value="8d9f7a3c1b2e5f6g">
```



![Token generation function added to low security configuration](screenshots/csrf_token_field_code.png)



**Verification:** I inspected the CSRF page source at low security level after making this change. The hidden field now appeared in the form with a randomly generated token value. This field is invisible to the user but automatically submitted with the form.



![Page source showing hidden token field](screenshots/csrf_token_in_source.png)

**Request flow with tokens:**

1. User visits password change page
2. Server generates random token (e.g., "8d9f7a3c1b2e5f6g")
3. Server stores token in user's session: `$_SESSION['session_token'] = "8d9f7a3c1b2e5f6g"`
4. Server embeds token in form as hidden field
5. User clicks "Change" button
6. Browser submits form including the hidden token field
7. Server compares submitted token with session token
8. If match, password change proceeds
9. If no match, request is rejected with error message

**Why attackers cannot bypass this:** When an attacker tricks the victim's browser into loading the malicious image URL, the request does not include the `user_token` parameter. Even if the attacker somehow knew the token value, they cannot inject it into the `<img>` tag because image tags do not support POST data or hidden form fields. The browser only sends the URL exactly as written in the `src` attribute.

**Additional protection mechanisms:**

**SameSite cookies:** Setting the session cookie's SameSite attribute to "Strict" or "Lax" prevents the browser from sending the cookie with cross-site requests. This means the malicious page's image request would not include the authentication cookie, so the server would reject it as unauthenticated even before checking the token.

**Tokens in request body, not cookies:** The token must be transmitted as a form parameter (POST body or GET query string), never as a cookie. If it were a cookie, browsers would automatically include it with cross-site requests, defeating the entire purpose.

### Exercise 3: Testing Protection Effectiveness

To verify the protection actually works, I needed to simulate an attack attempt after implementing token validation. I deliberately removed the `tokenField()` function call from the low security configuration to eliminate the hidden field, then attempted to change the password using the same malicious HTML page.

**Removing token generation:** I commented out the line that generates the token field in the form. After saving the file and refreshing the DVWA page, I inspected the source to confirm the hidden field was gone.



![Modified configuration with tokenField() removed](screenshots/csrf_token_removed_code.png)





![Page source showing no hidden token field](screenshots/csrf_no_token_in_source.png)



**Attack attempt:** I hosted the same malicious HTML file and loaded it while authenticated as gordonb. The browser sent the password change request exactly as before, but this time the request did not include a `user_token` parameter.

**Traffic interception with Burp Suite:** To see exactly what happened, I configured Burp Suite to intercept the request. The GET request showed all the expected parameters (password_new, password_conf, Change) but no user_token.

![Burp Suite showing intercepted request without token](screenshots/burp_no_token_request.png)



**Server response:** I forwarded the request and observed the response. The server returned HTTP 200 (successful page load) but displayed the error message "That request didn't look correct" instead of confirming the password change. This is the custom error message I configured in the validation logic when tokens do not match.



![Server response showing custom error message](screenshots/csrf_error_message.png)



**Verification:** I attempted to log out and log back in. The password was still "password123" (the current password), not "abc123" (the attempted new password). The attack failed. The server rejected the request because the validation logic detected the missing token and stopped processing before reaching the password change code.

![Code showing validation check that triggers error message](screenshots/csrf_validation_error_code.png)



**Successful attack vs. blocked attack:**

| Scenario | Token Present? | Validation Passes? | Password Changed? |
|----------|----------------|-------------------|-------------------|
| Before protection | N/A (no validation) | N/A | Yes (vulnerable) |
| With token (legitimate use) | Yes (correct value) | Yes | Yes (authorized) |
| Without token (attack attempt) | No | No | No (blocked) |

The token validation correctly distinguishes between legitimate user actions and forged requests from attackers.

**Limitations and bypass techniques:**

**XSS defeats CSRF tokens completely:** If an attacker finds a Cross-Site Scripting vulnerability on the target domain, they can inject JavaScript that reads the token directly from the page DOM:

```javascript
var token = document.querySelector('input[name="user_token"]').value;
// Send forged request with stolen token
```

Since XSS runs in the context of the legitimate domain, it has full access to the page content, including hidden fields. The token becomes visible to the attacker, and they can construct a valid request. This is why XSS vulnerabilities are often rated higher severity than CSRF - they bypass many common CSRF protections.

**Token leakage via Referer headers:** If a page containing a CSRF token links to an external website, the browser may include the full URL (including query string parameters) in the HTTP Referer header. If the token is in the URL instead of a hidden field, it leaks to the third-party site:

```
Referer: http://example.com/change_password?user_token=8d9f7a3c1b2e5f6g&password_new=abc123
```

This is why tokens should always be in POST body or hidden fields, never in GET URLs.

**Complementary defense strategies:**

**SameSite cookies (Strict/Lax):** Modern browsers support the SameSite cookie attribute, which controls whether cookies are sent with cross-site requests. Setting it to "Strict" blocks the cookie entirely on cross-site requests. Setting it to "Lax" allows it only on top-level navigation (clicking a link) but not on subresources (images, scripts). This stops the image-based CSRF attack completely because the authentication cookie never gets sent.

**Action re-authentication:** For high-value operations like changing passwords, transferring money, or modifying email addresses, require the user to re-enter their current password or complete a second factor challenge. This makes CSRF significantly harder because the attacker would need to know the user's password, not just trick their browser into sending a request.

## Findings

**CSRF exploits browser trust in authenticated sessions.** Browsers automatically attach cookies to every request for a domain, regardless of whether the request originated from that domain's pages or from a malicious third-party site. This automatic behavior allows attackers to forge requests on behalf of authenticated users.

**GET requests for state-changing operations are inherently vulnerable.** The password change functionality used GET parameters, making it trivial to forge requests through image tags, embedded objects, or crafted links. State-changing operations should always use POST, PUT, or DELETE methods, never GET.

**CSRF tokens provide effective protection when implemented correctly.** Server-side token validation successfully blocked the attack because the malicious request lacked the required token. The validation logic checked for token presence and correctness before processing the password change, preventing unauthorized actions.

**Token validation must happen before any state-changing code executes.** The protection logic in low.php checked the token first, then extracted password parameters only if validation passed. If the code retrieved passwords before checking tokens, it would create a race condition where an attacker might exploit timing or error messages.

**Hidden form fields prevent token theft from external domains.** The token was embedded as a hidden field in the legitimate form, making it inaccessible to JavaScript running on attacker-controlled domains. Cross-origin security policies prevent external scripts from reading DOM content from other origins.

**XSS vulnerabilities completely bypass CSRF token protection.** If an attacker can inject JavaScript into the target domain through XSS, they can read the token from the page and construct valid requests. This makes XSS prevention critical for CSRF defenses to work.

**SameSite cookies add browser-level protection independent of tokens.** Setting session cookies to SameSite=Strict prevents the browser from sending authentication cookies with cross-site requests. This stops CSRF attacks at the browser level before they reach the server, providing defense-in-depth.

## Challenges Faced

**Understanding why hosting on HTTP server was necessary:** Initially, I tried opening csrf_attack.html directly from the filesystem (`file:///`). The attack did not work because browsers block cross-origin requests from file:// origins to http:// origins as a security measure. Only after hosting the file on a real HTTP server did the browser treat it as a legitimate web page capable of making requests.

**Confirming token presence through inspection:** The hidden token field is invisible in the rendered page, so I had to use browser developer tools to inspect the HTML source. This taught me that not all security controls are visible to users - many operate silently in the background.

**Distinguishing between failed page load and failed validation:** When the attack was blocked, the HTTP response was still 200 OK, not an error code like 403 Forbidden. I initially thought the attack might have succeeded because the page loaded. Only by reading the response body did I see the "That request didn't look correct" message, confirming the validation logic rejected the request.

## Key Takeaways

**CSRF attacks exploit implicit trust, not technical vulnerabilities in code.** The vulnerability is not a bug in DVWA's password change logic. The code works exactly as designed. The problem is that browsers automatically send cookies, and servers trust requests with valid cookies. The attack exploits this trust relationship between browsers and servers.

**Defense requires proving request intent, not just authentication.** Session cookies prove the user is authenticated, but CSRF tokens prove the request originated from a legitimate form submission. Authentication alone is insufficient for state-changing operations.

**Token-based protection is industry standard for good reason.** Almost every web framework includes built-in CSRF protection using tokens. Django, Rails, ASP.NET, and Laravel all generate and validate tokens automatically. Understanding how they work helps developers avoid breaking the protection through custom code.

**Multiple defense layers provide better security than any single control.** Combining CSRF tokens with SameSite cookies, re-authentication for sensitive actions, and XSS prevention creates defense-in-depth. If one control fails, others can still prevent the attack.

**GET requests should never change state.** Using GET for password changes, funds transfers, or account modifications makes CSRF attacks trivial. POST requests cannot be forged through image tags or embedded objects, forcing attackers to use more complex techniques that are easier to detect and block.

**Testing protection effectiveness is as important as implementing it.** Just adding a tokenField() call is not enough. I had to verify the token appears in the form, the validation logic rejects missing tokens, and legitimate requests still work. Many security controls fail because they are implemented but never tested.

## Disclaimer

This lab was performed in a controlled environment using DVWA (Damn Vulnerable Web Application), which is specifically designed for security training and intentionally contains exploitable vulnerabilities. All testing was conducted on locally hosted applications with no involvement of external systems or unauthorized access.