# PA1 — Password Attacks Complete Mastery

---

## STEP 1: PASSWORD ATTACKS MINDSET

### 1.1 — Why Password Attacks Matter in OSCP

- **Weak credentials = easiest entry point** into any system
- **Service login bypass** — SSH, FTP, SMB, Web panels
- **Hash cracking** — after hash retrieval, crack offline for privilege escalation
- **Credential reuse** — one password can unlock multiple services
- **OSCP machines commonly have weak/default passwords** — always try these first

---

### 1.2 — Password Attack Types Overview

```
[Password Attack Types]
          |
    ______|________
   |       |       |
Brute   Dictionary  Rainbow
Force    Attack      Table
  |         |          |
Try every  Use a      Pre-computed
combo      wordlist   hash lookup
  |         |          |
Slow but  Fast &     Fastest —
complete  effective  needs table
```

---

### 1.3 — When to Use Which Attack — Decision Tree

```
[Need a Password]
         |
         v
[Do you have a Hash?]
   YES /    \ NO
      /      \
[Offline    [Online Service
 Cracking]   Brute Force]
      |              |
  [Hash type?]   [Service type?]
  MD5/SHA/NTLM   SSH/FTP/Web/SMB
      |              |
  John/Hashcat    Hydra/Medusa
                  /CrackMapExec
```

---

## STEP 2: WORDLISTS — YOUR AMMUNITION

### 2.1 — What are Wordlists?

Wordlists are pre-made text files containing passwords, used for dictionary attacks. **Quality over quantity** — choosing the right wordlist matters more than the biggest one.

---

### 2.2 — Important Wordlists on Kali Linux

| Path | Description |
|------|-------------|
| `/usr/share/wordlists/rockyou.txt` | 14M real-world passwords — most used |
| `/usr/share/seclists/Passwords/` | Large collection of password lists |
| `/usr/share/seclists/Passwords/Common-Credentials/top-passwords-shortlist.txt` | Quick top passwords |
| `/usr/share/seclists/Usernames/` | Username lists |
| `/usr/share/wordlists/dirb/common.txt` | Web directories |

---

### 2.3 — Extract rockyou.txt

```bash
# Decompress the file
gunzip /usr/share/wordlists/rockyou.txt.gz

# Count total passwords
wc -l /usr/share/wordlists/rockyou.txt
# Output: 14,344,391 passwords
```

---

### 2.4 — Generate Custom Wordlist with CeWL

CeWL scrapes a target website and builds a wordlist from words found on the page. Useful for custom context — company names, product names, slogans.

```bash
# Scrape target website, minimum 5-character words, output to custom.txt
cewl http://TARGET_IP -m 5 -w custom.txt

# With depth and authentication
cewl http://TARGET_IP -m 5 -d 3 -w custom.txt
```

**Flags:**
- `-m 5` → minimum word length of 5 characters
- `-d 3` → crawl depth (how many links deep)
- `-w` → output wordlist file

---

### 2.5 — Generate Custom Wordlist with Crunch

Crunch generates wordlists based on character sets and patterns.

```bash
# Generate all combos: 6 to 8 characters, alphanumeric
crunch 6 8 abcdefghijklmnopqrstuvwxyz0123456789 -o wordlist.txt

# Pattern-based: Password0001 to Password9999
crunch 8 8 -t Password@@@@ -o custom.txt

# Uppercase + numbers, exactly 8 chars
crunch 8 8 ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 -o upper_num.txt
```

**Pattern characters for `-t` flag:**
- `@` → lowercase letters
- `,` → uppercase letters
- `%` → numbers
- `^` → special characters

---

### 2.6 — Mutate Wordlists with Hashcat Rules

Rules apply transformations to existing wordlists — adding numbers, capitalizing, leet speak, etc.

```bash
# Apply best64 rules to rockyou and save mutated list
hashcat --stdout -r /usr/share/hashcat/rules/best64.rule \
  /usr/share/wordlists/rockyou.txt > mutated.txt

# Example transformations:
# password → Password, PASSWORD, p@ssw0rd, password123
```

**Common rule files in Kali:**

| Rule File | Purpose |
|-----------|---------|
| `best64.rule` | 64 most effective rules |
| `rockyou-30000.rule` | 30,000 rules from rockyou study |
| `d3ad0ne.rule` | Aggressive transformations |
| `leetspeak.rule` | Leet speak substitutions |

---

