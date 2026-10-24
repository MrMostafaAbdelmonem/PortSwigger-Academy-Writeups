```markdown
# Lab: Blind SQL Injection with Time Delays

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Burp Suite Embedded Browser (Chromium)
* **Tools:** Burp Suite (Proxy, Repeater)
* **Target Application:** E-Commerce Web Application (Tracking Cookie Vector)

---

## Vulnerability Overview & Technical Concept

When a web application is entirely "blind"—meaning it returns identical HTTP responses and user interfaces regardless of whether an injected SQL query evaluates to true, false, or causes a database error—conditional responses or error messages cannot be used to infer data. 

To bypass this limitation, **Time-Based Blind SQL Injection** is utilized. By injecting database-specific functions that force a deliberate processing delay (such as `pg_sleep()` in PostgreSQL), an attacker can measure the time it takes for the HTTP response to return. If the application's response is delayed by the specified duration, the injected query execution is confirmed.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Intercepting the Request via Burp Suite
1. Launched Burp Suite and enabled **Intercept** in the Proxy tab.
2. Navigated through the lab application (such as the login or account pages) using Burp's embedded browser.
3. Captured the incoming HTTP request and right-clicked to send it directly to **Burp Repeater** for payload testing.
4. Identified the injection vector within the session parameters or tracking headers (e.g., cookies or URL parameters).

---

### Step 2: Testing and Executing the Time-Delay Payload

Tested various DBMS-specific sleep commands to determine the underlying database engine. For PostgreSQL environments, string concatenation combined with the `pg_sleep()` function successfully triggered a delay:

* **Payload:** `x'|| (SELECT pg_sleep(10)) ||'`
* **HTTP Request Context:**
```http
GET / HTTP/1.1
Host: target-app.web-security-academy.net
Cookie: TrackingId=x'|| (SELECT pg_sleep(10)) ||'; session=abc...

```

#### Detailed Payload Mechanics:

* `x'`: Closes out the initial string parameter context safely.
* `||`: String concatenation operator used in SQL (particularly effective in PostgreSQL) to chain query results.
* `(SELECT pg_sleep(10))`: Executes a procedural function that forces the database server to sleep for 10 seconds.
* **URL Encoding:** Highlighted the payload characters and applied URL encoding (`Ctrl + U`) to ensure spaces and special characters (`+`, `'`, `||`) were transmitted correctly without breaking HTTP syntax.

---

## Proof of Concept (PoC) & Exploitation Result

* **Server Response Time:** Executing the crafted request caused the HTTP response to hang for exactly **10 seconds** before returning `HTTP 200 OK`.
* **Impact:** Confirmed backend execution of arbitrary SQL functions and established a reliable time-delay side-channel to infer database states.
* **Lab Resolution:** The lab registered as successfully completed upon triggering the verified 10-second backend delay.

---

## Remediation & Mitigation

1. **Implement Parameterized Queries (Prepared Statements):**
Ensure all dynamic user inputs, including cookies and headers, are strictly parameterized so they are never interpreted as executable code or database commands.
2. **Query Execution Timeout Limits:**
Configure strict execution timeouts on the database connection layer to prevent resource exhaustion and mitigate time-delay-based blind extraction attacks.

```

```
