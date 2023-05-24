# Active Directory Enumeration & Attacks
Despite being a robust and secure system, Active Directory (AD) can be considered vulnerable in specific scenarios as it is susceptible to various threats, including external attacks, credential attacks, and privilege escalation. In this walkthrough, I will demonstrate what steps I took in this Hack The Box academy module.

![AD](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/53e60b80-3610-4877-b434-534d011eed96)


## Tasks

- [x] #1 Initial Enumeration 
- [x] #2 LLMNR/NBT-NS Poisoning 
- [ ] #3 Enumerating, Retrieving Password Policis & Password Spraying
- [ ] #4 Enumerating Security Controls
- [ ] #5 Kerberoasting


## #1 Initial Enumeration
## **1.1 External Reconnaissance and enumeration principles**

*Goal: Validating information provided in the scoping document and looking for publicly accessible information*

### What to look for

+ **IP Space** (Valid ASN cloud presence, hosting providers, DNS record entries, etc.)
+ **Domain information** (Based on IP data, DNS, and site registrations. Domain admins, subdomains, publicly accessible domain services like mail servers, vpn portals, etc. Determining what kind of defenses are in place (SIEM, AV, IPS/IDS, etc.)
+ **Schema Format** (Discovering the organization's email accounts, AD usernames, and password policies)
+ **Data Disclosures** (Publicly accessible files like .pdf and .docx, intranet site listing, user metadata, shares, etc.)
+ **Breach Data** (Publicly released usernames, passwords, or critical information)

### Where to look

+ **ASN/IP Registrars** - [IANA](https://www.iana.org/whois), [BGP Toolkit](https://bgp.he.net/)
+ **Domain Registrars and DNS** - [ICANN](https://lookup.icann.org/en), [DomainTools](https://whois.domaintools.com/)
+ **Social Media, public-facing company websites, and job listings**
+ **Breach Data Sources** - [Haveibeenpwned](https://haveibeenpwned.com/)
+ **Cloud and Dev. Storage Space - Github, AWS buckets/Azure Blob** - [GreyHatWarfare](https://grayhatwarfare.com/), [TruffleHog](https://github.com/trufflesecurity/trufflehog)

### Questions
***While looking at inlanefreights public records; A flag can be seen. Find the flag and submit it. ( format == HTB{\******\} )**
> $dig inlanefreight.com any

![dig](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/1f32fbdb-5fd4-48b2-8cd4-1c53ccdfe52d)

## **1.2 Initial Enumeration of the domain**

*Goal: Enumerate the internal network, identifying hosts, critical services, and potencial avenues for a foothold*

**For this first portion of the test, we are starting on an attack host placed inside the network. We have not been provided credentials or an internal network map.**

We will start with passive identification of any hosts in the network, followed by active validation of the results to find out more about each host. This section requires us to SSH into Hack The Box's network, then RDP into their attacking machine using. We will use xfreerdp to start the engagement.

>$xfreerdp /u:htb-student /p[redacted] /v:10.129.95.237

Once we connect to the attacking machine (Parrot OS), we will start capturing traffic with **Wireshark**, and use **tcpdump** and **Responder** to validate the information, and build an understanding on what hosts this network consists. Then, finally, we will fping the network to find live hosts.

![rdp](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/5b2ee708-9688-49fc-94c4-7e6ccc706d12)
![responder](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/d0e07bfb-b20e-424b-bb9e-f2178cfe81e0)
![fping](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/1a3d1a0a-4ce3-433f-8e89-9eb07b0a4aa7)

Now that we have confirmation on three (3) live hosts in this network, we perform an **nmap** scan on them to find the answers to the below questions.

### Questions
***From your scans, what is the "commonName" of host 172.16.5.5?***
> 172.16.5.x

***What host is running "Microsoft SQL Server 2019 15.00.2000.00"? (IP address, not Resolved name)***
> 172.16.5.x

## **1.3 Kerbrute - Internal AD Username Enumeration**

*We were not provided with an user to start testing with, so we will use [Kerbrute](https://github.com/ropnop/kerbrute), and the statistically likely usernames list from [insidetrust](https://github.com/insidetrust/statistically-likely-usernames) to enumerate usernames.*

>$kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o usernames

![kerbrute1](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/971b2ce5-8d5d-446b-886e-4529e624ff7e)
![kerbrute2](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/fe5a69e8-ad5c-448a-ad7e-aceea43e9d96)


56 valid user accounts found!

## #2 LLMNR/NBT-NS Poisoning

> "Link-Local Multicast Name Resolution (LLMNR) and NetBIOS Name Service (NBT-NS) are Microsoft Windows components that serve as alternate methods of host identification that can be used when DNS fails."

Example:
- A host attempts to connect to the print server at \\print01.inlanefreight.local, but accidentally types in \\printer01.inlanefreight.local 
- The DNS server responds, stating that this host is unknown.
- The host then broadcasts out to the entire local network asking if anyone knows the location of \\printer01.inlanefreight.local.
- The attacker (us with Responder running) responds to the host stating that it is the \\printer01.inlanefreight.local that the host is looking for.
- The host believes this reply and sends an authentication request to the attacker with a username and NTLMv2 password hash.
- This hash can then be cracked offline or used in an SMB Relay attack if the right conditions exist.

### Tools of the trade

+ [Responder](https://github.com/lgandx/Responder)
+ [Inveigh](https://github.com/Kevin-Robertson/Inveigh)
+ [Metasploit](https://www.metasploit.com/)

>$sudo responder -I ens224

![responder2](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/079bba83-deb3-45f1-8c46-b6731ca14355)
![responder1](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/06205d2e-5196-465c-88f4-2c04c8940d4c)


Now that we have a couple password NetNTLMv2 hashes, we will crack them offline using **Hashcat** using the wordlist **rockyou**.

>$hashcat -m 5600 backupagent /usr/share/wordlists/rockyou.txt

![hashcat1](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/546f2494-342e-4988-bd7c-db37534c23ae)
![hashcat2](https://github.com/LaraBruno/Active-Directory-Enumeration-Attacks/assets/37584600/60b120ab-4e2f-4411-919f-0cc93498e19a)

### Questions
***Run Responder and obtain a hash for a user account that starts with the letter b. Submit the account name as your answer.***
>backupagent

***Crack the hash for the previous account and submit the cleartext password as your answer.***
>[redacted]

***Run Responder and obtain an NTLMv2 hash for the user wley. Crack the hash using Hashcat and submit the user's password as your answer.***
>[redacted]

## #3 Enumerating, Retrieving Password Policis & Password Spraying

*Soon :)*
