# Introduction to Web Technologies and Understanding HTTP and HTTPS

## Overview

This lab explored the fundamental building blocks of web communication by creating a basic HTML webpage and analyzing the differences between HTTP and HTTPS traffic. The goal was to understand how browsers and servers communicate, what information gets exposed in unencrypted connections, and how TLS certificates protect against eavesdropping and man-in-the-middle attacks.

## Objectives

- Build a basic HTML webpage to understand document structure and semantic elements
- Use browser developer tools to inspect HTTP request and response headers
- Analyze the security implications of exposed metadata in HTTP traffic
- Compare HTTP and HTTPS headers to identify encryption-specific parameters
- Investigate TLS certificate details including issuer, validity period, and encryption methods
- Explain how HTTPS protects confidentiality, integrity, and authentication

## Lab Environment

- **Development Machine:** Standard workstation with web browser (Chrome/Firefox)
- **Local HTML File:** Created and tested offline to understand basic HTML structure
- **HTTP Target:** Local file accessed via file:// protocol
- **HTTPS Target:** BBC News website (https://www.bbc.com) for secure connection analysis

## Tools Used

- Text editor (for HTML creation)
- Browser Developer Tools (Network tab for traffic inspection)
- Browser Security Panel (for TLS certificate examination)

## Methodology

### Exercise 1: Creating a Basic HTML Webpage

I started by building a simple HTML page to understand how browsers interpret document structure. The page needed a title, header, paragraph, image, and external link.

![HTML source code for basic webpage](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/34_01%20HTML%20source%20code%20for%20basic%20webpage.jpg)

**Document structure breakdown:**

- The `<!DOCTYPE html>` declaration tells the browser this is an HTML5 document, not an older version with different parsing rules. The `<html lang="en">` tag sets the language to English, which screen readers use to pronounce words correctly.

- Inside the `<head>` section, the `<meta charset="UTF-8">` tag ensures the browser interprets text using the UTF-8 character set, preventing garbled characters when displaying non-ASCII symbols. The `<meta name="viewport">` tag makes the page responsive on mobile devices by setting the initial zoom level to match the screen width.

- The `<title>` tag defines what appears in the browser tab and bookmark name. This is separate from the visible `<h1>` heading, which appears on the actual page.

- In the `<body>` section, the `<h1>` tag creates the main heading. Without this semantic structure, the browser would render all text identically. The `<p>` paragraph tags separate blocks of text visually and logically. Screen readers announce "paragraph" before reading the content, giving visually impaired users context.

- The `<img>` tag embeds an image using a URL. The `src` attribute points to the image location, which can be local or remote. If the image fails to load, the browser has no fallback without an `alt` attribute.

- The `<a>` tag creates a hyperlink. The `href` attribute defines the destination URL. When clicked, the browser sends a GET request to that URL and loads the response. Without links, the web would be a collection of isolated documents with no navigation.

![Rendered HTML webpage showing header, paragraph, image, and link](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/34_02%20Rendered%20HTML%20webpage%20showing%20header%2C%20paragraph%2C%20image%2C%20and%20link.jpg)

**Why semantic structure matters:**

- Browsers need tags to distinguish headings from body text. Without `<h1>`, a browser displays "Introduction to Web Technologies Course" in the same font and size as the paragraph below it. Users cannot visually identify the hierarchy.

- Links enable navigation. Before hyperlinks, users had to manually type every URL. The `<a>` tag with `href` lets browsers handle navigation automatically, creating the interconnected web we rely on today.

- Accessibility depends on semantic HTML. Screen readers announce "heading level 1" when encountering `<h1>`, then "paragraph" for `<p>`. This structure helps visually impaired users understand page organization without seeing it.

### Exercise 2: Analyzing HTTP Request and Response Headers

To see what information browsers send and receive during normal web traffic, I opened a local HTML file and inspected the HTTP headers using browser developer tools.

![Developer tools showing HTTP request headers](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/34_03%20Developer%20tools%20showing%20HTTP%20request%20headers.jpg)

**Request analysis:**

- **Request Method:** GET (retrieving a resource, not modifying server state)
- **Status Code:** 200 OK (server successfully returned the requested page)
- **Referrer Policy:** strict-origin-when-cross-origin (browser only sends the origin, not the full URL, to external sites)

**Response headers:**

- **Content-Type:** text/html (tells the browser to render the response as an HTML document)
- **Last-Modified:** Wed, 20 May 2026 16:12:13 GMT (timestamp of when the file was last changed)

**User-Agent string breakdown:**

The User-Agent header reveals extensive client-side information. In this case: `Mozilla/5.0 (Linux; Android 6.0; Nexus 5 Build/MRA58N) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/148.0.0.0 Mobile Safari/537.36`

This exposes:
- Operating system: Linux (Android 6.0)
- Device model: Nexus 5
- Browser engine: AppleWebKit 537.36
- Browser: Chrome 148.0.0.0
- Form factor: Mobile

**Why understanding headers matters for security:**

- HTTP is stateless. Every request is independent. Authentication, authorization, and data integrity must be explicitly enforced through headers and cookies because the protocol itself provides no memory between requests.

- Exposed version information aids reconnaissance. The User-Agent string reveals the exact browser version. An attacker can search CVE databases for known exploits targeting Chrome 148.0.0.0 specifically. Servers should not trust client-provided version strings for access control because they are trivially spoofed.

**Vulnerabilities from improper header handling:**

- **Broken Object Level Authorization (BOLA/IDOR):** If the server trusts the `id` parameter in a URL like `/api/users?id=123` without checking whether the authenticated user owns that resource, an attacker can change `id=123` to `id=124` and access another user's data.

- **Command Injection:** If the server passes User-Agent strings or other headers directly to shell commands without sanitization, an attacker can inject commands. For example, a User-Agent of `; rm -rf /` would execute shell code if concatenated into a bash command.

- **Cross-Site Scripting (XSS):** If the server echoes headers like Referer or User-Agent back into HTML responses without escaping, an attacker can inject JavaScript that executes in the victim's browser.

- **Session Fixation and Hijacking:** If session cookies lack the `HttpOnly` and `Secure` flags in response headers, JavaScript can read the session ID, and attackers can steal it via XSS. Without the `SameSite` flag, cross-site requests can hijack authenticated sessions.

### Exercise 3: Understanding HTTPS and Certificate Validation

To compare secure traffic, I navigated to https://www.bbc.com and inspected the HTTPS-specific headers and TLS certificate details.

![HTTPS request headers showing :scheme, Accept-Encoding, and Origin](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/34_04%20HTTPS%20request%20headers%20showing%20scheme%2C%20Accept-Encoding%2C%20and%20Origin.jpg)

**HTTPS-specific headers:**

- **:scheme: https** - This is an HTTP/2 pseudo-header (prefixed with a colon). It explicitly tells the server the request arrived over an encrypted TLS connection. This header is required in HTTP/2, while HTTP/1.1 relies on the connection context to determine encryption state.

- **Accept-Encoding: gzip, deflate, br, zstd** - The browser advertises compression algorithms it understands. If the server compresses the response with Brotli (br) or Zstandard (zstd), the browser can decompress it on the client side. This reduces bandwidth usage and improves page load speed without changing the underlying data.

- **Origin: https://www.bbc.com** - This header is critical for Cross-Origin Resource Sharing (CORS) security. When JavaScript running on one domain (e.g., attacker.com) tries to make a request to a different domain (e.g., bbc.com), the browser includes the Origin header. The server checks this value and decides whether to allow the cross-origin request based on its CORS policy. If the server does not explicitly whitelist the origin, the browser blocks the response.

- **TLS certificate inspection:**

![TLS certificate details showing issuer, validity period, and SHA-256 fingerprint](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/34_05%20TLS%20certificate%20details%20showing%20issuer%2C%20validity%20period%2C%20and%20SHA-256%20fingerprint.jpg)

- **Certificate Issuer:** GlobalSign RSA OV SSL CA 2018 (a trusted Certificate Authority)
- **Issued To:** www.bbc.com (confirms the certificate matches the domain)
- **Organization:** British Broadcasting Corporation
- **Validity Period:** Friday, June 27, 2025 to Monday, July 27, 2026 (1 year certificate lifespan)
- **Encryption Method:** SHA-256 (cryptographic hash function for certificate fingerprint)

**Certificate chain validation:**

- Browsers do not trust certificates blindly. When connecting to www.bbc.com, the browser receives the site's certificate. This certificate is not self-signed; it is signed by GlobalSign RSA OV SSL CA 2018, which is an intermediate Certificate Authority.

- The browser then checks whether GlobalSign's intermediate CA certificate is signed by a root CA in its trusted store. If the chain of trust leads back to a recognized root CA (e.g., GlobalSign Root CA), the browser accepts the certificate. If any link in the chain is missing or invalid, the browser displays a security warning.

**How HTTPS prevents eavesdropping:**

- When the browser connects to www.bbc.com over HTTPS, it negotiates a symmetric encryption key through the TLS handshake. All subsequent data (HTTP headers, request bodies, response bodies) is encrypted with this session key using algorithms like AES-256.

- If an attacker intercepts the traffic with a packet sniffer, they see encrypted ciphertext. Without the session key, the attacker cannot decrypt it. The session key never travels over the network in plaintext. It is derived through asymmetric cryptography (the server's public key encrypts a pre-master secret, which both sides use to derive the session key).

**How HTTPS prevents man-in-the-middle attacks:**

- An attacker cannot impersonate www.bbc.com without a valid certificate. If the attacker tries to intercept the connection and present a fake certificate, the browser checks the certificate's cryptographic signature.

- The signature is generated by GlobalSign's private key. The attacker does not have GlobalSign's private key, so they cannot create a valid signature. When the browser attempts to verify the signature using GlobalSign's public key (embedded in the intermediate CA certificate), the verification fails.

- The browser displays a warning: "Your connection is not private" or "Certificate error." The user sees the attack attempt and can terminate the connection before sending any data.

**HTTPS guarantees three security properties:**

- **Confidentiality:** Symmetric encryption (AES) prevents eavesdroppers from reading data in transit.

- **Integrity:** Message Authentication Codes (MACs) detect tampering. If an attacker modifies encrypted data, the MAC verification fails, and the browser discards the response.

- **Authentication:** TLS certificates prove the server's identity. The browser verifies the certificate chain back to a trusted root CA, ensuring the connection goes to the legitimate server and not an imposter.

## Findings

- **HTML semantic structure controls how browsers and assistive technologies interpret content.** Tags like `<h1>`, `<p>`, and `<a>` are not just visual formatting. They provide meaning. Screen readers announce heading levels, search engines rank pages based on header hierarchy, and browser reader modes extract content by recognizing semantic tags. Without this structure, a webpage is just undifferentiated text.

- **HTTP headers expose extensive client and server metadata.** The User-Agent string reveals operating system, device model, browser version, and rendering engine. The Server header discloses web server software and version. This information aids attackers during reconnaissance. Knowing a server runs nginx 1.19.0 lets an attacker search for known CVEs targeting that specific version.

- **HTTP is stateless, so security must be explicitly enforced in every request.** Authentication, authorization, and data validation cannot rely on "remembering" previous requests. Every HTTP transaction must carry its own security context through headers (e.g., Authorization tokens) and cookies (e.g., session IDs). Failure to validate these on every request creates Broken Object Level Authorization vulnerabilities.

- **HTTPS adds encryption-specific headers that HTTP lacks.** The `:scheme` pseudo-header in HTTP/2 explicitly declares the connection is encrypted. The `Origin` header enforces CORS policies to prevent malicious cross-origin requests. The `Accept-Encoding` header negotiates compression algorithms to reduce bandwidth without compromising security.

- **TLS certificates create a chain of trust from the root CA to the server.** Browsers ship with a list of trusted root CAs. When connecting to an HTTPS site, the browser verifies the server's certificate is signed by a trusted CA. This prevents attackers from impersonating legitimate sites unless they compromise a CA's private key, which is extremely difficult.

- **HTTPS encrypts the entire HTTP conversation, not just the request body.** Request and response headers, cookies, and all payload data are encrypted with the session key. An eavesdropper sees encrypted ciphertext and TLS handshake messages, but cannot read URLs, form data, or API responses.

- **Certificate validity periods limit the impact of compromised certificates.** The BBC certificate expires after one year. If the certificate's private key is stolen, the attacker can only impersonate the site until expiration. Certificate Revocation Lists (CRLs) and Online Certificate Status Protocol (OCSP) allow CAs to invalidate compromised certificates before expiration.

- **SHA-256 fingerprints uniquely identify certificates.** The fingerprint is a cryptographic hash of the entire certificate. If even a single byte changes (e.g., an attacker modifies the Common Name), the hash changes completely. Browsers use this to detect certificate substitution attacks.

## Challenges Faced

- **Understanding HTTP/2 pseudo-headers:** The `:scheme` header confused me initially because it starts with a colon, unlike standard headers. I learned that HTTP/2 introduced pseudo-headers to replace HTTP/1.1's request line (e.g., `GET /path HTTP/1.1`). These pseudo-headers (`:method`, `:scheme`, `:authority`, `:path`) must appear at the start of the header block and define request metadata.

- **Distinguishing between symmetric and asymmetric encryption in TLS:** I initially thought HTTPS encrypted all traffic with the server's public key. After researching, I learned TLS uses asymmetric cryptography (RSA/ECDHE) only during the handshake to securely exchange a symmetric session key. The session key (AES-256) encrypts the actual data because symmetric encryption is much faster than asymmetric for large volumes of traffic.

- **Certificate chain validation mechanics:** I did not understand how the browser verifies certificates without contacting the CA every time. I discovered browsers have a local trust store containing root CA certificates. When the server sends its certificate, the browser checks the signature using the CA's public key from the trust store. This works offline because the root CA's public key is pre-installed.

## Key Takeaways

- **HTML tags provide semantic meaning, not just visual formatting.** Browsers use tags to render pages, but screen readers, search engines, and reader modes rely on semantic structure to understand content. Using `<div>` instead of `<h1>` loses this meaning.

- **HTTP headers are a reconnaissance goldmine for attackers.** User-Agent, Server, and X-Powered-By headers expose version information. Attackers search CVE databases for known exploits targeting these exact versions. Production servers should minimize version disclosure.

- **Stateless HTTP requires explicit security on every request.** Unlike protocols with persistent connections, HTTP does not remember authentication state. Every request must include credentials (cookies, tokens) and every response must validate them. Failing to check authorization on a single endpoint creates an IDOR vulnerability.

- **HTTPS is not optional for security.** Unencrypted HTTP exposes passwords, session tokens, and sensitive data to anyone monitoring network traffic. Even on trusted networks (home Wi-Fi), attackers can intercept traffic. HTTPS should be the default for all web applications.

- **Certificate validation prevents impersonation attacks.** Without TLS certificates, an attacker on the network can intercept connections and present a fake server. The browser cannot distinguish the imposter from the legitimate server. Certificates create a cryptographic chain of trust that attackers cannot forge without compromising a Certificate Authority.

- **Encryption protects confidentiality, MAC protects integrity, certificates protect authentication.** HTTPS is not just encryption. It provides three security guarantees. Attackers cannot read encrypted data (confidentiality), cannot modify data without detection (integrity via MAC), and cannot impersonate the server (authentication via certificates).

- **Validity periods limit certificate compromise impact.** Short-lived certificates (90 days or less) reduce the window of opportunity if a private key is stolen. Even if an attacker obtains the key, it expires quickly. This is why Let's Encrypt issues 90-day certificates and encourages automation.

## Disclaimer

This lab was conducted using publicly accessible websites (BBC News) and locally created HTML files for educational purposes. No unauthorized access was attempted, and all traffic analysis was performed on my own browser traffic. The HTTPS analysis examined standard TLS certificate information that is publicly visible to all users.
