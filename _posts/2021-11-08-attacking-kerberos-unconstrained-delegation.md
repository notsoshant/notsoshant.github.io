---
layout: post
title: 'Attacking Kerberos: Unconstrained Delegation'
tags: [Active Directory, Red Teaming, Delegation, Unconstrained Delegation, Printer Bug]
---

So we are going to talk about Unconstrained Delegation in this blog. I have already covered a small introduction to Delegations and the Kerberos concepts you'd need to understand it in an [introduction blog](/blog/introduction-to-active-directory-delegations/). Please check it out first if you haven't. If you feel confident about your Kerberos basics then continue reading.

## How it works?

I will repeat the example I've used in the [introduction blog](/blog/introduction-to-active-directory-delegations/). Let us assume a scenario where:

- We are in a domain `COSMOS.LAB`.
- An internal web app `http://quark.cosmos.lab` can be used to access files.
- These files are stored remotely in `\\BOSON.COSMOS.LAB\share`. Only user `COSMOS\Einstein` has access to the share.
- IIS is running with `NetworkService` account.

In this scenario, we would face an access denied error because IIS is running as `NetworkService` and even if Client Impersonation is enabled, it would only work on a local system and won't carry over to remote resources. What we want is IIS to impersonate client while accessing remote services also, like CIFS in our example.

Now, a small refresher about Kerberos authentication- When a client authenticates to a service using Kerberos, it uses something called a service ticket as a means of authentication. Client receives this service ticket from the KDC. In order to authenticate with the KDC itself and request the service ticket, client requires a Ticket Granting Ticket (TGT).

Coming back to our example, if we want our IIS service to impersonate client while accessing the file share, it would also need to request a service ticket to the file share on behalf of the client. For that, in theory, it would need TGT of the client. This is the fundamental idea behind Unconstrained Delegation.

![Unconstrained Delegation demo diagram](/img/blog/2021/unconstrained-delegation/1.png){: .center-block :}

In Unconstrained Delegation, when a client would connect with IIS, it would send a copy of its TGT along with the service ticket to IIS. IIS can then use this copy of TGT to request service tickets further to other services, like CIFS, on behalf of client.

![Image](/img/blog/2021/unconstrained-delegation/2.png){: .center-block :}

Setting Unconstrained Delegation for an account requires `SeEnableDelegation` privileges, which only administrative accounts like Domain Admins would have. Unconstrained Delegation flicks the `TRUSTED_FOR_DELEGATION` UAC flag of the account, we can check if any account in the domain has this flag set to quickly find all accounts with Unconstrained Delegation enabled. With that being said, let's look at Unconstrained Delegation in bit more depth. You can skip to [Abusing Unconstrained Delegation](#abusing-unconstrained-delegation) section if you don't want to read the details.

## Traffic Analysis

![Image](/img/blog/2021/unconstrained-delegation/3.png){: .center-block :}

Here's a snapshot of how the traffic would look like with Unconstrained Delegation. Let's break it down step-by-step.

**Step 1**: Client requests TGT from KDC. Nothing out of ordinary.

**Step 2**: Client requests TGS (service ticket) to `HTTP/QUARK`. Upon receiving this request, KDC would notice that `QUARK$` has `TRUSTED_FOR_DELEGATION` UAC flag set. Thus, KDC would reply back with the TGS with `ok-as-delegate` flag set.

**Step 3**: Client would receive the `HTTP/QUARK` TGS from KDC. It would notice that the `ok-as-delegate` flag is set, which indicates that the service it is trying to authenticate with has Unconstrained Delegation set and client needs to send a copy of its TGT along with TGS. Thus, client would send another TGS request here, this time requesting a copy of its TGT. KDC would receive this request, and reply with a service ticket containing a `Forwardable` TGT.

**Step 4**: Client would then send the TGS(`HTTP/QUARK`) + TGT(`Forwardable`) to the `QUARK$` web service.

**Step 5**: Server would receive the request and while serving the response it would notice it needs to interact with a remote share on `BOSON$` server. `QUARK$` would then send a service request to KDC for SPN `CIFS/BOSON` along with TGT(`Forwardable`) it received from client. KDC would comply with the request and issue the TGS.