### 2.7 — Wordlist Selection Decision Tree

```
[Starting a Password Attack]
            |
            v
    [Context available?]
    YES /        \ NO
       /          \
[Use CeWL on    [Use rockyou.txt
 target site]    as default]
       |               |
[Company names,   [Add rules if
 product words]    not cracking]
       |               |
[Combine with    [hashcat -r best64]
 rockyou.txt]
```

---

## STEP 3: ONLINE BRUTE FORCE — HYDRA (Complete Guide)

### 3.1 — What is Hydra?

Hydra is a fast, parallelized network login cracker. It supports 50+ protocols and is the primary online brute-force tool for OSCP.

---

### 3.2 — Understanding Hydra Syntax

```bash
# Single username, password list
hydra -l USERNAME -P WORDLIST SERVICE://TARGET

# Username list, password list
hydra -L USERLIST -P WORDLIST SERVICE://TARGET

# Single username, single password (testing)
hydra -l USERNAME -p PASSWORD SERVICE://TARGET
```

**All Hydra Flags Explained:**

| Flag | Meaning | Example |
|------|---------|---------|
| `-l` | Single username | `-l admin` |
| `-L` | Username list file | `-L users.txt` |
| `-p` | Single password | `-p password123` |
| `-P` | Password list file | `-P rockyou.txt` |
| `-t` | Threads (default 16) | `-t 4` |
| `-s` | Custom port | `-s 2222` |
| `-v` | Verbose output | `-v` |
| `-V` | Show every attempt | `-V` |
| `-f` | Stop on first found | `-f` |
| `-o` | Save results to file | `-o results.txt` |
| `-e nsr` | Try null, same, reverse | `-e nsr` |
| `-w` | Wait time between attempts | `-w 3` |

---

### 3.3 — SSH Brute Force with Hydra

```bash
# Basic SSH brute force — single user
hydra -l root -P /usr/share/wordlists/rockyou.txt \
  ssh://192.168.1.10 -t 4 -f

# Multiple usernames
hydra -L users.txt -P /usr/share/wordlists/rockyou.txt \
  ssh://192.168.1.10 -t 4 -o ssh_results.txt

# Custom port SSH
hydra -l admin -P rockyou.txt \
  ssh://192.168.1.10 -s 2222 -t 4 -f

# Try null password, same as username, reverse username
hydra -l root -P rockyou.txt \
  ssh://192.168.1.10 -t 4 -e nsr
```

> **Why `-t 4` for SSH?** SSH has rate limiting and connection throttling. Using too many threads (like default 16) causes connection drops and bans. Keep threads low at 4.

---

### 3.4 — FTP Brute Force with Hydra

```bash
# Basic FTP brute force
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  ftp://192.168.1.10 -t 10 -f

# Check for anonymous login first (before brute forcing)
hydra -l anonymous -p anonymous ftp://192.168.1.10

# With username list
hydra -L users.txt -P rockyou.txt \
  ftp://192.168.1.10 -t 10 -o ftp_results.txt
```

---

### 3.5 — HTTP Login Form Brute Force with Hydra

This is the most complex but most useful for OSCP web challenges.

**Step 1 — Identify the login form:**
Open browser → DevTools (F12) → Network tab → Submit a wrong login → Look at the POST request

**Step 2 — Note these 3 things:**
1. Login page URL path (e.g., `/login`, `/admin/login.php`)
2. POST parameter names (e.g., `username=`, `password=`)
3. Failure message shown on wrong login (e.g., "Invalid credentials", "Login failed")

```bash
# HTTP GET form
hydra -l admin -P rockyou.txt \
  192.168.1.10 http-get /login

# HTTP POST form — most common
hydra -l admin -P rockyou.txt \
  192.168.1.10 http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials"

# HTTPS POST form
hydra -l admin -P rockyou.txt \
  192.168.1.10 https-post-form \
  "/login:username=^USER^&password=^PASS^:Wrong password"

# With cookie/session token needed
hydra -l admin -P rockyou.txt \
  192.168.1.10 http-post-form \
  "/login:user=^USER^&pass=^PASS^:Login failed:H=Cookie: PHPSESSID=abc123"
```

**Format breakdown:**
```
"PATH : POST_PARAMETERS : FAILURE_STRING"
  |           |                 |
Login URL   Form fields    Text shown on
            ^USER^ and     failed login
            ^PASS^ are
            replaced by
            Hydra
```

---

