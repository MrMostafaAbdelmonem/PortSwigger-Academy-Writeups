```markdown

# Lab: SQL Injection Vulnerability in WHERE Clause Allowing Retrieval of Hidden Data

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Apprentice  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Google Chrome (configured with **FoxyProxy**)
* **Interception Proxy:** Burp Suite Community Edition
* **Target Application:** E-Commerce Web Application

---

## Technical Overview & Vulnerability Analysis

The web application provides product filtering based on categories. When a user selects a specific category, the web backend executes a dynamic SQL query to fetch relevant inventory data.

### Expected Backend Query Structure:
```sql
SELECT * FROM products WHERE category = 'Gifts' AND released = 1

```

### Flaw Identification:

The application developer directly trusts user-supplied input via the `category` HTTP GET parameter without sanitization or input parameterization. This flaw allows an attacker to break out of the string literal context and inject arbitrary SQL clauses.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Intercepting the HTTP Request

1. Configured proxy traffic routing in Google Chrome using **FoxyProxy**.
2. Launched **Burp Suite**, navigated to `Proxy -> Intercept`, and ensured **Intercept is on**.
3. Navigated to the application homepage and clicked on the **"Gifts"** category filter.
4. Captured the generated GET request in Burp Suite:

```http
GET /filter?category=Gifts HTTP/1.1
Host: target-app.web-security-academy.net

```

### Step 2: Testing for Syntax Breaking & Fault Injection

To verify the lack of sanitization, a single quote (`'`) was appended to the `category` parameter value:

```http
GET /filter?category=Gifts' HTTP/1.1

```

* **Observation:** The database query broke due to mismatched single quotes:
```sql
SELECT * FROM products WHERE category = 'Gifts'' AND released = 1

```


This triggered an unhandled application error / missing product response, confirming raw SQL reflection.

### Step 3: Constructing the Logic-Bypass Payload

To force the database engine to evaluate the `WHERE` clause to boolean `TRUE` for all table records—bypassing the `released = 1` condition—a boolean SQL injection payload was crafted:

* **Payload:** `' OR 1=1--`

#### Detailed Payload Mechanics:

1. `'` (Single Quote): Properly terminates the string literal for `'Gifts'`.
2. `OR 1=1`: Introduces a logical boolean expression that always evaluates to **`TRUE`**.
3. `--` (SQL Comment): Instructs the SQL compiler to treat all remaining query code as a comment, successfully neutralizing the trailing business logic check (`AND released = 1`).

### Step 4: Payload Execution & Execution Flow

Forwarded the crafted request with the URL-encoded payload through Burp Suite:

```http
GET /filter?category=Gifts'+OR+1=1-- HTTP/1.1

```

#### Final Executed Backend Query:

```sql
SELECT * FROM products WHERE category = 'Gifts' OR 1=1--' AND released = 1

```

---

## Proof of Concept (PoC) & Impact

* **Outcome:** Since `1=1` evaluates to `TRUE` universally, the backend query returned every row from the `products` table regardless of category assignment or release state.
* **Impact:** Complete bypass of access control and logic filters, resulting in unauthorized information disclosure of hidden/unreleased products.

---

## Remediation & Mitigation

1. **Implement Parameterized Queries (Prepared Statements):**
Pre-compile SQL statements so user inputs are handled exclusively as raw data parameters rather than executable database syntax.
```java
String query = "SELECT * FROM products WHERE category = ? AND released = 1";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, userCategory);
ResultSet results = stmt.executeQuery();

```


2. **Input Validation & Allowlisting:**
Enforce standard input validation to ensure input parameters strictly match expected, pre-approved category identifiers.

```

---