**Step 6**: `QUARK$` receive the `CIFS/BOSON` TGS from KDC, accesses the file share and proceeds with the response.

## Abusing Unconstrained Delegation

The concept of sending TGTs to the server should already sound bit scary. What if the server gets compromised and the attacker is able to retrieve TGT from the memory? That's exactly what we would try to do here. And we would see what impact a stolen TGT can have.

But before all of that, when we have a shell on a domain-joined system, how would we know which accounts have Unconstrained Delegation enabled? We can do this with [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1):

```powershell
Get-DomainComputer -Unconstrained #Would also list DC, ignore that
Get-DomainUser -ldapfilter "(userAccountControl:1.2.840.113556.1.4.803:=524288)"
```

Now that we know which accounts are our target, we need to compromise them. Discussing techniques to compromise systems would be out of scope of this blog but things like [Kerberoasting](https://www.ired.team/offensive-security-experiments/active-directory-kerberos-abuse/t1208-kerberoasting) or getting RCE in the hosted webapp or abusing known vulnerabilities in hosted services are some examples.

Once we have compromised the target system, what we can do is simply monitor the network traffic for incoming TGTs. We can start monitoring using a tool called [Rubeus](https://github.com/GhostPack/Rubeus):

```batch
Rubeus monitor /interval:1
```

If users now interact with our target system, we would automatically get their TGTs. We can use various methods to force someone to interact with our target system. Few examples would be:

- Phishing
- `xp_dirtree` with SQL Injection
- Printer Bug

### Printer Bug

Printer Bug probably deserves a blog of its own but I'll cover the very basics here. First things first, it is not related to Delegations at all, it's a completely different bug. We are combining Printer Bug and Unconstrained Delegation to escalate our impact.

Printer Bug is a bug with how Print Spooler service works. By exploiting Printer Bug, we can force any system that runs the Print Spooler service to interact with any other system. For example, if we have a shell in a client machine, and the DC is running Print Spooler, we can ask DC to connect to client or any other machine. We can combine this behavior with Unconstrained Delegation. We can force DC to connect with a system which has Unconstrained Delegation set. That way, when DC would connect to that system, it would authenticate first. Due to Unconstrained Delegation, it would send a copy of its TGT to that system. And that's what we want! TGT of DC! It can allow us to perform loads of attack, I'll use the example of DCSync below.

### Demo

Now comes the time for a short demo. Assumption here is that we have `SYSTEM` access on a machine `QUARK$` which is set with Unconstrained Delegation. In the video, we first verify if Unconstrained Delegation is indeed set. We run `Rubeus` in monitor mode to monitor incoming tickets. Then we use [SpoolSample](https://github.com/leechristensen/SpoolSample), a tool by [Lee Christensen](https://twitter.com/tifkin_) that exploits the Printer Bug, to force DC to connect back to `QUARK$`. In the process, we capture DC's TGT in `Rubeus`. We then import the TGT in memory, we can import it with any domain joined account on any machine. And then we run DCSync.

<figure>
  <video controls="true" allowfullscreen="true" poster="/img/blog/2021/unconstrained-delegation/thumb1.png" width="100%">
    <source src="/img/blog/2021/unconstrained-delegation/video1.mp4" type="video/mp4">
  </video>
</figure>

## Further Reading

- [You do (not) Understand Kerberos Delegation - ATTL4S](https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf)
- [Hunting in Active Directory: Unconstrained Delegation & Forests Trusts - SpecterOps](https://posts.specterops.io/hunting-in-active-directory-unconstrained-delegation-forests-trusts-71f2b33688e1)
- [Not a Security Boundary: Breaking Forest Trusts - harmj0y](http://www.harmj0y.net/blog/redteaming/not-a-security-boundary-breaking-forest-trusts/)
- [The Printer Bug](https://www.slideshare.net/harmj0y/derbycon-the-unintended-risks-of-trusting-active-directory/41)
