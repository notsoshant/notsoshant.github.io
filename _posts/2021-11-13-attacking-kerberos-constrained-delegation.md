---
layout: post
title: 'Attacking Kerberos: Constrained Delegation'
tags: [Active Directory, Red Teaming, Delegation, Constrained Delegation, Protocol Transition, Kerberos Only]
---

In the [last blog](/blog/attacking-kerberos-unconstrained-delegation/), we discussed Unconstrained Delegation in detail. We also saw how dangerous Unconstrained Delegation can get. Unconstrained Delegation was the very first implementation of Delegations, introduced back in Windows Server 2000. It wasn't an ideal solution and Microsoft soon came with a better alternate called Constrained Delegation in Server 2003.

## S4U Extension

Major problem with Unconstrained Delegation was that we had to send TGTs along with our TGS so that the service can request TGS to other services on our behalf. How about if the service would be able to request TGS to other services using the TGS it has received from the client? That would eliminate the need for sending TGTs. But, if we think about how Kerberos authentication works, we do need TGT to request a TGS from KDC. This is where an extension to Kerberos was introduced. This extension allowed KDC to issue a TGS using a valid TGS also. This extension is called **Service-for-User (S4U)**, and it makes Constrained Delegation possible. S4U uses two kind of requests- **S4U2Proxy** and **S4U2Self**. Let's cover both of these.

### S4U2Proxy

S4U2Proxy request is pretty much what I've mentioned above. It allows a service to send a valid TGS to KDC and request TGS to another service. If it sounds bit confusing as of now, don't worry, there's an example below in [Traffic Analysis](#traffic-analysis) section that would explain the whole flow.

### S4U2Self

So, as I said, S4U2Proxy send a valid TGS to request another TGS. But what if the user is not using Kerberos to authenticate? In that case, the service wouldn't get the TGS from the client. This is where S4U2Self would chip in. S4U2Self allows a service to request a TGS to itself. So, when a client authenticates to the service using, say, NTLM authentication, what the service will do is first send S4U2Self request to get TGS to itself. Then use this TGS with S4U2Proxy to request TGS to another service. Again, if it all sounds confusing the example in [Traffic Analysis](#traffic-analysis) section would help.

## Kinds of Constrained Delegation

From the discussion above, we can somewhat get an idea that there are two kind of Constrained Delegations possible- one where we have Kerberos Authentication and only need S4U2Proxy and one where there is NTLM Authentication and we have to use both S4U2Self and S4U2Proxy. The first kind is called **Kerberos Only** and the second kind is called **Protocol Transition**, pretty self-explanatory names. Let's cover both of these. Small point to note here would be that, like Unconstrained Delegation, Constrained Delegation also requires a user with `SeEnableDelegation` to set it up on any account.

## Protocol Transition

![Protocol Transition image](/img/blog/2021/constrained-delegation/1.png){: .center-block :}

The image here shows how the configuration would look like with Protocol Transition enabled. Here we can see that the names of the services `QUARK$` is allowed to delegate to is already listed, it would get stored in `msDS-AllowedToDelegateTo` attribute of `QUARK$`. Additionally, when setting Protocol Transition for an account, `TRUSTED_TO_AUTH_FOR_DELEGATION` UAC setting also gets set. This flag is an indication, as we will see in the example below, to the KDC that the account supports S4U2Self requests.

### Traffic Analysis

![Protocol Transition Wireshark](/img/blog/2021/constrained-delegation/2.png){: .center-block :}

![Protocol Transition flow](/img/blog/2021/constrained-delegation/3.png){: .center-block :}

**Step 1**: Client authenticates to `QUARK$` using NTLM.

**Step 2**: `QUARK$` sends S4U2Self request to KDC, requesting TGS to itself. Interesting note here is that only the name of client (`Einstein`) would be part of this request, so it is possible to send a S4U2Self request for any arbitrary user.

**Step 3**: KDC would notice `QUARK$` has `TRUSTED_TO_AUTH_FOR_DELEGATION` set and accept the S4U2Self request. It would issue a TGS which has `Forwardable` flag set. Another thing to note here is that without `TRUSTED_TO_AUTH_FOR_DELEGATION` flag, KDC would still issue TGS, but without `Forwardable` flag.

