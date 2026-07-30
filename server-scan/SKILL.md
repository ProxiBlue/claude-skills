---
name: server-scan
description: Comprehensive Magento 2 server security investigation and compromise detection. Use when investigating a potentially compromised server, scanning for malware, detecting webshells, or performing post-incident forensics.
disable-model-invocation: true
---

# Server Security Check — Magento 2 Compromise Investigation

You are performing a comprehensive security investigation on a Magento 2 server. Execute each phase systematically. Report findings as you go — do not wait until the end.

**IMPORTANT**: This skill requires SSH access to the target server. Ask the user for:
1. SSH connection details (user@host -p port)
2. Root access method (su password, sudo, or root SSH)
3. Webroot path (default: `/var/www/webroot/ROOT` or `/var/www/html`)

Set these as variables for the investigation:
```
SSH_CMD="ssh -o ConnectTimeout=10 -o StrictHostKeyChecking=no USER@HOST -p PORT"
WEBROOT="/var/www/webroot/ROOT"
```

---

## PHASE 1: RAPID TRIAGE (run first, report immediately)

### 1.1 Server Identity & Age
```bash
# When was this server created?
stat /etc/machine-id
ls -lt --time=ctime /etc/ssh/ssh_host_*_key | head -1
cat /etc/hostname
uname -a
```

### 1.2 Active Threats Check
```bash
# Processes masquerading as kernel threads (run by non-root = MALICIOUS)
ps aux | grep -E '\[(kswapd|rcu_sched|mm_percpu|raid5|kstrp|watchdog|ksmd|xenbus|kthreadd|slub_flush)\]' | grep -v root

# GSocket reverse shell
ps aux | grep -E 'defunct|gs-netcat|gsocket'

# Deleted executables still running (common for in-memory malware)
ls -la /proc/*/exe 2>/dev/null | grep deleted

# Processes running from suspicious directories
ls -la /proc/*/cwd 2>/dev/null | grep -E 'pub/media|/tmp|/dev/shm'

# Suspicious listening ports
ss -tlnp | grep -v -E ':80|:443|:22|:3306|:6379|:9200|:9000'

# Check for rootkit via LD_PRELOAD
cat /etc/ld.so.preload 2>/dev/null

# Connections to known C2 infrastructure
netstat -an 2>/dev/null | grep -E '45\.77\.95\.4|109\.205\.213|193\.32\.162|gsocket\.ninja'
```

### 1.3 Crontab Analysis
```bash
# All user crontabs — look for base64, curl, wget, gsocket
crontab -l 2>/dev/null
for user in $(cut -f1 -d: /etc/passwd); do
  crons=$(crontab -l -u $user 2>/dev/null)
  [ -n "$crons" ] && echo "=== $user ===" && echo "$crons"
done

# Look for encoded/obfuscated cron entries
grep -r 'base64\|eval\|gsocket\|defunct\|GS_ARGS\|curl.*-o\|wget.*-O' /var/spool/cron/ /etc/cron* 2>/dev/null

# Check profile files for persistence
grep -l 'gsocket\|defunct\|GS_ARGS\|base64' ~/.profile ~/.bashrc ~/.bash_profile /etc/profile.d/* 2>/dev/null
```

**If Phase 1 finds active threats: STOP and report to user before continuing.**

---

## PHASE 2: FILESYSTEM MALWARE SCAN

### 2.1 PHP Files in Media Directories (should NOT exist except errors/ and tcpdf/)
```bash
# Find ALL PHP-executable files in pub/media
find $WEBROOT/pub/media/ -type f \( \
  -name "*.php" -o -name "*.php3" -o -name "*.php4" -o -name "*.php5" \
  -o -name "*.php7" -o -name "*.php8" -o -name "*.pht" -o -name "*.phtml" \
  -o -name "*.phar" -o -name "*.inc" -o -name "*.module" -o -name "*.shtml" \
  -o -name "*.pgif" \
\) -not -path "*/errors/*" -not -path "*/tcpdf/*" -ls 2>/dev/null

# Find files with double extensions (image.jpg.php)
find $WEBROOT/pub/media/ -type f -regextype posix-extended \
  -regex ".*\.(jpg|jpeg|png|gif|svg|webp|ico|css|js)\.(php|phtml|phar|inc|sh|pl|py|cgi).*" -ls 2>/dev/null
```

