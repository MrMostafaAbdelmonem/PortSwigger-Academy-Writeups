```markdown
# Lab: OS Command Injection, Simple Case

**Platform:** PortSwigger Web Security Academy  
**Category:** OS Command Injection  
**Difficulty:** Apprentice / Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Google Chrome
* **Proxy & Tools:** Burp Suite (Proxy, Repeater) with FoxyProxy Extension
* **Target Application:** E-Commerce Web Application (Check Stock Feature)

---

## Vulnerability Overview & Technical Concept

**OS Command Injection** occurs when a web application passes unsafe user-supplied data (such as form fields, parameters, or headers) directly to a system shell or command interpreter. If the input lacks proper sanitization or validation, an attacker can append shell metacharacters (like `|`, `;`, or `&`) to execute arbitrary operating system commands with the privileges of the web application service.

In Linux environments, the pipe operator (`|`) redirects the standard output of one command to become the standard input of the next command. When injected into a vulnerable backend system call, it allows a secondary malicious command (such as `whoami`) to execute immediately after or alongside the primary application process.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Intercepting the Request via Burp Suite
1. Configured **FoxyProxy** in Google Chrome to route traffic through **Burp Suite**.
2. Opened the lab environment, navigated to a specific product page, and clicked **"View Details"**.
3. Located the stock availability feature featuring a country selector dropdown (e.g., selecting "London") and clicked the **"Check Stock"** button.
4. With Burp Intercept enabled, captured the resulting HTTP **POST** request.
5. Inspected the request body and identified the parameters passed to the backend endpoint:

```http
POST /product/stock HTTP/1.1
Host: target-app.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

productId=2&storeId=1

```

6. Right-clicked the intercepted request and sent it to **Burp Repeater** for further analysis.

---

### Step 2: Injecting the OS Command

Appended the pipe operator (`|`) followed by the system command `whoami` directly to the `storeId` parameter value within Burp Repeater:

* **Injected Parameter:** `productId=2&storeId=1|whoami`
* **Full POST Body:**

```http
POST /product/stock HTTP/1.1
Host: target-app.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

productId=2&storeId=1|whoami

```

#### Detailed Payload Mechanics:

* `storeId=1`: The normal application parameter specifying the target store inventory ID.
* `|`: The shell pipe operator. In vulnerable backend scripts, this instructs the operating system shell to execute the subsequent command.
* `whoami`: A standard Linux utility that prints the username associated with the current effective user ID running the web process (e.g., `peter-vvlYFY`).

---

## Proof of Concept (PoC) & Exploitation Result

* **Server Response:** Executing the request returned the output of the injected command (`peter-vvlYFY`) directly in the HTTP response body where stock figures or error messages would normally render.
* **Impact:** Successful execution of arbitrary OS-level commands, demonstrating lack of input validation and direct shell command construction.
* **Lab Resolution:** The lab successfully registered as completed upon rendering the output of the `whoami` command.

---

## Remediation & Mitigation

1. **Avoid Invoking OS Commands Directly:**
Whenever possible, use built-in application programming interfaces (APIs) or language libraries to handle system-level tasks instead of spawning shell processes.
2. **Input Validation and Whitelisting:**
Enforce strict input validation using positive regex whitelists (e.g., ensuring `storeId` accepts strictly integer values).
3. **Use Parameterized Execution APIs:**
If operating system commands must be executed, use APIs that pass arguments as an array/list rather than a single concatenated string shell command, neutralizing metacharacter interpretation.

```

```
