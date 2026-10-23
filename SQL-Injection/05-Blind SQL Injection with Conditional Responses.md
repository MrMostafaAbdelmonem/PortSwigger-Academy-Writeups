```markdown
# Lab: Blind SQL Injection with Conditional Responses

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Burp Suite Embedded Browser (Chromium)
* **Tools:** Burp Suite (Proxy, Repeater, Intruder)
* **Target Application:** E-Commerce Web Application (Tracking Cookie Vector)

---

## Vulnerability Overview & Technical Concept

In **Blind SQL Injection** scenarios, the application does not explicitly reflect database query results or syntax errors within the HTTP response body. However, the application's underlying logic or presentation layers may change based on whether an injected SQL conditional expression evaluates to `TRUE` or `FALSE` (e.g., toggling the presence of a `"Welcome back!"` message).

By leveraging logical operators (`OR`) and subqueries combined with character comparison functions (such as `SUBSTRING` and length checks), an attacker can systematically infer and extract sensitive database contents—such as administrator passwords—character by character.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Endpoint & Vector Identification via Burp Suite
1. Launched Burp Suite and configured the browser to intercept traffic to the target lab environment.
2. Navigated to the application and observed a standard e-commerce interface featuring a persistent `"Welcome back!"` indicator tied to session or tracking parameters.
3. Inspected the HTTP request history and isolated the `TrackingId` cookie:

```http
GET / HTTP/1.1
Host: target-app.web-security-academy.net
Cookie: TrackingId=xyz123; session=abc...

```

4. **Sanitization Assessment:** Altering or truncating characters within the `TrackingId` cookie caused the application state to shift (the `"Welcome back!"` message disappeared). This indicated that unmitigated user input is concatenated directly into the backend SQL query logic without proper prepared statements or robust input sanitization, leading to application behavior shifts instead of clean error handling.

---

### Step 2: Validating the Injection Vector (`TRUE` / `FALSE` Logic)

To confirm injection viability, injected an intentional boolean tautology into the `TrackingId` value:

* **Payload:** `x' OR 'a'='a`
* **HTTP Request Structure:**

```http
GET / HTTP/1.1
Host: target-app.web-security-academy.net
Cookie: TrackingId=x' OR 'a'='a; session=abc...

```

* **Response:** `HTTP 200 OK` accompanied by the re-appearance of the `"Welcome back!"` message. This confirmed that the backend query evaluates injected conditions and dynamically alters application responses based on boolean states.

---

### Step 3: Determining Administrator Password Length

To extract the administrator's password, constructed a subquery targeting the `users` table to evaluate password length iteratively:

```sql
x'OR (SELECT 'a' FROM users WHERE username='administrator' AND length(password) = 12 )='a

```

#### Detailed Subquery Mechanics:

* **Subquery Structure:** `(SELECT 'a' FROM users WHERE username='administrator' AND length(password) = 12)` executes an independent inner query. If the target table and username exist and match the specified condition, it returns `'a'`.
* **Logical Evaluation:** The expression compares the subquery result against `'a'` (`... = 'a'`). If true, the full statement equates to a valid condition, triggering the `"Welcome back!"` response.

#### Automation via Burp Intruder:

1. Sent the request to **Burp Intruder**, highlighted the integer value (`12`), and designated it as the payload position.
2. Applied **URL Encoding** (`Ctrl + U`) to ensure special characters like spaces and quotes were properly parsed.
3. Configured the payload type as **Numbers**, setting the range from `5` to `30` to optimize enumeration time within Burp Suite Community Edition limits.
4. Executed the attack and filtered responses by length/content changes. The unique response size variance isolated length **`20`**, confirming the administrator password is **20 characters** long.

---

### Step 4: Enumerating the Password Character by Character

With the password length established, constructed a character-extraction payload utilizing the database `SUBSTRING` function:

```sql
x'OR (SELECT substring(password,1,1) FROM users WHERE username='administrator')='a

```

#### Detailed Payload Mechanics:

* `substring(password, 1, 1)`: Extracts a substring from the `password` column where the first parameter is the target column, the second parameter is the starting character index position, and the third parameter specifies the length of characters to extract (1 character).
* Comparing this extracted character against a test character (`='a'`) allows boolean inference of each position sequentially.

#### Intruder Attack Configuration (Cluster Bomb Strategy):

To automate multi-position and character bruteforcing efficiently:

1. Highlighted the index position (first `1`) and the target comparison character (`a`) as two separate payload positions.
2. Selected the **Cluster Bomb** attack type in Burp Intruder:
* **Payload Set 1 (Index Position):** Configured as **Numbers** ranging from `1` to `20`.
* **Payload Set 2 (Character Brute-force):** Configured with a character set matching alphabet and numeric options (`a-z`, `0-9`).


3. *Note for Community Edition Users:* Due to request volume limitations in the community release, enumeration can be executed in batches (e.g., splitting index ranges from 1–5, 6–10, etc.) to manage processing time effectively.

---

## Proof of Concept (PoC) & Exploitation Result

* **Extracted Data:** Sequentially mapped and reconstructed the complete 20-character plaintext password for the `administrator` user account via conditional response inference.
* **Impact:** Full compromise of administrative access privileges through blind boolean-based SQL injection.
* **Lab Resolution:** Authenticated successfully on the web application using the recovered administrator credentials, resolving the lab.

---

## Remediation & Mitigation

1. **Implement Parameterized Queries (Prepared Statements):**
Ensure all user-supplied input parameters—including HTTP headers and cookies like `TrackingId`—are handled strictly as data bindings rather than executable query string components.
2. **Generic Error Handling & State Consistency:**
Standardize application responses and UI messages (such as greeting banners) regardless of backend query execution states to eliminate side-channel data leakage.

```

```