**Step 4**: `QUARK$` sends S4U2Proxy request with TGS it got from S4U2Self and asks for TGS to `CIFS/BOSON`.

**Step 5**: KDC, upon receiving TGS request from `QUARK$`, would verify if `QUARK$` is allowed to delegate to `CIFS/BOSON` or not (by checking `msDC-AllowedToDelegateTo` parameter). Since `QUARK$` is allowed to, KDC would return TGS to `CIFS/BOSON` in response.

**Step 6**: The TGS returned by S4U2Proxy would then be used to access the remote share on `BOSON$`. Yet another thing to note here is that the SPN (`CIFS/BOSON`) is written in plaintext in the TGS. Thus, it can be modified to authenticate to other services on `BOSON$`.

### Abusing Protocol Transition

While analyzing the traffic, we noticed two things:

1.  Client name in S4U2Self request can be arbitrary. KDC essentially trusts the name of client provided in S4U2Self.
2. The SPN value in TGS are plaintext and can be substituted easily.

To abuse Protocol Transition, we will be utilizing both of these observations. But, for now, let's see how do we actually find accounts that support Constrained Delegation. We can use [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1) for that:

```powershell
Get-DomainComputer -TrustedToAuth -Properties cn,msds-allowedtodelegateto
```

Now that we know which accounts we have to target, we need to know how to craft S4U2Self and S4U2Proxy requests with the parameters we want. [Rubeus](https://github.com/GhostPack/Rubeus) has `s4u` module for that. Here's the command:

```bat
Rubeus s4u /impersonateuser:Administrator /user:QUARK$ /rc4:<NTLM> /msdsspn:cifs/BOSON /altservice:http /ptt
```

{: style="margin-bottom: 0px" :}
where:

- `impersonateuser`: The client name we want to requests TGS for
- `user`: Machine account that has Constrained Delegation enabled (can be user account also)
- `rc4`: NTLM hash of that machine account
- `msdsspn`: SPN to which `user` is allowed to delegate to
- `altservice`: Alternate services for which we want the TGS

Basically, what we will do here is first send a S4U2Self request to KDC with username Administrator. Since the KDC trusts the username provided in S4U2Self requests, it will return a valid TGS of user Administrator. And since `QUARK$` has Protocol Transition Constrained Delegation enabled, the returned TGS would also have `Forwardable` flag. This TGS would then be used in S4U2Proxy request next. Again, the TGS is of user Administrator, so the ticket to `CIFS/BOSON` returned by KDC would also be of user Administrator. Now, recall that the SPN value in TGS are in plaintext. So, we would modify the `CIFS/BOSON` to, for instance, `HTTP/BOSON` that would allow us to use `Invoke-Command` or maybe to `LDAP/BOSON` if `BOSON$` was DC which would have allowed us to DCSync.

Here's a video that will demo this attack. In the video, we first check for accounts that have Constrained Delegation enabled. We find `QUARK$` is allowed to delegate to `CIFS/BOSON`. We then assume `SYSTEM` access on `QUARK$`. After that, we try to run a command on `BOSON` to verify that we already don't have any such rights over it. Then we extract NTLM hash of the `QUARK$` account. Finally, we run Rubeus with `s4u` command to request the TGS `CIFS/BOSON` for user `Administrator` and then forge the SPN to `HTTP/BOSON` in order to run commands using WinRM.

<figure>
  <video controls="true" allowfullscreen="true" poster="/img/blog/2021/constrained-delegation/thumb1.png" width="100%">
    <source src="/img/blog/2021/constrained-delegation/video1.mp4" type="video/mp4">
  </video>
</figure>

## Kerberos Only

Now that we have covered the Protocol Transition, it's time to cover a much simpler kind of Constrained Delegation- Kerberos Only. Configuration for Kerberos is pretty much the same as [Protocol Transition](#protocol-transition), we just select 'Use Kerberos only' radio button instead of 'Use any authentication protocol'.

Like Protocol Transition, the list of services `QUARK$` is allowed to delegate to will be saved in `msDS-AllowedToDelegateTo` attribute. But, unlike Protocol Transition, Kerberos Only won't set the `TRUSTED_TO_AUTH_FOR_DELEGATION` UAC flag.

I won't deep dive into the traffic analysis for Kerberos Only here. The [traffic analysis of protocol transition](#traffic-analysis) and the diagram below should be sufficient enough to explain what would happen.

![Kerberos Only Wireshark](/img/blog/2021/constrained-delegation/4.png){: .center-block :}

![Kerberos Only flow](/img/blog/2021/constrained-delegation/5.png){: .center-block :}

### Abusing Kerberos Only

{: .box-warning :}
**Pre-requisites:** To understand this attack, you first need to understand the attacks for both [Protocol Transition](#abusing-protocol-transition) and [Resource Based Constrained Delegation](/blog/attacking-kerberos-resource-based-constrained-delegation/).

Kerberos Only is the most secure form of Delegation we have. To abuse this, we would actually be using an indirect method that exploits Resource Based Constrained Delegation first.

What do you need? You need `SYSTEM` access on a machine that has Kerberos Only Constrained Delegation set on the target machine. In our example below, we have `SYSTEM` on `QUARK$` and it has delegation set for `CIFS/BOSON` service.

How the attack path would look?

1. We will first add a new machine in domain. `STRANGE$` in our example.
2. We will add `STRANGE$` in `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute of `QUARK$`. *(If you didn't get what we did here, you probably need to go through [RBCD](/blog/attacking-kerberos-resource-based-constrained-delegation/) first)*{: .small :}
3. Abuse RBCD using `STRANGE$` on `QUARK$`. This will give us a forwardable TGS for `QUARK$`.
4. Use that TGS as proof of authentication in S4U2Proxy for `BOSON$`.

If it all sounded pretty complex then you're not alone, it really is. I'll try to explain as best as I could.

So, Kerberos Only has one major requirement- a valid forwardable TGS to be used in S4U2Proxy requests. If we recall the abuse case of [Protocol Transition](#abusing-protocol-transition), we were able to send the S4U2Self request to get a valid TGS but here in case of Kerberos Only, that TGS won't have `Forwardable` flag set. Question is when do we get a forwardable TGS? When a legit user requests TGS from KDC using its TGT. And in case of S4U2Proxy requests too.

Looking back at our example, `QUARK$` is the machine we have `SYSTEM` over and that has delegation set to `CIFS/BOSON`. As attacker, what we want to do is to run `Invoke-Command` over `BOSON$` as Administrator. Kerberos Only says that for S4U2Proxy request to `CIFS/BOSON`, we would need a valid TGS for `XYZ/QUARK` (`XYZ` meaning any service on `QUARK$`) of user Administrator.

Now, we can get a forwardable TGS for `XYZ/QUARK` if another resource is trying to either authenticate to or delegate to `XYZ/QUARK`. This is where we will use our RBCD trick. What we will do is create a machine `STRANGE$`. Then, set `QUARK$` to allow RBCD from `STRANGE$`. We will then use `STRANGE$` to delegate as Administrator into `QUARK$`. That way, we will get a valid forwardable TGS of Administrator (result of S4U2Proxy of RBCD). We will then use this TGS in the S4U2Proxy request to receive `CIFS/BOSON` TGS. This diagram will explain it better:

![Kerberos Only diagram](/img/blog/2021/constrained-delegation/6.png){: .center-block :}

Now comes our demo. This demo pretty much does the exact same thing we described above. First we will add the machine `STRANGE$`. Then we will set RBCD on `QUARK$` and verify if the RBCD is set correctly. Then we will RBCD from `STRANGE$` to `QUARK$` to get a valid TGS. Then use that TGS to perform a S4U2Proxy request to `BOSON$`.

<figure>
  <video controls="true" allowfullscreen="true" poster="/img/blog/2021/constrained-delegation/thumb2.png" width="100%">
    <source src="/img/blog/2021/constrained-delegation/video2.mp4" type="video/mp4">
  </video>
</figure>

Here's the script I've used in the demo:

<script src="https://gist.github.com/notsoshant/7d10544bb8b38b2f2169de4d5d872d4a.js"></script>

## Further Reading

- [You do (not) Understand Kerberos Delegation - ATTL4S](https://attl4s.github.io/assets/pdf/You_do_(not)_Understand_Kerberos_Delegation.pdf)
- [S4U2Pwnage - harmj0y](https://www.harmj0y.net/blog/activedirectory/s4u2pwnage/)
- [Another Word on Delegation - harmj0y](https://www.harmj0y.net/blog/redteaming/another-word-on-delegation/)
