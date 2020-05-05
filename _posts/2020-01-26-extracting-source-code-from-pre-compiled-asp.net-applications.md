---
layout: post
title: 'Extracting Source Code from Pre-Compiled ASP.Net applications'
tags: [Web Security, ASP.Net]
---

In a recent assignment, I found a Path Traversal vulnerability in an ASP.Net based web application. Naturally, the first thing I went after was the `web.config` file. Extracted the DB credentials from connection string, but the MSSQL port was not open. And did some more post-exploitation enumeration. What I also do with Path Traversal is try to read the source code for finding other vulnerabilities and things like checking if SQL queries are parameterized, the restrictions implemented on file uploads, etc.

I tried to read the source code this time too. I read the source for an `.aspx` file, cool. But when I tried to read it's `.aspx.cs` file, I got the 'file not found' error. What? With ASP.Net Webforms applications, there always is an `.aspx` file and a related `.aspx.cs` file. After bit of fiddling around, I figured out that there is a concept in ASP.Net called 'Pre-Compilation' being used here. Let me explain the whole concept and how I was able to extract the source code with a small example.

## What is Pre-Compilation?

Pre-Compilation is an ASP.Net feature in which a website, when being published, can get all of its logical code (the CS files) 'compiled' into a binary (DLL file). So a website with files like this:

<p align="center"><img src='/img/blog/2020/precompilation/1.png' /></p>

Will look like this after pre-compilation:

<p align="center"><img src='/img/blog/2020/precompilation/2.png' /></p>

Where did all the CS files go? Into the DLL files in the `bin` directory:

<p align="center"><img src='/img/blog/2020/precompilation/3.png' /></p>

You can read about Pre-Compilation in details from [Microsoft Docs](https://docs.microsoft.com/en-us/aspnet/web-forms/overview/older-versions-getting-started/deploying-web-site-projects/precompiling-your-website-cs).

## Using Path Traversal to read DLL files

So we know that the DLL files contains all the source code we want to read. Question is how would we know what is the name of the DLL file we want to read? In the ASPX file, we do specify the `codebehind` parameter that defines the name of the associated CS file. What would happen to this parameter in pre-compiled applications? Let's see.

Let us consider the following example where we have a Path Traversal vulnerability:

<p align="center"><img src='/img/blog/2020/precompilation/4.png' /></p>

Notice here that we don't have any `codebehind` parameter. But we can notice that the `inherits` parameter do mention the name of the DLL file. Once we have this name, we can download the DLL file too:

<p align="center"><img src='/img/blog/2020/precompilation/5.png' /></p>

## Reversing DLL to extract source code

The last step here is to extract the source code from this DLL file. To do that, we can use a .Net decompiler like [JetBrains dotPeek](https://www.jetbrains.com/decompiler/). Once we open the file in dotPeek, we can easily get the source code:

<p align="center"><img src='/img/blog/2020/precompilation/6.png' /></p>