### 3.6 — SMB Brute Force with Hydra and CrackMapExec

```bash
# Hydra SMB
hydra -l administrator -P rockyou.txt \
  smb://192.168.1.10

# CrackMapExec — better for SMB (handles errors cleaner)
crackmapexec smb 192.168.1.10 -u users.txt -p passwords.txt

# CME with single password spray
crackmapexec smb 192.168.1.10 -u users.txt -p "Password123"

# CME continue even after finding valid creds
crackmapexec smb 192.168.1.10 -u users.txt -p passwords.txt \
  --continue-on-success
```

**CME Output legend:**
- `[+]` → Valid credentials found
- `[-]` → Failed attempt
- `(Pwn3d!)` → Admin-level access confirmed

---

### 3.7 — RDP Brute Force with Hydra

```bash
# Basic RDP brute force
hydra -l administrator -P rockyou.txt \
  rdp://192.168.1.10 -t 4

# With custom port
hydra -l admin -P rockyou.txt \
  rdp://192.168.1.10 -s 3389 -t 4
```

> **Note:** RDP brute force is slow and noisy. Use sparingly in OSCP. Try default creds manually first.

---

### 3.8 — Reading Hydra Output

```
[22][ssh] host: 192.168.1.10   login: admin   password: password123
  |    |         |                  |                 |
Port  Service  Target IP        Username           Password FOUND
```

Save this immediately:
```bash
# Always use -o flag to save results
hydra -l admin -P rockyou.txt ssh://192.168.1.10 -o found_creds.txt
```

---

### 3.9 — Hydra Complete Decision Tree

```
[Need to brute force a service]
              |
              v
    [What service is it?]
    /    |      |     \
  SSH   FTP   HTTP   SMB/RDP
   |     |      |       |
-t 4   -t 10  POST    Use CME
              form    preferred
                |
        [Find failure string]
                |
          Open browser →
          F12 → Network →
          Submit wrong login →
          Note error message
                |
        [Build hydra command]
        "/path:user=^USER^
         &pass=^PASS^
         :failure_string"
```

---

## STEP 4: MEDUSA — ALTERNATIVE BRUTE FORCER

### 4.1 — Medusa vs Hydra

| Feature | Hydra | Medusa |
|---------|-------|--------|
| Speed | Fast | Slightly faster in some cases |
| HTTP forms | Better | Basic |
| SMB | Good | Good |
| SSH/FTP | Good | Good |
| OSCP preference | Primary | Backup |

Use Medusa when Hydra fails or gives unexpected errors.

---

### 4.2 — Medusa Syntax and Commands

```bash
# Basic syntax
medusa -h TARGET -u USERNAME -P WORDLIST -M MODULE

# SSH brute force
medusa -h 192.168.1.10 -u admin \
  -P /usr/share/wordlists/rockyou.txt -M ssh

# FTP with threads
medusa -h 192.168.1.10 -U users.txt \
  -P passwords.txt -M ftp -t 10

# HTTP basic auth
medusa -h 192.168.1.10 -u admin \
  -P rockyou.txt -M http -m DIR:/protected

# SMB
medusa -h 192.168.1.10 -U users.txt \
  -P rockyou.txt -M smbnt
```

**Medusa Flags:**

| Flag | Meaning |
|------|---------|
| `-h` | Target host IP |
| `-H` | File of target hosts |
| `-u` | Single username |
| `-U` | Username file |
| `-p` | Single password |
| `-P` | Password file |
| `-M` | Module (ssh, ftp, http, smbnt) |
| `-t` | Threads |
| `-f` | Stop after first valid pair |
| `-O` | Output file |
| `-v 6` | Verbose (level 6 = max) |

---

## STEP 5: HASH CRACKING — OFFLINE ATTACKS

### 5.1 — How Do You Get Hashes?

| Source | Method | Hash Type |
|--------|--------|-----------|
| `/etc/shadow` | Linux file read | SHA-512 `$6$` |
| Metasploit `hashdump` | Post-exploitation | NTLM |
| Mimikatz | Windows credential dump | NTLM, Kerberos |
| Database files | Web app config | MD5, bcrypt |
| Responder capture | Network poisoning | NTLMv2 |

---

### 5.2 — Identify Hash Type

```bash
# Interactive identifier
hash-identifier
# Paste hash when prompted

# Direct identification
hashid '$1$abc$hashhashhashhashhashhash'
hashid '5f4dcc3b5aa765d61d8327deb882cf99'

# Manual identification by prefix:
```

