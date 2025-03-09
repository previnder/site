---
title: Migrating a Progressive Web App (PWA) to a new domain without breaking existing installations
date: 2025-03-09
slug: /pwa-domain-change
---

Recently, I migrated [Discuit](https://discuit.org/), which is a [Progressive Web App (PWA)](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps), from `discuit.net` to `discuit.org`. Everything went smoothly and seamlessly, I’m glad to say, without it messing up the existing user experience in any way. In fact, had I not made an [announcement about it](https://discuit.org/Discuit/post/FZadJa9b), most users wouldn’t have even noticed the change.

Migrating a website to a new domain is usually straightforward, at least for simple websites: a basic HTTP 301 “Moved Permanently” response is often enough to redirect traffic from the old domain to the new. But for PWAs, this approach can break existing installations. This is because a blanket 301 redirect would change the [origin](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Origin) of both the [manifest file](https://developer.mozilla.org/en-US/docs/Web/Progressive_web_apps/Manifest) and the [service worker file](https://developer.mozilla.org/en-US/docs/Web/API/Service_Worker_API/Using_Service_Workers), both of which are required to be on the same origin they were first installed from.

This means that once a PWA is installed, its domain cannot be changed. However, we have something of a solution if we:

1. Keep the old domain working as it is for existing PWA users.
2. Redirect only new users to the new domain.

## Identifying requests coming from a PWA

Where it gets tricky is, how do we know which HTTP requests are coming from a PWA and which ones are not? Unfortunately, there’s no official, browser-supported, and fool-proof way of doing that.

### Using a custom HTTP header added by the service worker

Had I planned for precisely this kind of PWA domain migration from the beginning—that is, from the first release of the PWA—I could’ve added a definitive way of identifying requests coming from a PWA.

Since the service worker that a PWA includes is a proxy that sits between the application and the network, it can modify incoming and outgoing HTTP requests. And if I had written the service worker such that it modified all outgoing requests to include an HTTP header that says that the request is coming from a PWA, I would’ve had a straightforward method for identifying the PWAs.

However, since there is no reliable way of updating the service workers of all the previously installed PWAs at once, this method is useless, and I had to rely on other, less straightforward, methods.

### Using cookies

One of those methods was checking for the existence of cookies. Discuit sets a session cookie, named `SID`, when a user first visits the site, which has an expiration date a year from the time it’s set. Almost all users who have visited the site in recent months would send this cookie along with all the HTTP requests to the server, making it a fairly reliable but still not foolproof way of identifying most of the recent PWA users.

Of course, it will identify not just PWA users but also regular users who’ve visited the site recently as well; however, for my purposes this was not at all an issue.

### When cookies don’t exist

But what about someone who has not opened the PWA in more than a year since the session cookie was set? Or someone who’s cleared the cookies for some reason in the meantime? It would be better if the domain change didn’t break the PWA for those users as well. That could be made to work, firstly, by not redirecting both the service worker and manifest files regardless of any other factors[^2]; and secondly, by allowing web pages at the old domain to access resources at the new domain using [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS).

For the latter, the server needs to return the following HTTP headers for incoming requests to the new domain, `discuit.org`:

```shell
access-control-allow-credentials: true
access-control-allow-methods: GET, POST, HEAD, OPTIONS
access-control-allow-origin: https://discuit.net
```

### Navigation preload requests

Then there is the matter of [service worker navigation preloading](https://web.dev/blog/navigation-preload), a technique used in service workers to reduce initial page load time. We don’t want to redirect these requests either. Fortunately for us, all service worker navigation preload requests include an HTTP header named `Service-Worker-Navigation-Preload`.

### API requests

There is also no need, in any case, to redirect API requests. Since Discuit uses cookies for authentication, if the `SID` cookie is missing, redirecting these API requests that require authentication would result in an error.

Moreover, if the `SID` cookie is expired, this is one more condition that would allow the re-setting of that cookie. And on Discuit, all API requests have the pathname prefix of `/api/`, which makes it trivial to identify them.

## Final migration strategy

Now we have a fairly complete solution to our domain migration problem that doesn’t break any of the existing PWA installations. As a bonus, it also doesn’t require already logged in users to log back in on the new domain either; they could continue to use the old domain as it was.

In summary, the complete solution is:

1. HTTP 301 redirect all incoming requests for the old domain, `discuit.net`, to the new domain, `discuit.org`, except:
   1. If the HTTP request has a path that points to either the manifest file or the service worker file.
   2. If the HTTP request includes a `Service-Worker-Navigation-Preload` header.
   3. If the HTTP request includes a cookie named `SID`.
   4. If the HTTP request has a path prefix of `/api/`.
2. Enable cross-origin access to the `discuit.org` domain, for the `discuit.net` domain, using CORS headers.

To implement this, I wrote a reverse-proxy in Go that sits between the web-server and the Internet. It’s available [here on Github](https://github.com/discuitnet/rip).

## Future proofing

After all this is done, I updated the service worker to modify all outgoing HTTP requests to include an HTTP header, named `X-Service-Worker-Version`[^3], in any case, which might come in handy in the future, perhaps for a different use case.

[^1]: Depending on the situation, this might even be permanent or long lasting, such that even if the migration was rolled back, the app would still be broken.
[^2]: For the manifest file, this is necessary anyway, because requests to the manifest file are performed without credentials—so the `SID` cookie won’t be sent along with it—unless the `<link>` tag that points to the manifest includes the attribute `crossorigin="use-credentials"`.
[^3]: You have to make sure to include this header only in same-origin requests. Otherwise, CORS will block them.
