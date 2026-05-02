# OSINT for Penetration Testing — Technical Reference

A comprehensive, hands-on reference for Open Source Intelligence (OSINT) methodologies, tools, and techniques used in professional penetration testing engagements. Built from real-world research and lab work while preparing for the **Practical Network Penetration Tester (PNPT)** certification by TCM Security.

> **Disclaimer:** All techniques documented here are intended strictly for authorized penetration testing engagements, security research, and educational purposes. Always obtain explicit written permission before conducting reconnaissance against any target. Unauthorized OSINT collection may violate local and international laws including the CFAA, GDPR, and regional cybercrime statutes.

---

## Table of Contents

1. [Methodology Overview](#methodology-overview)
2. [Passive Reconnaissance](#passive-reconnaissance)
3. [DNS and WHOIS Enumeration](#dns-and-whois-enumeration)
4. [Subdomain Enumeration](#subdomain-enumeration)
5. [Google Dorking](#google-dorking)
6. [Shodan and Censys](#shodan-and-censys)
7. [Email and People Intelligence](#email-and-people-intelligence)
8. [Credential and Breach Intelligence](#credential-and-breach-intelligence)
9. [Social Media and LinkedIn Recon](#social-media-and-linkedin-recon)
10. [Web Technology Fingerprinting](#web-technology-fingerprinting)
11. [Certificate Transparency Logs](#certificate-transparency-logs)
12. [Wayback Machine and Historical Data](#wayback-machine-and-historical-data)
13. [Image and Metadata OSINT](#image-and-metadata-osint)
14. [Automated OSINT Frameworks](#automated-osint-frameworks)
15. [Building a Target Profile](#building-a-target-profile)
16. [Transitioning to Active Reconnaissance](#transitioning-to-active-reconnaissance)
17. [Tools Reference](#tools-reference)
18. [Resources](#resources)

---

## Methodology Overview

OSINT forms the foundation of every professional penetration test. The goal of this phase is to maximize information gathering with zero direct interaction with target systems. A thorough OSINT phase directly informs every subsequent attack phase. Subdomains become targets. Harvested emails become phishing lures. Breached credentials become password spray lists. Exposed services become initial access vectors.

### OSINT Penetration Testing Workflow

```
Define Scope
     |
     v
Passive DNS and WHOIS Enumeration
     |
     v
Subdomain and Certificate Discovery
     |
     v
Employee and Credential Harvesting
     |
     v
Technology Fingerprinting
     |
     v
Compile Attack Surface Map
     |
     v
Hand Off to Active Reconnaissance
```

### Key Principles

- Never interact directly with target systems during passive recon
- Document every finding with timestamps and source URLs
- Verify findings across multiple sources before acting on them
- Passive recon never triggers IDS, IPS, or firewall alerts
- Understand legal boundaries before beginning any engagement

---

## Passive Reconnaissance

Passive reconnaissance involves collecting publicly available information without sending a single packet to the target network.

### Key Data Points to Collect

```
Target Organization:
- Full legal name and subsidiaries
- Physical locations and offices
- Technology stack (job postings, Wappalyzer, Shodan)
- Key personnel (executives, IT staff, developers)
- Email format (firstname.lastname@company.com)
- Domain names and IP ranges (WHOIS, ASN lookup)
- Exposed services (Shodan, Censys)
- Breached credentials (HaveIBeenPwned, DeHashed)
```

### ASN and IP Range Lookup

```bash
# Find ASN for organization
whois -h whois.radb.net -- '-i origin AS<number>'

# BGP toolkit via API
curl https://api.bgpview.io/search?query_term=TargetOrg

# Hurricane Electric BGP Toolkit
https://bgp.he.net
```

---

## DNS and WHOIS Enumeration

### WHOIS Lookup

```bash
# Basic WHOIS
whois target.com

# WHOIS on IP address
whois 192.168.1.1

# ARIN lookup for IP ownership
whois -h whois.arin.net 192.168.1.1
```

WHOIS records expose registrant name, organization, email address, phone number, registrar, creation and expiry dates, and nameservers. Historical WHOIS data available via DomainTools can reveal previous owners and contact information even after privacy protection is applied.

### DNS Enumeration

```bash
# Full DNS record enumeration
dig target.com ANY

# Mail server records
dig target.com MX

# Name server records
dig target.com NS

# TXT records (SPF, DMARC, verification tokens)
dig target.com TXT

# Reverse DNS lookup
dig -x 192.168.1.1

# Zone transfer attempt against misconfigured DNS servers
dig axfr @ns1.target.com target.com
host -t axfr target.com ns1.target.com
```

### DNS Brute Forcing

```bash
# dnsrecon
dnsrecon -d target.com -t brt \
  -D /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt

# fierce
fierce --domain target.com

# dnsx
dnsx -d target.com -w wordlist.txt -o dns_output.txt
```

### DNSDumpster

```
https://dnsdumpster.com
```

Provides a visual map of DNS infrastructure including subdomains, mail servers, host records, and MX records with zero interaction with the target.

---

## Subdomain Enumeration

Subdomain enumeration expands the attack surface by discovering staging environments, development servers, forgotten admin panels, and legacy applications.

### Passive Subdomain Enumeration

```bash
# Sublist3r aggregates multiple search engines
sublist3r -d target.com -o subdomains.txt

# Amass passive mode
amass enum -passive -d target.com -o amass_passive.txt

# Assetfinder
assetfinder --subs-only target.com

# theHarvester
theHarvester -d target.com -b google,bing,yahoo,shodan
```

### Active Subdomain Bruteforcing

```bash
# Gobuster DNS mode
gobuster dns -d target.com \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -o gobuster_dns.txt

# ffuf virtual host fuzzing
ffuf -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt \
  -u https://FUZZ.target.com \
  -mc 200,301,302,403

# Amass active brute force
amass enum -active -d target.com -brute -o amass_active.txt
```

### Subdomain Takeover Detection

```bash
# subjack
subjack -w subdomains.txt -t 100 -timeout 30 -o takeovers.txt -ssl

# nuclei subdomain takeover templates
nuclei -l subdomains.txt -t takeovers/
```

---

## Google Dorking

Google dorks use advanced search operators to find sensitive information indexed by search engines. Entirely passive and never triggers target-side alerts.

### Core Operators

```
site:       Restrict results to a specific domain
filetype:   Search for specific file extensions
inurl:      Search within URLs
intitle:    Search within page titles
intext:     Search within page body
cache:      View cached version of a page
```

### High Value Dorks

```bash
# Enumerate all subdomains
site:target.com

# Find login portals
site:target.com inurl:login
site:target.com inurl:admin
site:target.com intitle:"login"

# Find exposed sensitive files
site:target.com filetype:pdf
site:target.com filetype:xls OR filetype:xlsx
site:target.com filetype:sql
site:target.com filetype:log
site:target.com filetype:env
site:target.com filetype:bak
site:target.com filetype:conf OR filetype:config

# Find directory listings
site:target.com intitle:"index of"
site:target.com intitle:"index of /" "parent directory"

# Find email addresses
"@target.com" filetype:xls
"@target.com" filetype:csv

# Find exposed credentials
site:target.com intext:password filetype:log
site:target.com intext:"api_key" OR intext:"api_secret"

# Find GitHub exposed data
site:github.com "target.com" password
site:github.com "target.com" secret
site:github.com "target.com" api_key
```

### Google Hacking Database

```
https://www.exploit-db.com/google-hacking-database
```

The GHDB contains thousands of verified dorks organized by category including footholds, files containing passwords, sensitive directories, and vulnerable servers.

---

## Shodan and Censys

Shodan and Censys index internet-facing devices and services globally, providing passive visibility into a target's external attack surface without any interaction with the target network.

### Shodan CLI

```bash
# Install
pip install shodan
shodan init <API_KEY>

# Search by organization
shodan search org:"Target Organization"

# Search by hostname
shodan search hostname:target.com

# Detailed host information
shodan host 192.168.1.1

# Find specific exposed services
shodan search hostname:target.com port:3389
shodan search hostname:target.com port:22
shodan search "apache" hostname:target.com

# Find exposed databases
shodan search "mongodb" org:"Target Organization"
shodan search "elasticsearch" org:"Target Organization"

# Download and parse results
shodan download results.json.gz org:"Target Organization"
shodan parse --fields ip_str,port,org results.json.gz
```

### High Value Shodan Dorks

```
# Exposed Remote Desktop Protocol
port:3389 org:"Target"

# Jenkins CI servers
http.title:"Dashboard [Jenkins]"

# Exposed Kibana dashboards
http.title:"Kibana"

# Exposed Elasticsearch instances
port:9200 json country:"US"

# Citrix Gateway login pages
title:"Citrix Gateway"

# Pulse Secure VPN
http.html:"Pulse Secure" port:443

# VNC with no authentication
port:5900 authentication disabled
```

### Censys

```bash
# Certificate search
certificates.parsed.names: target.com

# Open port search
services.port: 3389 AND autonomous_system.name: "Target Organization"
```

---

## Email and People Intelligence

### theHarvester

```bash
# Harvest emails, subdomains, hosts, and employee names
theHarvester -d target.com -b google
theHarvester -d target.com -b linkedin
theHarvester -d target.com -b shodan

# All sources combined with HTML report
theHarvester -d target.com -b all -f output.html
```

### Hunter.io

```
https://hunter.io
```

Discovers email addresses associated with a domain, identifies the email format used by the organization, and provides confidence scores for each address. The domain search endpoint returns all publicly known email addresses along with the sources where they were found.

### Phonebook.cz

```
https://phonebook.cz
```

Searches for all known email addresses, domains, and URLs associated with a target domain. Useful for bulk email harvesting across large organizations.

### Email Format Identification

```bash
# Common enterprise email formats to test
firstname.lastname@target.com
f.lastname@target.com
firstname@target.com
flastname@target.com

# CrossLinked — generate username lists from LinkedIn
python3 crosslinked.py -f '{first}.{last}@target.com' "Target Company"
```

---

## Credential and Breach Intelligence

Breached credential databases are among the most valuable OSINT sources for penetration testers. Discovered credentials can be used directly in password spraying attacks or used to build targeted wordlists for hash cracking.

### HaveIBeenPwned

```bash
# Web interface
https://haveibeenpwned.com
https://haveibeenpwned.com/Passwords

# API query for domain breach exposure
curl -s "https://haveibeenpwned.com/api/v3/breacheddomain/target.com" \
  -H "hibp-api-key: <API_KEY>"
```

### DeHashed

```
https://dehashed.com
```

Searches across breach databases and returns plaintext passwords, hashed passwords, usernames, IP addresses, and associated emails. Requires a subscription for full results.

### BreachDirectory

```
https://breachdirectory.org
```

Free breach lookup service providing partial plaintext passwords and SHA-1 hashes from known breach datasets.

### Building a Target-Specific Wordlist

```bash
# Generate organization-specific wordlist with CeWL
cewl https://target.com -d 3 -m 6 -w custom_wordlist.txt

# Combine with standard wordlists
cat custom_wordlist.txt /usr/share/wordlists/rockyou.txt | sort -u > final_wordlist.txt

# Apply hashcat mutation rules
hashcat --stdout custom_wordlist.txt \
  -r /usr/share/hashcat/rules/best64.rule > mutated_wordlist.txt
```

---

## Social Media and LinkedIn Recon

### LinkedIn Reconnaissance

LinkedIn is the most valuable OSINT source for corporate penetration testing. It exposes organizational structure, key personnel, internal technologies, and security team members.

**High Value Targets on LinkedIn:**

```
- C-Suite and executive team (high-value phishing targets)
- IT and security personnel (access to critical systems)
- Developers (GitHub profiles, code repositories)
- HR personnel (likely to open email attachments)
- Recently hired employees (reduced security awareness)
- Job postings (reveal internal technology stack in detail)
```

**Technology Discovery via Job Postings:**

Job postings frequently expose internal infrastructure including operating systems and versions, virtualization platforms, security tools in use (SIEM, EDR, firewalls), programming languages, cloud providers, and internal software and ERP systems.

### LinkedIn OSINT Tools

```bash
# linkedin2username — generate username lists from LinkedIn profiles
python3 linkedin2username.py \
  -u <your_linkedin_email> \
  -p <password> \
  -c "Target Company"

# CrossLinked — username enumeration without API
python3 crosslinked.py -f '{first}.{last}@target.com' "Target Company"
```

### Twitter OSINT

```bash
# Twint — Twitter scraping without API key
twint -u target_username --email --phone
twint -s "target.com" --since 2024-01-01

# Google dorks for Twitter and Reddit
site:twitter.com "target.com" password
site:reddit.com "target.com" VPN
```

---

## Web Technology Fingerprinting

Identifying the technology stack of a target web application enables targeted exploit research and precise attack planning.

### WhatWeb

```bash
# Basic fingerprint
whatweb target.com

# Aggressive mode
whatweb -a 3 target.com

# Scan multiple targets from file
whatweb -i targets.txt --log-verbose=output.txt
```

### WAF Detection

```bash
# wafw00f
wafw00f target.com
wafw00f -a target.com    # Test all WAF signatures

# nmap WAF detection scripts
nmap -p 80,443 --script http-waf-detect target.com
nmap -p 80,443 --script http-waf-fingerprint target.com
```

### HTTP Header Analysis

```bash
# Inspect response headers
curl -I https://target.com
curl -I -L https://target.com

# Extract specific security-relevant headers
curl -v https://target.com 2>&1 | \
  grep -E "Server:|X-Powered-By:|Set-Cookie:|X-Frame-Options:|Strict-Transport-Security:"
```

---

## Certificate Transparency Logs

SSL/TLS certificates are publicly logged, revealing subdomains, internal hostnames, and infrastructure details that may not appear in standard DNS records.

### crt.sh

```bash
# Web interface
https://crt.sh/?q=%.target.com

# API query and subdomain extraction
curl -s "https://crt.sh/?q=%.target.com&output=json" | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
names = set()
for entry in data:
    for name in entry['name_value'].split('\n'):
        names.add(name.strip())
for name in sorted(names):
    print(name)
"
```

### Google Transparency Report

```
https://transparencyreport.google.com/https/certificates
```

---

## Wayback Machine and Historical Data

The Wayback Machine archives historical versions of websites, revealing old endpoints, deprecated admin panels, exposed configuration files, and legacy applications that may still be accessible.

### Manual Inspection

```
https://web.archive.org/web/*/target.com
https://web.archive.org/web/*/target.com/admin
https://web.archive.org/web/*/target.com/wp-admin
```

### Waybackurls

```bash
# Install
go install github.com/tomnomnom/waybackurls@latest

# Extract all archived URLs
waybackurls target.com | tee wayback_urls.txt

# Filter for interesting endpoints
cat wayback_urls.txt | grep -E "\.php|\.asp|\.aspx|admin|login|config|backup"

# Filter for sensitive file extensions
cat wayback_urls.txt | grep -E "\.sql|\.bak|\.env|\.log|\.conf|\.xml|\.json"
```

### GAU (Get All URLs)

```bash
# Install
go install github.com/lc/gau/v2/cmd/gau@latest

# Fetch URLs from multiple sources
gau target.com | tee gau_output.txt

# Combine sources and deduplicate
cat <(waybackurls target.com) <(gau target.com) | sort -u > all_urls.txt
```

---

## Image and Metadata OSINT

### EXIF Data Extraction

Images posted publicly by target organizations may contain embedded EXIF metadata including GPS coordinates, device information, software versions, and author details.

```bash
# Extract all metadata
exiftool image.jpg

# Recursive directory scan
exiftool -r /path/to/images/

# Extract GPS coordinates specifically
exiftool -GPSLatitude -GPSLongitude image.jpg
```

### Metagoofil — Document Metadata

```bash
# Extract metadata from publicly indexed documents
metagoofil -d target.com -t pdf,doc,xls,ppt -l 20 -o output/
```

Reveals author names, software versions, internal file paths, and usernames from documents indexed by search engines.

### Reverse Image Search

```
Google Images:   https://images.google.com
TinEye:          https://tineye.com
Yandex Images:   https://yandex.com/images
```

---

## Automated OSINT Frameworks

### Recon-ng

Full-featured web reconnaissance framework with modular architecture similar to Metasploit.

```bash
# Launch recon-ng
recon-ng

# Install all modules
marketplace install all

# Create a workspace
workspaces create target_com

# Add target domain
db insert domains
> domain: target.com

# Run modules sequentially
modules load recon/domains-hosts/hackertarget
run

modules load recon/hosts-hosts/resolve
run

modules load recon/domains-contacts/whois_pocs
run

# Review all findings
show hosts
show contacts
show credentials
```

### SpiderFoot

```bash
# Install
pip install spiderfoot

# Launch web interface
spiderfoot -l 127.0.0.1:5001

# CLI scan
spiderfoot -s target.com -t all -o output.json
```

### Maltego

Graph-based OSINT tool for visualizing relationships between domains, IP addresses, people, organizations, and infrastructure. Particularly effective for mapping the full organizational attack surface and identifying trust relationships between entities. Available natively in Kali Linux.

---

## Building a Target Profile

After completing passive reconnaissance, compile all findings into a structured attack surface map before transitioning to active reconnaissance.

### Target Profile Template

```
Organization:         [Name]
Primary Domain:       target.com
Subdomains:           [list from enumeration]
IP Ranges:            [CIDR blocks from ASN lookup]
ASN:                  AS[number]
Nameservers:          [ns1, ns2]
Mail Servers:         [MX records]
SPF Record:           [value]
DMARC Policy:         [value]

Key Personnel:
  CEO:                [name, email, LinkedIn URL]
  IT Manager:         [name, email, LinkedIn URL]
  Developers:         [names, GitHub profiles]

Email Format:         firstname.lastname@target.com
Confirmed Addresses:  [list]
Breached Accounts:    [list from HaveIBeenPwned]

Technology Stack:
  Web Server:         [Apache/Nginx/IIS + version]
  CMS:                [WordPress/Drupal/custom]
  WAF:                [vendor if detected]
  Cloud Provider:     [AWS/Azure/GCP]
  CDN:                [Cloudflare/Akamai]

Exposed Services:     [from Shodan/Censys]
Certificate History:  [from crt.sh]
Historical Endpoints: [from Wayback Machine]
```

---

## Transitioning to Active Reconnaissance

After exhausting passive OSINT, transition to active reconnaissance using the attack surface map as a guide.

```bash
# Port scanning confirmed IP ranges
nmap -T4 -A -p- <target_ip_range>

# Web directory bruteforcing on discovered subdomains
gobuster dir -u https://subdomain.target.com \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt

# Vulnerability scanning
nikto -h https://target.com

# Use harvested credentials in password spraying
crackmapexec smb target_range.txt -u users.txt -p passwords.txt
```

---

## Tools Reference

| Tool | Category | Purpose |
|---|---|---|
| theHarvester | Email/Subdomain | Harvests emails, subdomains, hosts from public sources |
| Maltego | Visualization | Graph-based link analysis across OSINT sources |
| Shodan | Infrastructure | Internet-facing asset and service discovery |
| Censys | Infrastructure | Certificate and service enumeration |
| Recon-ng | Framework | Modular web reconnaissance framework |
| SpiderFoot | Framework | Automated multi-source OSINT collection |
| Amass | Subdomain | In-depth subdomain enumeration and mapping |
| Sublist3r | Subdomain | Fast passive subdomain enumeration |
| Gobuster | Brute Force | DNS and directory brute forcing |
| ffuf | Brute Force | Fast web fuzzer for virtual host and directory discovery |
| Sherlock | Username | Username enumeration across 300+ platforms |
| CrossLinked | LinkedIn | Employee username list generation from LinkedIn |
| WhatWeb | Fingerprinting | Web technology identification |
| wafw00f | Fingerprinting | Web application firewall detection |
| CeWL | Wordlist | Custom wordlist generation from target websites |
| Metagoofil | Metadata | Document metadata extraction |
| exiftool | Metadata | Image and file EXIF data extraction |
| waybackurls | Historical | Archived URL extraction from Wayback Machine |
| GAU | Historical | URL collection from multiple archive sources |
| dnsrecon | DNS | DNS enumeration and zone transfer testing |
| fierce | DNS | DNS reconnaissance and subdomain discovery |

---

## Resources

### Documentation and References

- OSINT Framework: https://osintframework.com
- TCM Security PNPT Course: https://academy.tcm-sec.com
- PayloadsAllTheThings: https://github.com/swisskyrepo/PayloadsAllTheThings
- HackTricks: https://book.hacktricks.xyz
- SecLists: https://github.com/danielmiessler/SecLists
- Google Hacking Database: https://www.exploit-db.com/google-hacking-database

### Breach Databases

- HaveIBeenPwned: https://haveibeenpwned.com
- DeHashed: https://dehashed.com
- BreachDirectory: https://breachdirectory.org

### Passive Recon Services

- Shodan: https://shodan.io
- Censys: https://search.censys.io
- DNSDumpster: https://dnsdumpster.com
- crt.sh: https://crt.sh
- Wayback Machine: https://web.archive.org
- Hunter.io: https://hunter.io
- Phonebook.cz: https://phonebook.cz

---

## Author
Jay Shalwala 
Studying for the PNPT certification by TCM Security. This repository documents practical OSINT techniques applied in real lab environments and authorized engagements.