**Hash Prefix Cheat Sheet:**

| Prefix | Type | Hashcat Mode |
|--------|------|-------------|
| `$1$` | MD5crypt (Linux old) | 500 |
| `$5$` | SHA-256crypt (Linux) | 7400 |
| `$6$` | SHA-512crypt (Linux) | 1800 |
| `$y$` | yescrypt (modern Linux) | 7400 |
| `$2b$` | bcrypt | 3200 |
| 32-char hex, no prefix | NTLM (Windows) | 1000 |
| 32-char hex, no prefix | MD5 | 0 |

---

### 5.3 — Understanding /etc/shadow Format

```
root:$6$salt$hashhashhashhash...:18000:0:99999:7:::
 |    |  |         |               |
 |    |  |     Hash value      Last change (days)
 |    | Salt  (SHA-512)
 |   $6$ = SHA-512
Username
```

To extract just the hash for cracking:
```bash
# Copy from $6$ to end of hash (no spaces, no colons)
$6$rounds=5000$salt$hashhashhashhash...
```

---

### 5.4 — John the Ripper — Complete Guide

John the Ripper is the go-to tool for Linux hash cracking and versatile offline attacks.

**Basic cracking:**
```bash
# Auto-detect format and crack
john hash.txt --wordlist=/usr/share/wordlists/rockyou.txt

# Specify format explicitly (more reliable)
john hash.txt --format=sha512crypt \
  --wordlist=/usr/share/wordlists/rockyou.txt

# MD5crypt Linux hashes
john hash.txt --format=md5crypt \
  --wordlist=/usr/share/wordlists/rockyou.txt

# NTLM Windows hashes
john hash.txt --format=NT \
  --wordlist=/usr/share/wordlists/rockyou.txt
```

**Combine /etc/passwd and /etc/shadow:**
```bash
# unshadow merges both files for John
unshadow /etc/passwd /etc/shadow > combined.txt

# Then crack
john combined.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

**View cracked results:**
```bash
# Show all cracked passwords
john hash.txt --show

# Show with specific format
john hash.txt --show --format=sha512crypt
```

**John with rules:**
```bash
# Apply best64 rules
john hash.txt --wordlist=rockyou.txt --rules=best64

# Apply Jumbo rules (more aggressive)
john hash.txt --wordlist=rockyou.txt --rules=jumbo

# Single mode (username-based mutations)
john hash.txt --single
```

**John format reference:**

| Hash Type | John Format Flag |
|-----------|----------------|
| Linux SHA-512 `$6$` | `--format=sha512crypt` |
| Linux MD5 `$1$` | `--format=md5crypt` |
| Linux SHA-256 `$5$` | `--format=sha256crypt` |
| Windows NTLM | `--format=NT` |
| MD5 raw | `--format=raw-md5` |
| SHA1 raw | `--format=raw-sha1` |
| bcrypt `$2b$` | `--format=bcrypt` |

---

### 5.5 — Hashcat — GPU Accelerated Cracking

Hashcat uses GPU power for extremely fast cracking. Far faster than John for large wordlists.

**Basic syntax:**
```bash
hashcat -m HASH_MODE -a ATTACK_MODE hash.txt wordlist.txt
```

**Hash Mode Reference (-m flag):**

| Mode | Hash Type | When Used |
|------|-----------|-----------|
| `0` | MD5 | Web apps, databases |
| `100` | SHA1 | Web apps |
| `500` | MD5crypt `$1$` | Old Linux |
| `1000` | NTLM | Windows SAM, AD |
| `1800` | SHA-512crypt `$6$` | Modern Linux — OSCP common |
| `3200` | bcrypt `$2b$` | Web apps (slow) |
| `5600` | NetNTLMv2 | Responder captures |
| `7400` | SHA-256crypt `$5$` | Linux |

**Attack Mode Reference (-a flag):**

| Mode | Type | Usage |
|------|------|-------|
| `0` | Dictionary | Wordlist attack |
| `1` | Combination | Two wordlists combined |
| `3` | Brute force (mask) | Pattern-based |
| `6` | Hybrid wordlist + mask | Wordlist + suffix |
| `7` | Hybrid mask + wordlist | Prefix + wordlist |

---

### 5.6 — Hashcat Examples

**Linux SHA-512 hash (most common in OSCP):**
```bash
# Save hash to file first
echo '$6$rounds=5000$salt$hashhashhashhash...' > hash.txt

