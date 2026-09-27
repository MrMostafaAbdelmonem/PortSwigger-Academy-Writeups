```markdown
# Lab: SQL Injection Attack, Querying the Database Type and Version on MySQL and Microsoft

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Burp Suite Embedded Browser (Chromium)
* **Target Application:** E-Commerce Web Application

---

## Vulnerability Overview & Technical Concept

Database Fingerprinting allows an attacker to identify the underlying Database Management System (DBMS) engine (e.g., MySQL, Microsoft SQL Server, PostgreSQL, Oracle). Identifying the precise database type enables executing database-specific system functions to retrieve sensitive version metadata.

On both **MySQL** and **Microsoft SQL Server (MSSQL)**, version data is stored within the global system variable `@@version`.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Endpoint Identification
Navigated to the application homepage via Burp Suite's embedded browser and selected the **"Pets"** category filter. The request structure in the URL bar was observed:

```http
[https://target-app.web-security-academy.net/filter?category=Pets](https://target-app.web-security-academy.net/filter?category=Pets)

```

### Step 2: Determining Column Count (`NULL` Injection)

To construct a valid `UNION` attack, the exact number of columns selected by the original query must be determined using `NULL` placeholders.

#### A. Single Column Test:

Appended `' UNION SELECT NULL-- -` to the URL parameter:

```http
/filter?category=Pets'+UNION+SELECT+NULL--+-

```

* **Response:** `HTTP 500 Internal Server Error`
* **Analysis:** Column count mismatch between the injected query and original query.

#### B. Two Columns Test:

Appended a second `NULL` placeholder:

```http
/filter?category=Pets'+UNION+SELECT+NULL,NULL--+-

```

* **Response:** `HTTP 200 OK`
* **Outcome:** The web page rendered normally without database syntax errors, confirming the backend query expects **exactly two columns**.

---

### Step 3: Database Version Fingerprinting

Referencing the SQL Injection Cheat Sheet confirmed that `@@version` is the unified syntax for extracting version strings on **MySQL** and **Microsoft SQL Server**.

Substituted one `NULL` placeholder with `@@version` to retrieve the database version string:

```http
/filter?category=Pets'+UNION+SELECT+NULL,@@version--+-

```

#### Detailed Payload Mechanics:

* `'`: Terminated the original string parameter context (`Pets`).
* `+`: Serves as URL-encoded representation for space characters (`%20`).
* `UNION SELECT`: Merges the results of the injected version query with the category search output.
* `NULL, @@version`: Matches the required two-column structure while executing the database version extraction string.
* `--+-`: SQL comment sequence (`--` followed by a space and arbitrary trailing character) to ignore all remaining backend SQL query logic.

---

## Proof of Concept (PoC) & Exploitation Result

* **Extracted Data:** Executing the crafted GET request rendered the database version details directly within the product listing area (e.g., `8.0.28-0ubuntu0.20.04.3`).
* **Impact:** Successful fingerprinting of the backend DBMS, enabling targeted post-exploitation steps.
* **Lab Resolution:** The lab successfully registered as completed upon rendering the server version output.

---

## Remediation & Mitigation

1. **Use Parameterized Queries (Prepared Statements):**
Ensures user input is handled strictly as data parameters rather than executable SQL code, preventing context escaping.
```java
String query = "SELECT * FROM products WHERE category = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, userCategory);
ResultSet results = stmt.executeQuery();

```


2. **Database Hardening & Least Privilege:**
Restrict web application database account permissions to prevent unnecessary access to system variables and administrative functions.

```

---

```