### 2.2 Upload Directory Scan (primary attack vectors)
```bash
# PolyShell target: custom_options/quote
find $WEBROOT/pub/media/custom_options/ -type f ! -name '.htaccess' -ls 2>/dev/null

# SessionReaper target: customer_address
find $WEBROOT/pub/media/customer_address/ -type f -ls 2>/dev/null | head -50
find $WEBROOT/pub/media/customer_address/ -type f 2>/dev/null | wc -l

# Other sensitive upload dirs
find $WEBROOT/pub/media/customer/ -type f ! -name '.htaccess' -ls 2>/dev/null
find $WEBROOT/pub/media/import/ -type f ! -name '.htaccess' -ls 2>/dev/null
find $WEBROOT/pub/media/downloadable/ -type f ! -name '.htaccess' -ls 2>/dev/null
find $WEBROOT/pub/media/tmp/ -type f -name "*.php" -ls 2>/dev/null
```

### 2.3 GIF89a / PNG Polyglot Detection (images containing PHP)
```bash
# Files claiming to be images but containing PHP code
find $WEBROOT/pub/media/ -type f \( -name "*.jpg" -o -name "*.jpeg" -o -name "*.png" \
  -o -name "*.gif" -o -name "*.webp" -o -name "*.ico" \) \
  -exec sh -c 'head -c 500 "{}" | grep -Pql "<\?php|eval\(|base64_decode|shell_" && echo "SUSPECT: {}"' \; 2>/dev/null

# Use `file` command to detect mismatches (says "PHP" or "text" for images = bad)
find $WEBROOT/pub/media/ -type f \( -name "*.jpg" -o -name "*.gif" -o -name "*.png" \) \
  -not -path "*/cache/*" -not -path "*/.thumbs/*" \
  -exec sh -c 'type=$(file -b "{}"); echo "$type" | grep -qi "php\|text\|ascii" && echo "MISMATCH: {} = $type"' \; 2>/dev/null
```

### 2.4 PHP Code Hidden in Non-PHP Files
```bash
# Search for PHP tags inside image, JS, CSS, and text files
grep -rl '<?php\|eval(\|base64_decode\|shell_exec\|system(\|passthru\|exec(' \
  $WEBROOT/pub/media/ \
  --include="*.jpg" --include="*.jpeg" --include="*.png" --include="*.gif" \
  --include="*.svg" --include="*.webp" --include="*.ico" \
  --include="*.css" --include="*.txt" --include="*.csv" --include="*.htm" \
  --include="*.html" 2>/dev/null
```

### 2.5 Known Malware File Signatures
```bash
# accesson.php backdoor (PolyShell)
find $WEBROOT/ -name 'accesson.php' -type f -ls 2>/dev/null

# Known backdoor filenames
find $WEBROOT/ -type f \( \
  -name "cache.php" -not -path "*/vendor/*" \
  -o -name "json-shell.php" \
  -o -name "bypass.php" -o -name "bypass.phtml" \
  -o -name "c.php" -o -name "r.php" -o -name "rce.php" \
  -o -name "rootz.php" -o -name "sysapi.php" \
  -o -name "gsfa1faewf.txt" \
\) -ls 2>/dev/null

# GSocket artifacts
find / -name 'defunct.dat' -o -name 'defunct' -o -name 'qfile' 2>/dev/null | grep -v /proc/

# Session deserialization payloads in non-session directories
find $WEBROOT/pub/media/ -name "sess_*" -ls 2>/dev/null

# Beacon value 8194460 (accesson.php fingerprint)
grep -rl '8194460' $WEBROOT/pub/ --include="*.php" 2>/dev/null

# Known malicious POST parameter handlers
grep -rl 'VENDOR_NEW_PATH_MAGE\|7faa27b473' $WEBROOT/ --include="*.php" -not -path "*/vendor/*" 2>/dev/null
```