# Crack with rockyou
hashcat -m 1800 -a 0 hash.txt \
  /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 1800 -a 0 hash.txt \
  /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# Show cracked result
hashcat -m 1800 hash.txt --show
```

**Windows NTLM hash:**
```bash
# Save NTLM hash (32-char hex)
echo 'aad3b435b51404eeaad3b435b51404ee' > ntlm.txt

# Crack
hashcat -m 1000 -a 0 ntlm.txt \
  /usr/share/wordlists/rockyou.txt

# With rules
hashcat -m 1000 -a 0 ntlm.txt rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule
```

**Mask (brute force) attack:**
```bash
# 6 character — all character types
hashcat -m 1800 -a 3 hash.txt ?a?a?a?a?a?a

# 8 character — lowercase + digits only
hashcat -m 1800 -a 3 hash.txt ?l?l?l?l?d?d?d?d

# Custom mask: starts with capital, 6 lower, 2 digits
hashcat -m 1800 -a 3 hash.txt ?u?l?l?l?l?l?d?d
```

**Mask character sets:**

| Symbol | Character Set |
|--------|--------------|
| `?l` | Lowercase a-z |
| `?u` | Uppercase A-Z |
| `?d` | Digits 0-9 |
| `?s` | Special chars |
| `?a` | All of above combined |

**NetNTLMv2 (from Responder):**
```bash
hashcat -m 5600 -a 0 netntlmv2.txt \
  /usr/share/wordlists/rockyou.txt
```

---

### 5.7 — Hash Cracking Decision Tree

```
[You Have a Hash]
        |
        v
[Identify hash type]
hash-identifier / hashid
        |
        v
  [Linux hash $6$?]
  YES /       \ NO
     /         \
[hashcat      [Windows NTLM?]
 -m 1800]     YES /     \ NO
                 /       \
           [hashcat    [MD5 or SHA1?]
            -m 1000]   hashcat -m 0
                       or -m 100
        |
        v
[Not cracked with rockyou?]
        |
        v
[Apply rules — best64.rule]
hashcat ... -r best64.rule
        |
        v
[Still not cracked?]
        |
        v
[Generate custom wordlist]
CeWL on target → john/hashcat
        |
        v
[Try mask attack]
hashcat -a 3 with patterns
```

---

## STEP 6: RAINBOW TABLES

### 6.1 — What are Rainbow Tables?

Rainbow tables are pre-computed lookup tables mapping hash values back to plaintext passwords. They trade storage space for zero computation time — instant results.

**How it works:**
```
[Password] → [Hash function] → [Hash]
"password"  →    MD5          → 5f4dcc3b5aa765d61d8327deb882cf99

Rainbow table stores:
5f4dcc3b5aa765d61d8327deb882cf99 → "password"
```

---

### 6.2 — Online Rainbow Table Services

| Site | Supported Hashes |
|------|-----------------|
| `crackstation.net` | MD5, SHA1, SHA256, NTLM, LM |
| `hashes.com` | MD5, SHA1, NTLM, many more |
| `md5decrypt.net` | MD5 focused |
| `ntlm.pw` | NTLM specific |

**OSCP workflow:**
```bash
# 1. Get hash
# 2. Go to crackstation.net
# 3. Paste hash → Submit
# 4. Common passwords crack instantly
# 5. If not found → use John/Hashcat offline
```

---

### 6.3 — Why Rainbow Tables Fail on Salted Hashes

```
Without salt:
"password" → MD5 → 5f4dcc3b... ← Rainbow table works

With salt (Linux /etc/shadow):
"password" + "$6$randomsalt$" → SHA-512 → unique_hash
                                             ↑
                              Rainbow table USELESS
                              Must use John or Hashcat
```

Linux `/etc/shadow` hashes are always salted — rainbow tables will never work. Always use John or Hashcat for these.

---

## STEP 7: CREDENTIAL REUSE + PASSWORD SPRAYING

### 7.1 — Credential Reuse — High Value OSCP Technique

When you find credentials on one service, **always try them everywhere else.**

```bash
# Found credentials: admin:Summer2023!
# Test on all open services immediately

