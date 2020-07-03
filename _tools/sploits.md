---
layout: page
title: Sploits
description: Collection of exploits I've written
github: https://github.com/notsoshant/sploits
ext-js: https://buttons.github.io/buttons.js
---

[Follow @notsoshant](https://github.com/notsoshant){:class="github-button" data-size="large" aria-label="Follow @notsoshant"}
[notsoshant/sploits]({{ page.github }}){:class="github-button" data-icon="octicon-repo-forked" data-size="large" aria-label="View notsoshant/sploits on GitHub"}

This page is just a collection of exploits I have written while practicing Windows Exploitation.

### QuickHeal Buffer Overflow (CVE-2017-5005) Exploit

> [{{ page.github }}/tree/master/quickheal-cve-2017-5005]({{ page.github }}/tree/master/quickheal-cve-2017-5005){:target="_blank"}

This script can be used to generate malicious Mach-O file which can exploit QuickHeal and execute arbitrary shellcode.

Full writeup is available here: [Analysis of CVE-2017-5005: QuickHeal Buffer Overflow]({{ site.url }}/blog/analysis-of-CVE-2017-5005-quickheal-buffer-overflow/){:target="_blank"}

---

### MS07-017 Exploit

> [{{ page.github }}/tree/master/ms07-017]({{ page.github }}/tree/master/ms07-017){:target="_blank"}

Exploit for the Windows Animated Cursor Remote Code Execution Vulnerability (CVE-2007-0038). This involved bypassing the weak ASLR implementation of Windows Vista.

Full writeup is available here: [Windows Exploitation: ASLR Bypass (MS07–017)]({{ site.url }}/blog/windows-exploitation-aslr-bypass-ms07-017/){:target="_blank"}

---

### QuickZip Exploit

> [{{ page.github }}/tree/master/quickzip-4.60]({{ page.github }}/tree/master/quickzip-4.60){:target="_blank"}

My version of the QuickZip exploit discussed in this [Offensive Security article](https://www.offensive-security.com/vulndev/quickzip-stack-bof-0day-a-box-of-chocolates/){:target="_blank"}.

Full writeup is available here: [Windows Exploitation: Dealing with bad characters — QuickZip exploit]({{ site.url }}/blog/windows-exploitation-dealing-with-bad-characters-quickzip-exploit/){:target="_blank"}

---

### PMSoftware Simple Web Server 2.2-rc2 Exploit

> [{{ page.github }}/tree/master/simple-web-server-2.2]({{ page.github }}/tree/master/simple-web-server-2.2){:target="_blank"}

Exploit for PMSoftware Simple Web Server 2.2-rc2 I created while learning Egghunting technique.

Full writeup is available here: [Windows Exploitation: Egg hunting]({{ site.url }}/blog/windows-exploitation-egg-hunting/){:target="_blank"}

---

### Practice Vulnserver Exploits

> [{{ page.github }}/tree/master/vulnserver]({{ page.github }}/tree/master/vulnserver){:target="_blank"}

Set of exploits for various Vulnserver commands.