### 2.6 .htaccess Tampering
```bash
# Count all .htaccess in media (legitimate Magento has ~7, attackers plant thousands)
find $WEBROOT/pub/media/ -name ".htaccess" 2>/dev/null | wc -l

# Check for attacker-planted .htaccess (127 bytes = common attacker pattern)
find $WEBROOT/pub/media/ -name ".htaccess" -size 127c 2>/dev/null | wc -l

# Compare root media .htaccess with vendor original
diff <(cat $WEBROOT/pub/media/.htaccess 2>/dev/null) \
     <(cat $WEBROOT/vendor/mage-os/magento2-base/pub/media/.htaccess 2>/dev/null || \
       cat $WEBROOT/vendor/magento/magento2-base/pub/media/.htaccess 2>/dev/null) 2>/dev/null

# Legitimate media .htaccess should contain "php_flag engine 0"
grep -L "php_flag engine 0" $WEBROOT/pub/media/.htaccess 2>/dev/null && echo "WARNING: root media .htaccess missing php_flag engine 0"
```

### 2.7 Recently Modified Files (last 7 days)
```bash
# PHP/PHTML files modified recently outside expected paths
find $WEBROOT/ -type f \( -name "*.php" -o -name "*.phtml" \) -mtime -7 \
  -not -path "*/vendor/*" -not -path "*/var/*" -not -path "*/generated/*" \
  -not -path "*/pub/static/*" -not -path "*/pub/media/static/*" \
  -not -path "*/.git/*" -not -path "*/node_modules/*" \
  -not -path "*/scraper/*" -not -path "*/test/*" \
  -ls 2>/dev/null

# New files in media in last 4 days (excluding cache/catalog product images)
find $WEBROOT/pub/media/ -type f -mtime -4 \
  -not -path "*/cache/*" -not -path "*/catalog/product/cache/*" \
  -not -path "*/tmp/*" -not -path "*/.thumbs/*" \
  -not -path "*/pub/media/static/*" \
  -not -name "*.jpg" -not -name "*.jpeg" -not -name "*.png" \
  -not -name "*.gif" -not -name "*.webp" -not -name "*.svg" \
  2>/dev/null | head -50
```

---

## PHASE 3: CRITICAL FILE INTEGRITY

### 3.1 Entry Points
```bash
# pub/index.php — verify clean (no eval, no require to suspicious files, no C2 URLs)
head -40 $WEBROOT/pub/index.php
md5sum $WEBROOT/pub/index.php

# List all PHP in pub/ root (should only be: index.php, get.php, static.php, health_check.php, cron.php)
find $WEBROOT/pub/ -maxdepth 1 -name "*.php" -ls 2>/dev/null

# Check for backdoor PHP files added to pub/ root
find $WEBROOT/pub/ -maxdepth 1 -name "*.php" ! \( \
  -name "index.php" -o -name "get.php" -o -name "static.php" \
  -o -name "health_check.php" -o -name "cron.php" \) -ls 2>/dev/null
```

### 3.2 Generated Code (Interceptor Backdoors)
```bash
# Check FrontController Interceptor for tampering (most common injection point)
grep -c 'eval\|base64_decode\|shell_exec\|system(' \
  $WEBROOT/generated/code/Magento/Framework/App/FrontController/Interceptor.php 2>/dev/null

# Check all generated interceptors for suspicious code
grep -rl 'eval(\|base64_decode\|shell_exec\|system(\|passthru\|cryption_block' \
  $WEBROOT/generated/ --include="*.php" 2>/dev/null

# Check app/autoload.php
grep -c 'VENDOR_NEW_PATH_MAGE\|eval\|base64' $WEBROOT/app/autoload.php 2>/dev/null
```

### 3.3 Vendor File Integrity
```bash
# Check if critical vendor files were modified (compare mtime with install date)
ls -la $WEBROOT/vendor/magento/module-customer/Model/Session.php 2>/dev/null
ls -la $WEBROOT/vendor/magento/module-backend/Model/Auth.php 2>/dev/null

# Quick check for eval/base64 in vendor (should be very rare in legitimate code)
grep -rl 'eval(base64_decode\|eval(gzinflate\|assert(\$_' $WEBROOT/vendor/ --include="*.php" 2>/dev/null | head -10
```

### 3.4 JavaScript Skimmer Check
```bash
# Check for modified jQuery
ls -la $WEBROOT/lib/web/jquery.js 2>/dev/null
grep -c 'eval\|createElement.*script\|document\.write' $WEBROOT/lib/web/jquery.js 2>/dev/null

# Check for suspicious JS in pub/static
find $WEBROOT/pub/static/ -name "*.js" -newer $WEBROOT/pub/index.php -ls 2>/dev/null | head -20
```

---

## PHASE 4: CONFIGURATION & INFRASTRUCTURE