ssh admin@192.168.1.10              # SSH
ftp 192.168.1.10                    # FTP (manual login)
smbclient //192.168.1.10/share -U admin  # SMB
curl -u admin:Summer2023! http://192.168.1.10/admin  # Web
```

---

### 7.2 — Password Spraying — What and Why

Password spraying is the reverse of brute force: **one password tested against many usernames**.

**Why?** Traditional brute force → many wrong passwords per account → **account lockout**
Password spray → one password per account → **no lockout triggered**

```
Brute force:   admin:pass1, admin:pass2, admin:pass3... → LOCKOUT
Password spray: admin:Pass123, user:Pass123, john:Pass123 → NO LOCKOUT
```

---

### 7.3 — Spraying Commands

```bash
# SMB password spray — most useful in OSCP
crackmapexec smb 192.168.1.10 \
  -u users.txt -p "Password123" --continue-on-success

# SSH password spray
hydra -L users.txt -p "Password123" \
  ssh://192.168.1.10 -t 4

# RDP password spray
hydra -L users.txt -p "Welcome1" \
  rdp://192.168.1.10 -t 4

# Multiple passwords but go slow (spray carefully)
crackmapexec smb 192.168.1.10 \
  -u users.txt -p passwords_short.txt --continue-on-success
```

**Common spray passwords to try:**
```
Password1
Password123
Welcome1
Company2024
Summer2024
Winter2024
Admin123!
P@ssword1
```

---

### 7.4 — Credential Reuse Flow

```
[Found FTP creds: admin:Company2023!]
              |
              v
[Try on every service]
    |         |         |        |
   SSH       SMB      Web      RDP
   |          |         |        |
Worked!     Worked!  Worked!  Worked!
   |          |         |
Shell!   File access  Admin panel
              |
         [Config file
          found with
          new creds]
              |
         [Repeat cycle]
```

---

## STEP 8: SERVICE-SPECIFIC ATTACK TIPS

### 8.1 — Default Credentials — Always Try First

**Before any brute force, manually test these:**

```
admin:admin
admin:password
admin:(blank)
admin:1234
root:root
root:toor
root:(blank)
guest:guest
user:user
test:test
```

---

### 8.2 — Default Credential Resources

| Resource | URL |
|----------|-----|
| DefaultCreds Cheat Sheet | `github.com/ihebski/DefaultCreds-cheat-sheet` |
| Default Password DB | `default-password.info` |
| CIRT Password List | `cirt.net/passwords` |

---

### 8.3 — Web Application Default Credentials

| Application | Default Credentials |
|-------------|-------------------|
| WordPress | `admin:admin`, `admin:password` |
| Apache Tomcat | `tomcat:tomcat`, `admin:admin`, `manager:manager` |
| Jenkins | `admin:admin`, or no auth |
| phpMyAdmin | `root:` (blank), `root:root` |
| DVWA | `admin:password` |
| Joomla | `admin:admin` |
| Drupal | `admin:admin` |
| WebMin | `admin:admin` |
| Grafana | `admin:admin` |
| GitLab | `root:5iveL!fe` (older versions) |

---

### 8.4 — SNMP Community String Brute Force

SNMP v1/v2 uses "community strings" as passwords. Default is almost always `public`.

```bash
# Brute force community strings
onesixtyone -c /usr/share/seclists/Discovery/SNMP/snmp.txt \
  192.168.1.10

# Manual test with common strings
onesixtyone -c - 192.168.1.10 <<< "public
private
manager
community
snmp
cisco"

# Once you find the string, enumerate with snmpwalk
snmpwalk -c public -v2c 192.168.1.10
```

---

### 8.5 — Service Attack Priority Order

```
[New service discovered]
         |
         v
1. Check default credentials (manual — 2 min)
         |
         v
2. Search for known default creds for that service
         |
         v
3. Try credential reuse from other services
         |
         v
4. Password spray with common passwords
         |
         v
5. Full brute force with rockyou.txt
         |
         v
6. Custom wordlist (CeWL) + brute force
         |
         v
7. Move on — come back after more enum
```

---

## STEP 9: COMPLETE ATTACK SCENARIO — START TO FINISH

### 9.1 — Full Scenario: Linux Machine

**Target:** 192.168.1.10
**Open ports:** 22 (SSH), 80 (HTTP)

---

**STEP 1 — Enumerate web application:**
```bash
gobuster dir -u http://192.168.1.10 \
  -w /usr/share/wordlists/dirb/common.txt
