# Active Directory Ports and Services

Reference for the network ports (the "doors") a domain client needs open to a domain controller. Every door answers one question: what does a computer need in order to log a user in and stay part of the domain?

## The mental model

- An IP address gets you to the building (the DC). Ports are the numbered doors on that building.
- Each service listens behind its own door, by convention always the same number.
- TCP vs UDP are two styles of knocking:
  - TCP = the formal knock (knock, wait, confirm, then talk). Reliable; used when data must arrive intact.
  - UDP = the casual knock (shout the question, hope for a shout back). Fast; used for quick lookups.
- Some services listen for BOTH styles, which is why a port like 53 shows up on both the TCP and UDP allow rules. One MikroTik rule can only describe one style, so allowing "the AD doors" takes two rules (one TCP, one UDP).

## The essential doors

### 53 - DNS (the phone book)
Turns names into addresses. The client only knows the name landon.lab; it needs the DC's address. Nothing else works until the client can FIND the DC. Block this and the client is lost.

### 88 - Kerberos (the ID check / the login itself)
This IS the login. Verifies the password and issues a ticket proving who the user is; that ticket is then shown to other services instead of re-typing the password. The single most important door for authentication. Block 88 and logins fail outright.

### 389 - LDAP (the directory lookup)
How a client READS the directory: which groups is this user in, what policy applies, where does the user belong. Block it and the client can authenticate but cannot determine what the user is allowed to do.

### 445 - SMB (file delivery + Group Policy)
Two jobs: file shares (mapped network drives), and - more importantly for the domain - delivering Group Policy from the DC's SYSVOL share. Block it and Group Policy silently stops applying.

### 464 - Kerberos password changes
The specific door for CHANGING a password (separate from the login at 88). The "must change password at next login" flow uses this.

### 135 + 49152-65535 - RPC (the receptionist and the back offices)
Some Windows services do not sit at a fixed door; they get a random high-numbered door each time. The client asks the receptionist at door 135 "which door is service X at today?" and is sent to a high door in the 49152-65535 range. That is why both 135 AND the big high range must be open. The wide range feels uncomfortable but is the standard AD requirement, and it is only opened toward the server subnet.

### 636, 3268, 3269 - LDAP variants
Fancier versions of 389. 636 is encrypted LDAP. 3268/3269 are the Global Catalog (a domain-wide lookup used in bigger multi-domain setups). Barely used in a single-domain lab, but included because they are standard AD and make the rule future-proof if the domain grows.

### 123 - NTP (the clock) [UDP]
Time sync. Kerberos refuses to work if client and DC clocks drift more than about 5 minutes apart (it assumes something shady). So NTP keeps clocks matched, which keeps logins working. Famous real-world AD breakage: "why can't anyone log in? the time is off."

## The two allow rules built in the lab

TCP door list:
```
53,88,135,389,445,464,636,3268,3269,49152-65535
```

UDP door list:
```
53,88,123,389,464
```

(UDP is a shorter list - fewer services use the casual knock.)

## One-sentence summary

A client needs to find the DC (53), prove who the user is (88), read the directory (389), receive files and policy (445), and keep its clock synced (123) - plus a few supporting and encrypted variants. Everything else is blocked.

## Quick self-check
- Which door is the actual login? -> 88 (Kerberos)
- Which door breaks logins if clocks drift? -> 123 (NTP), because Kerberos depends on synced time.

Learned in: [[Sessions/2026-07-16 Client Join and Inter-VLAN Firewall]]
See also: [[Topics/Firewall-Rules-Concepts]]
