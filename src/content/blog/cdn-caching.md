---
title: "CDN Caching"
date: 2025-08-08
description: "An in-depth look at CDN Caching"
---

# Starting from scratch
When you enter `chatgpt.com` what happens next?
1. Browser Checks if You Typed a Search or a URL
- If what you typed looks like a URL (e.g., has .com), the browser treats it as a website address.
- If not, it sends it directly to your default search engine.

2. Browser Checks Local Cache (DNS Cache)
- Your computer keeps a small memory (cache) of recent domain name lookups. You can check this on `chrome://net-internals/#dns`
- If you’ve visited `chatgpt.com` recently, your OS/browser might already know its IP address (e.g., 172.64.155.209).
- If it finds it, it skips to Step 5.

DNS Servers


3. DNS Lookup (Finding ChatGPT's IP)
If your computer doesn’t know the IP address:
- Browser asks the OS → OS asks the DNS resolver (usually from your ISP or a custom one like Google DNS 8.8.8.8 or Cloudflare 1.1.1.1).
- Resolver asks the root DNS servers → finds .com name servers.

- .com name servers point to Google’s authoritative DNS (more) servers.
- Resolver asks Google’s DNS → gets back the IP for google.com.
- This IP is stored in the cache for next time.


# CDN Caching

## What is CDN Caching?

## How does CDN Caching work?

## How to implement CDN Caching?