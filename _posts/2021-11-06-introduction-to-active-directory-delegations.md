---
layout: post
title: 'Introduction to Active Directory Delegations'
tags: [Active Directory, Red Teaming, Delegation, Kerberos, Double Hop Problem]
---

I spent past few months working on [Cybernetics](https://app.hackthebox.com/prolabs/cybernetics) lab from HTB. Awesome labs, thoroughly enjoyed it, tons of things to learn. One such concept was Delegation Attacks in Active Directory. I found it interesting and thought why not play with it in my own labs and write about it. It is not a vulnerability that we are exploiting, or something that is only applicable to older versions of Windows. It is a feature of Active Directory. So, there's no inherent "patch" for these, only extra set of precautions and we all know sysadmins love to follow those, right?

But before we jump onto Delegations, there are certain key concepts we need to know. This blog is dedicated only to those key concepts. We will be discussing how Kerberos works, Client Impersonation and Double Hop Problem. If you feel confident enough these concepts then feel free to skip to the [Delegation](#delegation) section below.

## Kerberos

Kerberos is like SSO before SSOs were cool- there's a single point of authentication that gives client a "token" and we use that "token" to authenticate with all other services. We'll not go super deep into how Kerberos authentication works. I'll quickly cover the very basic and key concepts we will need to understand Delegations. Like everyone who tries to explain Kerberos, I'll take help of a diagram here.

{: .text-right .text-muted .small :}
![Credits: adsecurity.org](https://adsecurity.org/wp-content/uploads/2015/04/Visio-KerberosComms-1024x511.png){: .center-block :}
Image Credits: [adsecurity.org](https://adsecurity.org)


In this image, we can see there are three parties involved:

1. **User**: Who wants to authenticate
2. **Key Distribution Center (KDC)**: Residing in Domain Controller, this is a single point of authentication that assigns the "token" to the client
3. **Application Server**: The resource the client wants to authenticate to

The "token" in case of Kerberos is a ticket. Two types of ticket:

1. **Ticket-Granting Ticket (TGT)**: It is your identity. Proves who you are. Kind of like your passport. Assigned to you by KDC when you log in with, say, your username and password.
2. **Ticket-Granting Service (TGS) or Service Ticket**: It is your ticket to one specific service. Kind of like airplane ticket. Again, assigned by the KDC when you request access to a service.

So we see the term "service" here a lot. A service is basically anything that a client would want to access- a website (IIS), a file share (SMB), remote desktop (RDP), etc. Correlate it to an airport or train station. There could be multiple websites, or IIS servers, in an environment. We need to uniquely identify each one of them. That's where we have something called **Service Principal Name (SPN)**. An SPN is of the format `service/host:port`. An example would be `HTTP/web01.domain.local:8080`.

There's lot more to Kerberos than just this, feel free to go ahead and read about it in depth. I'll move on to our next topic- Client Impersonation and Double Hop Problem.

## Double Hop Problem

Let us assume a scenario where:

- We are in a domain COSMOS.LAB.
- An internal web app `http://quark.cosmos.lab` can be used to access files.
- These files are stored locally in `C:\quarkshare\`. Only user `COSMOS\Einstein` has access to the folder.
- IIS is running with `NetworkService` account.

In this scenario, we would face an error because the `NetworkService` account would not have access to `C:\quarkshare\`. In order for this web application to work, we would need to enable something called **Client Impersonation**. Very briefly, Client Impersonation means that the IIS service would 'impersonate' the privilege of user logging in. So, with Client Impersonation, IIS service running as `NetworkService` would be able to access `C:\quarkshare\` with the privileges of `Einstein`.

![Client Impersonation](/img/blog/2021/intro-to-delegations/1.png){: .center-block :}

This all sounds well and good but the problem arises when we are trying to access a remote resource. In our example, if instead of accessing `C:\quarkshare\`, we tried to access, say, `\\BOSON.COSMOS.LAB\share` then IIS would throw access denied error even with Client Impersonation enabled.

![Image](/img/blog/2021/intro-to-delegations/2.png){: .center-block :}

This is because Client Impersonation only works on a local system and won't carry over to remote resources. This problem is called **Double Hop Problem**. And Delegation was introduced to resolve this problem.

## Delegation

It is like Client Impersonation, but meant to work over network. Our IIS server would essentially save the 'credential material' of the client and use it to authenticate to other services on behalf of the client. What this 'credential material' is depends on the type of Delegation we are using. Which brings me to the types of Delegations:

1. [Unconstrained Delegation](/blog/attacking-kerberos-unconstrained-delegation/)
2. [Constrained Delegation](/blog/attacking-kerberos-constrained-delegation/)
3. [Resource Based Constrained Delegation](/blog/attacking-kerberos-resource-based-constrained-delegation/)

I have covered all of these in their respective blogs, along with their abuses. Feel free to jump onto those.
