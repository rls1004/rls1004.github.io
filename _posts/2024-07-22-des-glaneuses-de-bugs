---
layout: post
title: "Des glaneuses de bugs"
image: /img/202407/pickup.png
tags: [browser]
---
* TOC
{:toc}

![](/img/202407/pickup.png)

# Simple Idea

1. There might be patches for vulnerabilities in code that hasn't been merged yet (=pull requests).
2. Patches that have been merged but not included in the release are still valid bugs in the latest version (=commits).

⚠️ It's important to check all branches because patches may be applied to branches other than the main branch first.

## Expectations

- Quick updates on new vulnerabilities
- Development of exploits for the latest version
- Finding new bug variants

## Implementation

Write a script to fetch pull requests and commit logs from all branches and identify security bugs, then display the results.

**Identifying Security Bugs**

- If accessing the bug ID on WebKit Bugzilla results in an "Access Denied" message, it is likely a security bug. (although this may not always be the case and could happen for other reasons)

# Progress...

Upon the first run, a pull request that had been open for 10 hours was discovered (pull request submitted on June 21).  

- Pull Request : [28961](https://github.com/WebKit/WebKit/pull/28961)
- Commit : [0b73d2178347c7e5f73ee11001d566b3d0c02eea](https://github.com/WebKit/WebKit/pull/28961/commits/0b73d2178347c7e5f73ee11001d566b3d0c02eea)
- Bugzilla : [Bug Access Denied](https://bugs.webkit.org/show_bug.cgi?id=274242)

**Summary**

Do not guess the MIME type based on the file extension; instead, adhere to the `X-Content-Type-Options: nosniff` header.

**Timeline**

(5.23) Issue reported  
  
(5.24, 5.26, 6.21) Pull request submitted  
(6.29) Code review & pull request  
  
(6.30) Merged  
  
Disappointed that it wasn't the type of vulnerability I was looking for 😞  
  
However, the same issue was found in browsers other than Safari.  
  
<video width="30%" controls autoplay>
    <source src="/img/202407/nosniff_ios.mp4" type="video/mp4">
</video>
<video width="30%" controls autoplay>
    <source src="/img/202407/nosniff_android.mp4" type="video/mp4">
</video>

Rendering the same page differently:  
**(iOS)** Safari, Chrome, Firefox, and Edge all execute the script.  
**(Android)** Chrome, Firefox, and Edge all treat it as text.  
**(Mac)** Handles it as text or file download.  

## The Bug

### Steps to reproduce

(1) Create test.html

```html
<script>alert("Hi");</script>
```

(2) Create .htaccess

```
<Files test.html>
Header always set X-Content-Type-Options "nosniff"
Header always set Content-Type ""
</Files>
```

(3) Run the web server & visit test.html

### Expected behavior

`<script>alert("Hi");</script>` should be treated as text or the test.html file should be downloaded, allowing the user to decide how to handle it.

### Actual behavior

The `alert("Hi")` script is executed.

### Description

Files with the `X-Content-Type-Options: nosniff` header should not have their type guessed.  
  
This header is an important security mechanism that instructs the browser to strictly follow the MIME type provided by the server.  
  
For more information on content sniffing, refer to [Content Sniffing on Wikipedia](https://en.wikipedia.org/wiki/Content_sniffing)  
  
According to the [MIME Sniffing Standard](https://mimesniff.spec.whatwg.org/#determining-the-computed-mime-type-of-a-resource), the script should not be executed in the above case.