### 4.1 resource_config.json (controls what get.php serves)
```bash
# Check allowed_resources — customer_address should NOT be listed
cat $WEBROOT/var/resource_config.json 2>/dev/null | python3 -m json.tool 2>/dev/null || cat $WEBROOT/var/resource_config.json
```

### 4.2 Nginx Configuration
```bash
# Check for deny rules on sensitive paths
grep -A2 'customer_address\|customer/\|downloadable\|import\|custom_options\|guest-carts\|address_file' /etc/nginx/conf.d/SITES_ENABLED/*.conf 2>/dev/null || \
grep -A2 'customer_address\|customer/\|downloadable\|import\|custom_options\|guest-carts\|address_file' /etc/nginx/sites-enabled/*.conf 2>/dev/null

# Check PHP execution is limited to entry points
grep -E 'location.*\.php|fastcgi_pass' /etc/nginx/conf.d/SITES_ENABLED/*.conf 2>/dev/null | head -20

# Check catch-all PHP deny exists
grep -A1 '\.php\$.*\.phtml' /etc/nginx/conf.d/SITES_ENABLED/*.conf 2>/dev/null
```

### 4.3 env.php Encryption Key (CosmicSting indicator)
```bash
# If encryption key was stolen via XXE, attacker can forge admin JWT tokens
# Check when env.php was last modified
ls -la $WEBROOT/app/etc/env.php

# Check access logs for CosmicSting exploitation
grep -c 'estimate-shipping-methods' /var/log/nginx/access.log 2>/dev/null
```

---

## PHASE 5: LOG ANALYSIS

### 5.1 Attack Endpoint Access
```bash
# SessionReaper: customer address file upload
grep -c "address_file/upload" /var/log/nginx/access.log 2>/dev/null
grep "address_file/upload" /var/log/nginx/access.log 2>/dev/null | awk '{print $1}' | sort -u

# PolyShell: guest-carts item upload
grep -c "guest-carts.*items" /var/log/nginx/access.log 2>/dev/null

# CosmicSting: shipping estimation XXE
grep -c "estimate-shipping-methods" /var/log/nginx/access.log 2>/dev/null

# Direct access to media PHP files (execution attempts)
grep -E "media/.*\.(ph(p|tml|ar)|inc|pgif|module)" /var/log/nginx/access.log 2>/dev/null | tail -20

# guest-carts order endpoint abuse
grep "guest-carts.*/order" /var/log/nginx/access.log 2>/dev/null | awk '{print $1}' | sort | uniq -c | sort -rn | head -10

# Error log: access forbidden (shows blocked attempts)
grep "access forbidden by rule" /var/log/nginx/error.log 2>/dev/null | tail -20
```

### 5.2 Magento Application Logs
```bash
# Deserialization attempts
grep -i 'unserialize\|deserialization\|FileCookieJar\|SyslogUdp\|BufferHandler' \
  $WEBROOT/var/log/exception.log 2>/dev/null | tail -10

# Suspicious admin activity
grep -i 'admin.*login\|admin.*user.*create' $WEBROOT/var/log/system.log 2>/dev/null | tail -20
```

### 5.3 Unique Attacker IPs
```bash
# All IPs hitting upload endpoints
cat /var/log/nginx/access.log 2>/dev/null | \
  grep -E "address_file/upload|custom_options.*quote|guest-carts.*/order" | \
  awk '{print $1}' | sort -u
```

---

## PHASE 6: DATABASE CHECKS

Use the database MCP tool or direct MySQL access:

```sql
-- Unauthorized admin accounts
SELECT user_id, username, email, created, modified, is_active 
FROM admin_user ORDER BY created DESC;

-- JavaScript skimmers in CMS blocks
SELECT block_id, title, identifier 
FROM cms_block 
WHERE content LIKE '%<script%' OR content LIKE '%eval(%' 
   OR content LIKE '%base64%' OR content LIKE '%document.createElement%';

-- Skimmers in config
SELECT path, value 
FROM core_config_data 
WHERE (value LIKE '%<script%' OR value LIKE '%eval(%' OR value LIKE '%base64%')
  AND path LIKE 'design/%';

-- Layout XML injection (CVE-2024-20720)
SELECT * FROM layout_update 
WHERE xml LIKE '%assert%' OR xml LIKE '%system%' OR xml LIKE '%eval%';

-- Template injection in order addresses
SELECT entity_id, firstname, lastname, street 
FROM sales_order_address 
WHERE firstname LIKE '%{{%' OR lastname LIKE '%{{%' 
   OR street LIKE '%{{%' OR vat_id LIKE '%{{%'
LIMIT 10;
```

