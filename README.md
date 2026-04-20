# 🛡️ SecureAudit — Java Secure Code Review
### CodeAlpha Cybersecurity Internship — Task 3

![Java](https://img.shields.io/badge/Java-1.8-orange?style=flat-square&logo=java)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-2.x-green?style=flat-square&logo=springboot)
![Docker](https://img.shields.io/badge/Docker-Compose-blue?style=flat-square&logo=docker)
![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?style=flat-square&logo=mysql)
![License](https://img.shields.io/badge/License-Educational-red?style=flat-square)

---

## 📌 About This Project

This project is a **Secure Code Review** audit performed on a deliberately vulnerable Java web application as part of the **CodeAlpha Cybersecurity Internship (Task 3)**.

The base application [`java-sec-code`](https://github.com/JoyChou93/java-sec-code) by JoyChou contains intentional security vulnerabilities for educational purposes. This repository documents the audit process, identified vulnerabilities, proof-of-concept demonstrations, and remediation recommendations.

A custom **SecureAudit dashboard** UI was designed and injected into the running Docker container to present the vulnerability modules professionally.

---

## ⚠️ Disclaimer

> This project is for **educational purposes only**.  
> All testing was performed on a **local isolated Docker environment**.  
> Do NOT use any of these techniques on systems you do not own or have explicit permission to test.

---

## 🎯 Audit Objectives

- Identify critical security vulnerabilities in a Java Spring Boot application
- Demonstrate exploitability with proof-of-concept attacks
- Provide remediation recommendations for each vulnerability
- Document findings in a structured security report

---

## 🏗️ Project Structure

```
java-sec-code-master/
├── src/
│   └── main/
│       ├── java/org/joychou/
│       │   ├── controller/          # Vulnerable endpoint controllers
│       │   │   ├── Rce.java         # Remote Code Execution
│       │   │   ├── SQLI.java        # SQL Injection
│       │   │   ├── PathTraversal.java  # Path Traversal
│       │   │   ├── SSRF.java        # Server-Side Request Forgery
│       │   │   ├── XSS.java         # Cross-Site Scripting
│       │   │   ├── XXE.java         # XML External Entity
│       │   │   ├── FileUpload.java  # Unrestricted File Upload
│       │   │   ├── CORS.java        # CORS Misconfiguration
│       │   │   ├── Jwt.java         # JWT Vulnerabilities
│       │   │   ├── CommandInject.java  # Command Injection
│       │   │   ├── SSRF.java        # SSRF
│       │   │   └── ...              # More vulnerabilities
│       │   ├── security/
│       │   │   └── WebSecurityConfig.java  # Spring Security config
│       │   └── config/
│       │       └── SwaggerConfig.java
│       └── resources/
│           ├── templates/           # Thymeleaf HTML templates
│           │   ├── index.html       # Custom SecureAudit dashboard
│           │   └── login.html       # Custom SecureAudit login
│           └── application.properties
├── docker-compose.yml
└── pom.xml
```

---

## 🔴 Vulnerabilities Identified

### 1. Remote Code Execution (RCE) — CRITICAL

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/Rce.java` |
| **Endpoint** | `GET /rce/runtime/exec?cmd={command}` |
| **Severity** | 🔴 Critical |
| **CVSS Score** | 10.0 |

**Description:** User-supplied input is passed directly to `Runtime.exec()` without any validation, allowing arbitrary OS command execution.

**Proof of Concept:**
```
http://localhost:8080/rce/runtime/exec?cmd=whoami
→ Response: root

http://localhost:8080/rce/runtime/exec?cmd=ls
→ Lists all server files
```

**Vulnerable Code:**
```java
Runtime rt = Runtime.getRuntime();
String[] commands = {"/bin/sh", "-c", cmd};  // cmd is user input!
Process proc = rt.exec(commands);
```

**Remediation:**
```java
// Use a whitelist of allowed commands
List<String> allowedCommands = Arrays.asList("whoami", "date");
if (!allowedCommands.contains(cmd)) {
    return "Command not allowed";
}
```

---

### 2. SQL Injection — CRITICAL

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/SQLI.java` |
| **Endpoint** | `GET /sqli/jdbc/vuln?username={input}` |
| **Severity** | 🔴 Critical |
| **CVSS Score** | 9.8 |

**Description:** User input is concatenated directly into SQL queries, allowing attackers to manipulate database logic.

**Proof of Concept:**
```
Normal:   /sqli/jdbc/vuln?username=joychou
→ joychou: 123

Injected: /sqli/jdbc/vuln?username=admin' or '1'='1
→ joychou: 123  wilson: 456  lightless: 789
```
The injection dumped ALL users from the database.

**Vulnerable Code:**
```java
String sql = "select * from users where username = '" + username + "'";
// username is user input — never do this!
```

**Remediation:**
```java
// Use Prepared Statements
String sql = "select * from users where username = ?";
PreparedStatement ps = connection.prepareStatement(sql);
ps.setString(1, username);
```

---

### 3. Path Traversal — CRITICAL

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/PathTraversal.java` |
| **Endpoint** | `GET /path_traversal/vul?filepath={path}` |
| **Severity** | 🔴 Critical |
| **CVSS Score** | 7.5 |

**Description:** The application does not sanitize file path input, allowing attackers to escape the intended directory and read arbitrary system files.

**Proof of Concept:**
```
http://localhost:8080/path_traversal/vul?filepath=../../../etc/passwd
→ Returns contents of /etc/passwd (Linux system file)
```

**Vulnerable Code:**
```java
File file = new File(filepath);  // filepath is user input!
// No path validation performed
```

**Remediation:**
```java
// Normalize and validate the path
Path normalizedPath = Paths.get(baseDir).resolve(filepath).normalize();
if (!normalizedPath.startsWith(baseDir)) {
    throw new SecurityException("Path traversal detected!");
}
```

---

### 4. SSRF — Server-Side Request Forgery — HIGH

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/SSRF.java` |
| **Endpoint** | `GET /ssrf/urlConnection/vuln?url={url}` |
| **Severity** | 🟠 High |
| **CVSS Score** | 8.6 |

**Description:** The server makes HTTP requests to URLs controlled by the attacker, allowing access to internal network resources.

**Proof of Concept:**
```
http://localhost:8080/ssrf/urlConnection/vuln?url=file:///etc/passwd
→ Reads internal files

http://localhost:8080/ssrf/urlConnection/vuln?url=http://169.254.169.254/
→ Access cloud metadata (AWS/GCP)
```

**Remediation:** Implement a URL whitelist, block internal IP ranges (127.x, 10.x, 192.168.x), and disable file:// protocol.

---

### 5. File Upload — CRITICAL

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/FileUpload.java` |
| **Endpoint** | `POST /file/any` |
| **Severity** | 🔴 Critical |
| **CVSS Score** | 9.0 |

**Description:** No file type validation is performed, allowing upload of malicious files including web shells.

**Remediation:** Validate file extension and MIME type, store files outside the web root, rename uploaded files randomly.

---

### 6. XXE — XML External Entity — HIGH

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/XXE.java` |
| **Endpoint** | `POST /xxe/vuln` |
| **Severity** | 🟠 High |
| **CVSS Score** | 7.5 |

**Description:** XML parser processes external entity declarations, allowing file disclosure and SSRF.

**Remediation:** Disable external entity processing in the XML parser configuration.

---

### 7. CORS Misconfiguration — MEDIUM

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/CORS.java` |
| **Severity** | 🟡 Medium |

**Description:** Overly permissive CORS policy allows any origin to make cross-origin requests.

**Remediation:** Restrict `Access-Control-Allow-Origin` to trusted domains only.

---

### 8. JWT Vulnerabilities — MEDIUM

| Field | Details |
|-------|---------|
| **File** | `src/main/java/org/joychou/controller/Jwt.java` |
| **Severity** | 🟡 Medium |

**Description:** Weak JWT signing and validation allows token forgery.

**Remediation:** Use strong secret keys, validate all claims, and reject `alg: none` tokens.

---

## 🚀 How to Run

### Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- [Git](https://git-scm.com/) installed

### Quick Start

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/CodeAlpha_SecureCodeReview
cd CodeAlpha_SecureCodeReview

# Start the application
docker-compose up -d

# Access the application
# http://localhost:8080
```

### Login Credentials
```
admin / admin123
joychou / joychou123
```

### Stop the Application
```bash
docker-compose down
```

### Docker Environment
- Java 1.8.0_102
- MySQL 8.0.17
- Spring Boot 2.x
- Thymeleaf

---

## 🔬 Testing the Vulnerabilities

### RCE
```bash
curl "http://localhost:8080/rce/runtime/exec?cmd=whoami"
# Expected: root
```

### SQL Injection
```bash
curl "http://localhost:8080/sqli/jdbc/vuln?username=admin'+or+'1'%3D'1"
# Expected: all users dumped
```

### Path Traversal
```bash
curl "http://localhost:8080/path_traversal/vul?filepath=../../../etc/passwd"
# Expected: /etc/passwd contents
```

---

## 🛠️ Tech Stack

| Technology | Version | Purpose |
|-----------|---------|---------|
| Java | 1.8 | Backend language |
| Spring Boot | 2.x | Web framework |
| Thymeleaf | 3.x | Template engine |
| MySQL | 8.0 | Database |
| MyBatis | 3.x | ORM framework |
| Docker | Latest | Containerization |
| Maven | 3.x | Build tool |

---

## 📊 Vulnerability Summary

| # | Vulnerability | Severity | CVSS | Status |
|---|--------------|----------|------|--------|
| 1 | Remote Code Execution | 🔴 Critical | 10.0 | Demonstrated |
| 2 | SQL Injection | 🔴 Critical | 9.8 | Demonstrated |
| 3 | Path Traversal | 🔴 Critical | 7.5 | Demonstrated |
| 4 | File Upload | 🔴 Critical | 9.0 | Identified |
| 5 | SSRF | 🟠 High | 8.6 | Demonstrated |
| 6 | XXE | 🟠 High | 7.5 | Identified |
| 7 | CORS | 🟡 Medium | 5.3 | Identified |
| 8 | JWT | 🟡 Medium | 6.5 | Identified |

---

## 👨‍💻 Author

**[Taha El Cheikh]**
- CodeAlpha Cybersecurity Intern
- GitHub: [@majd-el-masri]((https://github.com/tahacheikh2003))


---

## 🙏 Credits

- Original vulnerable application: [java-sec-code](https://github.com/JoyChou93/java-sec-code) by JoyChou
- Internship: [CodeAlpha](https://www.codealpha.tech)

---

## 📄 License

This project is for **educational purposes only**.  
