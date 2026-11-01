```markdown
# Lab: Blind OS Command Injection with Time Delays

**Platform:** PortSwigger Web Security Academy  
**Category:** OS Command Injection  
**Difficulty:** Practitioner  
**Author:** Mostafa Abdelmonem  

---

## Environment & Tools Used
* **Browser:** Google Chrome
* **Proxy & Tools:** Burp Suite (Proxy, Repeater) with FoxyProxy Extension
* **Target Application:** E-Commerce Web Application (Feedback Form Feature)

---

## Vulnerability Overview & Technical Concept

**Blind OS Command Injection** occurs when an application passes user-supplied input directly to a system shell command, but the application does not return the output of the command in its HTTP response. 

When dealing with a completely "blind" injection point where no direct visual or textual feedback is returned, **Time Delays** are utilized as an out-of-band side-channel. By forcing the underlying operating system to pause execution (such as using the `ping` utility with a count flag), an attacker can measure the time it takes for the HTTP response to return. A delayed response confirms that the injected operating system command executed successfully.

---

## Step-by-Step Exploitation Walkthrough

### Step 1: Intercepting the Request via Burp Suite
1. Configured **FoxyProxy** in Google Chrome to route browser traffic through **Burp Suite**.
2. Opened the lab environment and navigated to the **"Submit Feedback"** link on the main page.
3. Filled out the required feedback form fields (Name, Email, Subject, Message) while Burp Suite Proxy interception was enabled.
4. Captured the resulting HTTP **POST** request and forwarded it directly to **Burp Repeater** for analysis.
5. Inspected the request body format:

```http
POST /feedback/submit HTTP/1.1
Host: target-app.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=Krmsdfasdfhasjdfsdf&name=Mostafa&email=x&subject=cookie&message=Hello

```

---

### Step 2: Analyzing Potential Injection Vectors

Evaluated individual parameters to determine which field is most likely passed directly to backend mail functions (such as PHP's `mail()` or `sendmail()`):

* **`csrf`**: Security token used to prevent Cross-Site Request Forgery; processing failure would abort execution immediately rather than pass to shell utilities.
* **`name`, `subject`, `message**`: Standard text fields stored directly into database records.
* **`email`**: The primary candidate. Backend feedback scripts frequently pass email parameters directly into system mail commands or command-line wrappers like `sendmail`.

---

### Step 3: Injecting the Time-Delay Ping Payload

Modified the `email` parameter value inside Burp Repeater to append a shell separator and a loopback ping command designed to force a 10-second processing delay:

* **Injected Parameter:** `email=x||ping+-c+10+127.0.0.1||`
* **Full POST Body:**

```http
POST /feedback/submit HTTP/1.1
Host: target-app.web-security-academy.net
Content-Type: application/x-www-form-urlencoded

csrf=Krmsdfasdfhasjdfsdf&name=Mostafa&email=x||ping+-c+10+127.0.0.1||&subject=cookie&message=Hello

```

#### Detailed Payload Mechanics:

* `x`: An invalid initial email string designed to ensure the primary command or mail process fails or falls through.
* `||`: The OS shell logical OR operator. If the first command execution encounters an error or fails, the shell proceeds to execute the subsequent command.
* `+`: URL-encoded representation of space characters, required so the web server correctly parses command arguments.
* `ping -c 10 127.0.0.1`: Instructs the operating system to send 10 ICMP echo requests to the loopback interface (`127.0.0.1`). Since each ping request inherently introduces a 1-second interval, sending 10 packets forces a deliberate **10-second delay** on the server side.

---

## Proof of Concept (PoC) & Exploitation Result

* **Server Response Time:** Executing the request caused the HTTP response to hang for precisely **10 seconds** before returning the final response code.
* **Impact:** Confirmed unmitigated command execution vulnerability stemming from insecure handling of input within backend mail dispatch functions.
* **Lab Resolution:** The lab successfully registered as completed upon triggering the verified 10-second system delay.

---

## Remediation & Mitigation

1. **Avoid Shell Execution for Mail Operations:**
Never pass user-supplied input directly to operating system mail binaries or shell interpreters. Use built-in programming language mail libraries (such as SMTP wrappers) that handle parameters securely without invoking a shell.
2. **Strict Input Validation and Sanitization:**
Enforce strict validation regex on email input fields to reject any characters outside standard email formats (blocking shell metacharacters like `|`, `;`, `&`, and spaces).

```

```
