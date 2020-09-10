---
layout: post
title: 'A Not-So-Blind RCE with SQL Injection'
tags: [Web Security, ASP.Net, Remote Code Execution, SQL Injection]
---

Once again, I'm back with another story of an interesting finding. This time I'll be explaining an SQL injection instance, but this was bit different. The application here is based on ASP.Net, is using MSSQL, supports stacked queries and the DB user is also `sysadmin`. Everything looks nice and perfect to execute `xp_cmdshell`. The only problem I faced is that I cannot get the output of queries stacked after the first query. And on top of that, the application is behind a firewall that is not allowing any access to outside world. So, I can execute OS commands, yes, but cannot see it's output. This becomes a kind of blind RCE. But, as the title says, this is a not-so-blind RCE.

I have set up an identical test environment to demonstrate the exact problem we have in hand. Let's see how we will extract the output of `xp_cmdshell` here.

### Performing the UNION based SQL Injection

First, let's analyse the vulnerable request and try to perform a UNION based SQL Injection. Looking at the screenshot below, the `txtUserName` parameter is vulnerable to SQL Injection.

![Image](/img/blog/2020/not-so-blind-sqli/1.png){: .center-block :}

Naturally, we'll try to close the query with a comment. But we get another error when we do so:

![Image](/img/blog/2020/not-so-blind-sqli/2.png){: .center-block :}

The error says `basicsalary` is an invalid column name. By closely observing the part of the query disclosed in the error message with only a single quote as payload, we find that the query has `Payslips` table being joined. Maybe this `basicsalary` column is part of that table. In that case, we also need to join `Payslips` table in our payload before we comment rest of the query out. Let's try that:

![Image](/img/blog/2020/not-so-blind-sqli/3.png){: .center-block :}

The query executes and we have some data! We'll proceed further with typical `ORDER BY` and then `UNION` statements to gain a working UNION based SQL Injection:

![Image](/img/blog/2020/not-so-blind-sqli/4.png){: .center-block :}

### Checking the privileges of DB user

The next step here is to check if the DB user is a `sysadmin` or not, since only `sysadmin` can enable `xp_cmdshell` and execute OS level commands, which is our ultimate goal here.

Here I would like to introduce an [awesome SQL Injection Cheat Sheet](http://pentestmonkey.net/cheat-sheet/sql-injection/mssql-sql-injection-cheat-sheet) that I use. It is from pentestmonkey. Looking at this cheat sheet, we find that we can use `SELECT is_srvrolemember('sysadmin')` query to figure out if our current DB user is `sysadmin` or not. Let's try that:

![Image](/img/blog/2020/not-so-blind-sqli/5.png){: .center-block :}

Since we get a `1` in the response, we can confirm that the current DB user is indeed a `sysadmin`.

### Checking the support for stacked queries

Stacked queries means that we can run multiple queries in a single statement by separating them with a semicolon character, just like we can do in command line. So if we have a query like this:

```sql
SELECT * FROM user WHERE userid = "<injection-point>"
```

With stacked queries supported, we can run queries like this:

```sql
SELECT * FROM user WHERE userid = "-1" AND 1=2; WAITFOR DELAY '0:0:5'; -- "
```

Without stacked queries, we are only limited to `SELECT` statements and cannot run any `INSERT`, `UPDATE`, `DELETE` or something like `EXEC` queries. But with stacked queries, we can execute any kind of query we want. That's why peeps, without stacked queries, **don't** mark any integrity impact in the CVSS vector.

Anyways, moving ahead, let's check if the stacked queries are supported or not. We will stack a `waitfor delay` query after our query and see if it executes:

![Image](/img/blog/2020/not-so-blind-sqli/6.png){: .center-block :}

A delay of 5 seconds here confirm that the stacked queries are supported. Now let's see if we are able to get the output of stacked queries too. To do so, we'll stack our `SELECT 1,2,3,4` instead of using `UNION`:

![Image](/img/blog/2020/not-so-blind-sqli/7.png){: .center-block :}

Notice that we do not get any output for our `SELECT 1,2,3,4`, which means we'll not get any output for the queries we will stack after the initial query.

### Executing xp_cmdshell

Now let's enable `xp_cmdshell` and confirm if we are at least able to execute OS command. To do that, refer back to the cheat sheet. Following SQL commands needs to executed:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
```

Let's do that:

![Image](/img/blog/2020/not-so-blind-sqli/8.png){: .center-block :}

Awesome! We have enabled `xp_cmdshell`. Now let's test that:

![Image](/img/blog/2020/not-so-blind-sqli/9.png){: .center-block :}

A delay of 3 seconds (for a default of 4 pings) show that we indeed have command execution! At this moment, what we can do is try to connect back to our VPS server and gain a working shell. If the application is behind a firewall configured to block outgoing connections, but is still allowing DNS queries, you can use a cool DNS exfiltration method described [here](https://www.redsiege.com/blog/2018/11/sqli-data-exfiltration-via-dns/). But the firewall in our case is not allowing any outside interaction. So far, we are stuck with a blind RCE.

### Escalating blind RCE to not-so-blind RCE

What we can do maybe is run a command and redirect its output to a file, a file which is inside the webroot and we can access that file from the website itself. For instance, if our website is hosted in `C:\inetpub\wwwroot`, we will use `xp_cmdshell` to execute a command like `whoami > C:\inetpub\wwwroot\opt.txt` and then browse `http:\\site.com\opt.txt` to get the output of `whoami` command.

Well...

![Image](https://media.giphy.com/media/26uf2JHNV0Tq3ugkE/giphy.gif){: .center-block :}

We do not know the physical path of the website in this scenario. So we have to somehow read the file in some other way. Looking back at our good old cheat sheet, we do find a way! We can execute following queries to read a file:

```sql
CREATE TABLE mydata (line varchar(8000));
BULK INSERT mydata FROM 'c:\windows\win.ini';
SELECT line FROM mydata;
```

Let's use this method. First, let's execute our command and store it's output in a temporary file:

![Image](/img/blog/2020/not-so-blind-sqli/10.png){: .center-block :}

Now, we'll create a table and store the contents of our temporary file in that table:

![Image](/img/blog/2020/not-so-blind-sqli/11.png){: .center-block :}

Once we have the contents in our table, let's read it using the UNION query we have:

![Image](/img/blog/2020/not-so-blind-sqli/12.png){: .center-block :}

And boom! We can now read the output of commands we want to execute!