# Found: /admin
```

**STEP 2 — Test default credentials manually:**
```
admin:admin → FAILED
admin:password → FAILED
admin:(blank) → FAILED
```

**STEP 3 — Build custom wordlist from target site:**
```bash
cewl http://192.168.1.10 -m 5 -w custom.txt
wc -l custom.txt
# 347 words
```

**STEP 4 — Identify login form parameters:**
- Open browser → go to `http://192.168.1.10/admin`
- F12 → Network tab → Submit wrong login
- Note: POST to `/admin`, params `user=` and `pass=`, error says "Wrong credentials"

**STEP 5 — Hydra HTTP POST brute force:**
```bash
hydra -l admin -P custom.txt 192.168.1.10 \
  http-post-form "/admin:user=^USER^&pass=^PASS^:Wrong credentials" -f
# [SUCCESS] login: admin  password: Company2023!
```

**STEP 6 — Credential reuse on SSH:**
```bash
ssh admin@192.168.1.10
# password: Company2023!
# SUCCESS — shell obtained
```

**STEP 7 — Read /etc/shadow:**
```bash
sudo cat /etc/shadow
# root:$6$rounds=5000$randomsalt$hashhashhashhashhashhashhashhash...:
```

**STEP 8 — Crack the root hash:**
```bash
# Save hash
echo '$6$rounds=5000$randomsalt$hashhashhashhash...' > root_hash.txt

# Crack with hashcat
hashcat -m 1800 -a 0 root_hash.txt \
  /usr/share/wordlists/rockyou.txt

# Show result
hashcat -m 1800 root_hash.txt --show
# root_hash.txt:toor123
```

**STEP 9 — Escalate to root:**
```bash
su root
# Password: toor123
whoami
# root
```

---

## PA1 CHECKLIST

```
[ ] rockyou.txt location known and extracted
[ ] CeWL custom wordlist generated from a target
[ ] Hydra SSH brute force practiced
[ ] Hydra HTTP POST form attack with failure string
[ ] Hash type identification with hashid / hash-identifier
[ ] John the Ripper cracked /etc/shadow hash
[ ] Hashcat -m 1800 SHA-512 crack performed
[ ] Crackstation.net tested with a hash
[ ] Default credentials checklist memorized
[ ] Credential reuse scenario practiced end-to-end
[ ] Password spraying with CrackMapExec practiced
[ ] Medusa used as Hydra alternative
```

---

## STUDENT SELF-TEST

> **Question:** You found this hash from `/etc/shadow`:
> `$6$rounds=5000$salt$hashhashhashhash`
>
> Which tool will you use?
> Which `-m` flag?
> Which wordlist to start with?
> Write the exact command.

**Answer:**
```bash
# Tool: Hashcat (GPU accelerated, faster than John for this)
# -m flag: 1800 (SHA-512crypt = $6$)
# Wordlist: rockyou.txt to start

# Save the hash
echo '$6$rounds=5000$salt$hashhashhashhash' > hash.txt

# Run hashcat
hashcat -m 1800 -a 0 hash.txt /usr/share/wordlists/rockyou.txt

# If not found, add rules
hashcat -m 1800 -a 0 hash.txt /usr/share/wordlists/rockyou.txt \
  -r /usr/share/hashcat/rules/best64.rule

# View result
hashcat -m 1800 hash.txt --show

# Alternative with John:
john hash.txt --format=sha512crypt \
  --wordlist=/usr/share/wordlists/rockyou.txt
john hash.txt --show
```

---

## QUICK REFERENCE CARD

```
TOOL         USE CASE                    KEY FLAG
---------    -------------------------   --------
hydra        Online brute force          -l/-L -p/-P -t -f
medusa       Online brute force alt      -h -u/-U -P -M
crackmapexec SMB/WinRM attack           -u -p --continue-on-success
john         Offline hash crack          --format= --wordlist= --rules=
hashcat      Offline hash crack (GPU)    -m -a -r --show
cewl         Custom wordlist from site   -m -d -w
crunch       Generate wordlist           min max charset -t -o
hash-id      Identify hash type          (interactive)
hashid       Identify hash type          hashid 'HASH'

HASH MODES (hashcat -m):
0    = MD5
100  = SHA1
500  = md5crypt $1$
1000 = NTLM
1800 = sha512crypt $6$  ← MOST COMMON IN OSCP
3200 = bcrypt $2b$
5600 = NetNTLMv2
```

---

*PA1 COMPLETE*

**Next:** Use PA2 — Advanced Hash Attacks + Kerberoasting + Pass-the-Hash
