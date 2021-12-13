---
layout: post
title: 'Attacking Kerberos: Resource Based Constrained Delegation'
tags: [Active Directory, Red Teaming, Delegation, Resource Based Constrained Delegation]
---

Now that we are done with [Unconstrained](/blog/attacking-kerberos-unconstrained-delegation/) and [Constrained](/blog/attacking-kerberos-constrained-delegation/) Delegations, it is time for the finale. In this blog we'll discuss Resource Based Constrained Delegation (RBCD).

## How it works?

Major restriction with Unconstrained or Constrained Delegation is that we require a user with `SeEnableDelegation` privileges to set it up. This privilege is usually restricted to (and it should be) administrative accounts like Domain Admins. This makes sense since delegation is such a sensitive configuration. But it isn't very convenient to ask Domain Admins to configure it every time someone, say a web developer, needs it. Classic example of security and usability being two opposite ends of spectrum.

So to ease some of this pain, Microsoft came up with a different kind of delegation that any user can set up. Catch here is that this delegation is set on the resource that is on the receiving end of delegation, not on the resource that wants to delegate to other resources.

{: .text-right .text-muted .small :}
![Delegation vs RBCD comparison diagram](https://shenaniganslabs.io/images/TrustedToAuthForDelegationWho/Diagrams/DelegationTypes.png){: .center-block :}
Image Credits: [shenaniganslabs.io](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)

Any user that has `Write` access on a resource can set that resource with RBCD, can tell that resource "dude, you can receive delegated credentials from resource X, Y and Z". Say we want to set up delegation from machine `QUARK$` to `BOSON$`. Instead of setting up anything on `QUARK$` like we would with Constrained Delegation, we would set it in `BOSON$`. The `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute stores the value of resources that are allowed to delegate to that machine.

![RBCD](/img/blog/2021/rbcd/1.png){: .center-block :}

## Traffic Analysis

![RBCD Wireshark](/img/blog/2021/rbcd/2.png){: .center-block :}

![RBCD flow](/img/blog/2021/rbcd/3.png){: .center-block :}

**Step 1**: Client send an NTLM-authenticated HTTP request to `QUARK$`.

**Step 2**: `QUARK$` sends S4U2Self request to KDC. But, since `TRUSTED_TO_AUTH_FOR_DELEGATION` is not set on `QUARK$`, KDC would return the TGS without the `Forwardable` flag.

**Step 3**: `QUARK$` would send S4U2Proxy request with non-forwardable TGS anyway. KDC would notice the TGS doesn’t have forwardable flag. So it will check the `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute for `BOSON$` and find `QUARK$`. KDC would thus accept the request and issue TGS to `CIFS/BOSON`.

**Step 4**: The SMB session would be established and the execution continues

## Abusing Resource Based Constrained Delegation

To abuse RBCD, there are primarily two pre-requisites:

1. An account (`COSMOS\Hawking` in our example) with `Write` permissions over `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of computer object you want to target (`BOSON$` here).
2. Credentials of account which has a SPN (`QUARK$` in our example). If we don’t own such account, we can create a Machine Account with [`Powermad.ps1`](https://github.com/Kevin-Robertson/Powermad/blob/master/Powermad.ps1).

Here's how attack would proceed:

1. First, we will verify if user `Hawking` has `Write` permissions over `BOSON$` or not.
2. Then we will add `QUARK$` in `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of `BOSON$`.
3. We will then perform s4u using [Rubeus](https://github.com/GhostPack/Rubeus) to get TGS to `BOSON$` and Pwn!

The video below demostrates the attack. We first verify using Bloodhound output that user `Hawking` has `Write` access over `BOSON$`. We can also do this using PowerShell, the commands to do so are shown in the video. Then we setup RBCD from `QUARK$` to `BOSON$`, and verify that. Then finally we run the Rubeus command that fetches the `CIFS/BOSON` TGS, and replaces it with `HTTP/BOSON` so that we can run `Invoke-Command` over `BOSON$`. 

<figure>
  <video controls="true" allowfullscreen="true" poster="/img/blog/2021/rbcd/thumb1.png" width="100%">
    <source src="/img/blog/2021/rbcd/video1.mp4" type="video/mp4">
  </video>
</figure>

Here's the script I've used in the demo:

<script src="https://gist.github.com/notsoshant/9a23fb4f5f87b01a3bd9369c3c938cfe.js"></script>

## Further Reading

- [Wagging the Dog: Abusing Resource-Based Constrained Delegation to Attack Active Directory - Elad Shamir](https://shenaniganslabs.io/2019/01/28/Wagging-the-Dog.html)
- [You do (not) Understand Kerberos Delegation - ATTL4S](https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf)
- [Powermad](https://github.com/Kevin-Robertson/Powermad)
