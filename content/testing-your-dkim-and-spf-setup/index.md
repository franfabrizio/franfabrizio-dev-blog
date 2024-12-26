---
title: "Testing Your DKIM and SPF Email Setup"
date: 2024-11-06T13:56:10-06:00
draft: true
toc: true
---

Recently, I've been debugging some problems I'm having with one of my mail domains getting routed to folks' spam folders. One of the first things to do in such a case is to verify that your DKIM and SPF settings are correct. 

I came across two very helpful tools to help with that, and wanted to give them a bit of visibility here.

The first is www.mail-tester.com. 

The second is www.learndmark.com. 

DMARC Results

--- Connection parameters ---
Source IP address: 0.0.0.0
Hostname: example1.com
Sender: user@example2.com

--- SPF ---
RFC5321.MailFrom domain: example2.com
Auth Result: PASS
DMARC Alignment: PASS

--- DKIM ---
Domain: example2.com
Selector: fm3
Algorithm: rsa-sha256 (2048-bit)
Auth Result: PASS
DMARC Alignment: PASS

-- DKIM ---
Domain: example3.com
Selector: fm3
Algorithm: rsa-sha256 (2048-bit)
Auth Result: PASS
DMARC Alignment: example3.com != example2.com

--- DMARC ---
RFC5322.From domain: example2.com
Policy (p=): none
SPF: PASS
DKIM: PASS
DMARC Result: PASS

--- Final verdict ---
DMARC does not take any specific action regarding message delivery. Generally, this means that the message will be successfully delivered. However, it's important to note that other factors like spam filters can still reject or quarantine a message.

---------------------
Thanks for using learndmarc.com
This free service is brought to you by URIports.com - DMARC Monitoring Reinvented.