---

## PHASE 7: KNOWN CVE-SPECIFIC CHECKS

### 7.1 CosmicSting (CVE-2024-34102)
- Check for XXE via estimate-shipping-methods endpoint in logs
- Verify encryption key hasn't been compromised (check for unauthorized API calls with valid JWT)
- Look for CMS block modifications with injected JavaScript

### 7.2 SessionReaper (CVE-2025-54236)
- Files in `pub/media/customer_address/` with session-like names (`sess_*`)
- Deserialization chains targeting Monolog or GuzzleHttp
- Backdoor files at `pub/1cbb0edc6cb0.php`, `pub/8303e8b447b4.php`, `pub/static.php`, `pub/bootstrap.php`, `pub/sysapi.php`

### 7.3 PolyShell (APSB25-94)
- Files in `pub/media/custom_options/quote/`
- GIF89a polyglot files (`GIF89a<?php...`)
- `accesson.php` distributed across `assets/images/` in many directories
- Beacon value `8194460` in PHP files

### 7.4 CVE-2024-20720 (Layout XML Injection)
- Malicious entries in `layout_update` database table
- Modified `generated/code/Magento/Cms/Controller/Index/Index/Interceptor.php`

### 7.5 GSocket Reverse Shell
- Binary at `~/.config/htop/defunct`
- Key file `defunct.dat` in various locations
- Cron entry with "SEED PRNG" comment or base64-encoded payload
- Process name `[defunct]` running as web user

---

## PHASE 8: REPORT

Generate a structured report with:

1. **Severity**: CRITICAL / HIGH / MEDIUM / CLEAN
2. **Active Threats**: Any currently running malicious processes
3. **Malware Found**: List all suspicious files with paths, sizes, and content summaries
4. **Attack Timeline**: When did attacks start/stop based on file dates and logs
5. **Attacker IPs**: All unique IPs involved
6. **Attack Vectors Used**: Which CVEs/techniques were attempted
7. **What Was Executed**: Evidence of successful vs failed exploitation
8. **Remediation Actions**: What needs to be done immediately
9. **Recommended Hardening**: Nginx deny rules, config changes, patches needed

### Known Attacker IP Ranges (cross-reference with logs)
```
# CosmicSting: 157.230.230.193, 185.175.225.116, 142.252.84.169, 185.208.158.16
# SessionReaper: 23.146.184.93, 23.249.27.221, 34.227.25.4, 44.212.43.34, 45.32.66.51
# PolyShell: 104.129.16.8, 104.168.14.206, 107.172.97.122, 155.94.198.5
# GSocket C2: 45.77.95.4, 109.205.213.203, 193.32.162.10
# Template attacks: 45.128.199.3, 45.134.20.11, 86.104.15.60
```

### Known C2 Domains (check DNS/network logs)
```
# Skimmer loaders: jslibrary.net, cdnstatics.net, statspots.com, cssmagic.shop
# Backdoor C2: sagecrafft.com, worcksbot.com, tecnokauf.ru, dev-clientservice.com
# GSocket: g.gsocket.ninja, d.gsocket.ninja
# WebSocket C2: wss://accept.bar/common, wss://cdn.iconstaff.top/common
```

### Recommended Nginx Deny Rules
```nginx
# Inside location /media/ { } block:
location /media/customer_address/ { deny all; }

# Standalone blocks:
location ~ /rest/([^/]*/)?V1/guest-carts/.*/order { deny all; }
location ^~ /customer/address_file/upload { deny all; }
location /media/customer/ { deny all; }
location /media/downloadable/ { deny all; }
location /media/import/ { deny all; }
location /media/custom_options/ { deny all; }
location ~ /rest/([^/]*/)?V1/customers/me { deny all; }

# Catch-all PHP deny (must be last):
location ~* (\.php$|\.phtml$|\.htaccess$|\.htpasswd$|\.git|\.user\.ini) { deny all; }
```

**IMPORTANT**: The `/media/customer_address/` deny MUST be inside the `location /media/ {}` block due to nginx prefix match ordering. A standalone block outside will be ignored.
