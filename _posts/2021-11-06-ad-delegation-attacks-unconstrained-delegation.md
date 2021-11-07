---
layout: post
title: 'AD Delegation Attacks: Unconstrained Delegation'
tags: [Active Directory, Red Teaming, Delegation, Unconstrained Delegation]
---

I spent past few months working on [Cybernetics](https://app.hackthebox.com/prolabs/cybernetics) lab from HTB. Awesome labs, thoroughly enjoyed it, tons of things to learn. One such thing was Delegation Attacks in Active Directory. I found the concept interesting. It is not a vulnerability that we are exploiting, or something that is only applicable to older versions of Windows. It is a feature of Active Directory that we are trying to abuse. And I like these kind of attacks since there's no inherent "patch" for these. So first let's look at what actually Delegation is. If you are familiar with the concept of Delegation feel free to skip to 'Unconstrained Delegation' section.

## Delegation

To understand Delegation better, let's first discuss why we need Delegation.

### Double Hop Problem

Let us assume a scenario where:

- We are in a domain COSMOS.LAB
- An internal web app `http://quark.cosmos.lab` can be used to access files.
- These files are stored locally in `C:\quarkshare\`. Only user `COSMOS\Einstein` has access to the folder.
- IIS is running with `NetworkService` account.

In this scenario, we would face an error because the `NetworkService` account would not have access to `C:\quarkshare\`. In order for this web application to work, we would need to enable something called **Client Impersonation**. Explaining Client Impersonation is not the primary goal of this blog, but very briefly, Client Impersonation means that the IIS service would 'impersonate' the privilege of user logging in. So, with Client Impersonation, IIS service running as `NetworkService` would be able to access `C:\quarkshare\` with the privileges of client.

![Client Impersonation](/img/blog/2021/unconstrained-delegation/1.png){: .center-block :}

This all sounds well and good but the problem arises when we are trying to access a remote resource. In our example, if instead of accessing `C:\quarkshare\`, we tried to access, say, `\\BOSON.COSMOS.LAB\share` then IIS would still throw access denied error even with Client Impersonation enabled. This is because Client Impersonation only works on a local system and won't carry over to remote resources. This problem is called Double Hop Problem. And Delegation was introduced to resolve this problem.

![Image](/img/blog/2021/unconstrained-delegation/2.png){: .center-block :}

### Introducing Delegations

It is like Client Impersonation, but meant to work over network. Our IIS server would essentially save the 'credential material' in server to use it to authenticate to other services on behalf of the client. What this 'credential material' is depends on the type of Delegation we are using. Which brings me to the types of Delegations:

1. Unconstrained Delegation
2. Constrained Delegation
    - Kerberos Only
    - Protocol Transition
3. Resource Based Constrained Delegation

There's an abuse scenario for each of these. In this blog, I'll only focus on Unconstrained Delegation.

## Unconstrained Delegation

A small refresher about Kerberos authentication- When a client authenticates to a service using Kerberos, it uses somethings called a service ticket as a means of authentication. Client receives this service ticket from the KDC. In order to authenticate with the KDC itself and request the service ticket, client requires a Ticket Granting Ticket (TGT).

So, if we want our IIS service to impersonate client, it would also need to request a service ticket to the remote file server on behalf of the client. For that, in theory, it would need TGT of the client. This is the fundamental idea behind Unconstrained Delegation.

In Unconstrained Delegation, client would send a copy of its TGT along with the service ticket to the service. The service can then use this copy of TGT to request service tickets further to other services, like a remote file server, on behalf of client.

![Image](/img/blog/2021/unconstrained-delegation/3.png){: .center-block :}

Setting Unconstrained Delegation for an account requires `SeEnableDelegation` privileges, which only administrative accounts like Domain Admins would have. If a low privileged user has this privilege then big big trouble! Also, setting Unconstrained Delegation on a account sets `TRUSTED_FOR_DELEGATION` UAC flag. With that being said, let's look at Unconstrained Delegation in bit more depth. You can skip to 'Abusing Unconstrained Delegation' section if you don't want to read the details.

### Traffic Analysis

![Image](/img/blog/2021/unconstrained-delegation/4.png){: .center-block :}

Here's a snapshot of how the traffic would look like with Unconstrained Delegation. Let's break it down step-by-step.

Step 1: Client requests TGT from KDC. Nothing out of ordinary.

Step 2: Client requests TGS (service ticket) to `HTTP/QUARK`. Upon receiving this request, KDC would notice that `QUARK` has `TRUSTED_FOR_DELEGATION` UAC flag set. Thus, KDC would reply back with the TGS with `ok-as-delegate` flag set.

Step 3: Client would receive the `HTTP/QUARK` TGS from KDC. It would notice that the `ok-as-delegate` flag is set, which indicates that the service it is trying to authenticate with has Unconstrained Delegation set and client needs to send a copy of its TGT along with TGS. Thus, client would send another TGS request here, this time requesting a copy of its TGT. KDC would receive this request, and reply with a service ticket containing a `Forwardable` TGT.

Step 4: Client would then send the TGS(`HTTP/QUARK`) + TGT(`Forwardable`) to the `QUARK` web service.

Step 5: Server would receive the request and while serving the response it would notice it needs to interact with a remote share on `BOSON` server. `QUARK` would then send a service request to KDC for SPN `CIFS/BOSON` along with TGT(`Forwardable`) it received from client. KDC would comply with the request and issue the TGS.

Step 6: `QUARK` receive the `CIFS/BOSON` TGS from KDC, accesses the file share and proceeds with the response.

### Abusing Unconstrained Delegation

The concept of sending TGTs to the server should already sound bit scary. What if the server gets compromised and the attacker is able to retrieve TGT from the memory? That's exactly what we would try to do here. And we would see what impact a stolen TGT can have.

First things first, when we have a shell on a domain-joined system, how would we know which accounts have Unconstrained Delegation enabled? We can do this with [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1):

```powershell
Get-DomainComputer -Unconstrained #Would also list DC, ignore that
Get-DomainUser -ldapfilter "(userAccountControl:1.2.840.113556.1.4.803:=524288)"
```

Now that we know which accounts are our target, we need to compromise them. Discussing techniques to compromise systems would be out of scope of this blog but things like [Kerberoasting](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/t1208-kerberoasting) or getting RCE in the hosted webapp or abusing known vulnerabilities in hosted services are some examples.

Once we have compromised the target system, what we can do is simply monitor the network traffic for incoming TGTs. We can start monitoring using a tool called Rubeus:

```batch
Rubeus monitor /interval:1
```

If users now interact with our target system, we would automatically get their TGTs. We can use various methods to force someone to interact with our target system. Few examples would be:

- Phishing
- `xp_dirtree` with SQL Injection
- Printer Bug

Exploiting Printer Bug when Print Spooler service is running on a DC can potentially give you complete access over DC, and enables you to perform attacks like DCSync.

<figure>
  <video controls="true" allowfullscreen="true" poster="/img/blog/2021/unconstrained-delegation/thumb1.png" width="100%">
    <source src="/img/blog/2021/unconstrained-delegation/video1.mp4" type="video/mp4">
  </video>
</figure>

## Further Reading

- [You do (not) Understand Kerberos Delegation - ATTL4S](https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf)
- [Not a Security Boundary: Breaking Forest Trusts](http://www.harmj0y.net/blog/redteaming/not-a-security-boundary-breaking-forest-trusts/)
- Tool to exploit Printer Bug: [SpoolSample](https://github.com/leechristensen/SpoolSample)
