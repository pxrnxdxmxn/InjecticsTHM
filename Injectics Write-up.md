**Exectutive summary:**
	I've found a SQLi and chained into SSTI vulnerability that leads to unfiltered RCE. Potential risk: High. 

**Scope:** 
	Targets: injectics.thm, 10.10.170.31
	Date: 21.11.2025
	Allowed: everything

**Impact:**
	SQLi vulnerability that was found through bruteforcing payloads and lead into RCE with SSTI

**Steps to reproduce:**
		1.  Use nmap to find open ports: nmap -sC -sV -v injectics.thm
		
		2.  Since i found open 80 port - visit site on that port
		
		3.  The page welcomed me. I checked page source. Here's `mail.log` that show us important information about database logics.
		
		4. As usual fuzz directories with `dirsearch -u "http://injectics.thm"`
		
		5.  I found avialable `phpmyadmin` directory, but it gives us nothing, so don't hold too much time here
		
		6.  In `mail.log` we found valuable information such as credentials and DB logic; It says that if table `users` will be deleted or corrupted it insert default credentials into new DB: superadmin@injectics.thm  | superSecurePasswd101; dev@injectics.thm | devPasswd123
		
		7.  Use BurpSuite to catch login request and make bruteforce sql injections  through Intruder. When i sent it to Intruder i decide to try default payloads from https://github.com/payloadbox/sql-injection-payload-list/blob/master/Intruder/exploit/Auth_Bypass.txt
		
		8. Payload ' or 'x' = 'x works. It sent me successful login
		
		9.  I've logged in with dev@injectics.thm and found payload
		
		10. I added a simple SQLi payload into "Leaderboard Editor": 1’ SELECT 1; 
			1. Since it doesn't work i tried: 1; SELECT 1; 
			2. It works. So i gonna try: 1; DROP table users;
			3. Database was dropped. I have to wait like a minute untill DB will create new one with default credentials
			4. When DB restarted i logged in with superadmin@injectics.thm as admin and found flag
			
		11. I tried SSTI in "Update Profile" and it works
			1. I tried simple payload to detect SSTI: {{7 * 7}}
			2. I attempted to enumerate accessible functions, but Twig had several restrictions enabled (e.g., `source`, `range`, `dump` were blocked).
			3. After testing multiple payloads, I discovered that Twig’s `sort` filter could be abused to trigger dangerous PHP functions.  
			4. Working payload:  `{{ ['id',''] | sort('passthru') }}`  This executed system-level commands through `passthru`, giving a primitive form of RCE.
			5. Using this technique, I executed commands to inspect the `/var/www/html` directory and attempted to access the `/flags` folder which was found recently with `dirsearch` and returned `403 Forbidden` when accessed normally.
			6. By running commands via the SSTI payload, I bypassed the access restriction and retrieved the flag file content successfully.
**Evidence:**
![[Pasted image 20251121013602.png]]
{{ ['cat /var/www/html/flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt','']|sort('passthru') }}
![[Pasted image 20251121013658.png]]


**Risk level and Priority**

**Risk Level: HIGH**
**Justification:**  
The identified SSTI vulnerability allowed execution of arbitrary system commands through the Twig template engine. This gives an attacker full control over the backend server, including the ability to read protected files, bypass access controls (403), extract sensitive information, and potentially escalate to full remote code execution (RCE).  
This vulnerability is critical as it compromises the entire application and underlying host.

### **Immediate Actions (Emergency):**

- Disable the vulnerable functionality (e.g., the “Update Profile” feature) until a patch is applied.
    
- Remove dangerous Twig filters/functions such as `sort`, callable filters, and any user‑controlled dynamic evaluation.
    
- Ensure Twig debug mode is disabled.
    

### **Short‑Term Fixes:**

- Strictly sanitize and validate user input; allow only alphanumeric characters and block `{}`, `[]`, `|`, quotes, and other template‑related symbols.
    
- Apply output escaping using `{{ variable | escape }}` everywhere user input is reflected.
    
- Enable Twig sandbox mode and explicitly whitelist allowed functions and filters.
    
- Ensure restricted directories (e.g., `/flags`) cannot be accessed by the web user directly.
    

### **Long‑Term Improvements:**

- Conduct full code review focusing on template rendering logic.
    
- Add static code analysis and security linting as part of CI/CD.
    
- Upgrade Twig and ensure proper sandbox configuration.
    
- Perform regular security testing, including injection‑focused pentests.
    
- Deploy WAF (e.g., ModSecurity) with rules mitigating template injection attempts.
