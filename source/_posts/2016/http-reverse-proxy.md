---
title: All your websites are belong to us (with a reverse proxy)
lang: en-GB
categories:
  - WebDev
tags:
  - apache
  - nginx
  - reverse proxy
  - letsencrypt
cover: /images/2016/10/organic-lea-outdoor.jpg
date: 2016-10-10 09:00:00
---

I have taken the habit to use a **single domain** as an umbrella for **multiple websites**. Which in short means we can benefit from different software stacks being hosted on different platforms.

As a bonus, we can use a [custom domain on GitHub Pages](https://help.github.com/articles/using-a-custom-domain-with-github-pages/) with our own SSL certificate.

<!--more-->

# tl;dr

[Apache httpd](https://httpd.apache.org/) `ProxyPass` and `ProxyPassReverse` are our best friend to mount an external URL (and its descendants) onto a folder of our very own domain.

# Why using a reverse proxy?

We are in 2016 and mentioning [reverse proxy](https://en.wikipedia.org/wiki/Reverse_proxy) in a conversation sounds odd and pretty much dated from a different century. But hey, who cares?

I use reverse proxies for various reasons:

1. **to isolate components**
  So instead of putting everything in a larger and larger monolithic website, we can manage them as **different git repositories** and have a **different build process** as well (like `/photography` and `/talks` on this website)
1. **to manage different stacks**
  In the case of a conference with a yearly edition or so, we can iterate over the software stack and change accordingly to our needs **without having to upgrade legacy editions** nor to keep continuing them because we feel obliged to.
1. **to provide a transparent experience to our users**
  We can host content in different places and still provide a coherent experience to a user without having them to feel the spread of our infrastructure.
1. **to upgrade individual components**
  One by one rather than the entire stacks. Which makes the life easier in term of Q&A scope. Obviously, we fall in the [microservices trap](http://martinfowler.com/articles/microservices.html) so the more individual projects we have, the more scattered our attention and efforts can be.
1. **to run Docker containers or web applications**
  We can prevent to expose them directly on the port 80 or 443 – although this is not the point of this article as we are rather focused on *proxying external content*.

It is a good way to hide complex and purposeful components under a same and apparently unique domain. This is for example how websites like the [BBC](https://www.bbc.co.uk/) **feel like one** website whereas they are in reality composed of dozens and **dozens of different websites** developed by *independent* teams.

![](/images/2016/10/farmers-market.jpg)

# How does it work?

An HTTP request directed to our hosting provider will usually look like the following examples:

```
# Root
example.com → VirtualHost → /var/www/example.com

# Folder
example.com/cheese → VirtualHost → /var/www/example.com/cheese

# Specific file
example.com/doc/index.html → VirtualHost → /var/www/example.com/doc/index.html
```

By default we assume the folders `/cheese` and `/doc` are contained in the same directory as the root of the website.

Let's say we actually have decided to opt in for a _whitelabeled content provider_ for a part of the website and moved another part of it to a _static website_ hosted on [GitHub Pages](https://pages.github.com/). The above example would evolve into:

```
# Root
example.com → VirtualHost → /var/www/example.com

# Folder
example.com/cheese → VirtualHost + ProxyPass → http://cheese.com/whitelabel/example.com

# File
example.com/doc/index.html → VirtualHost + ProxyPass → http://example.github.io/site-doc/index.html
```

It should be clear enough so let's dive a bit more in how to achieve this.

# Configuring ProxyPass

The configuration of a reverse mainly relies on a declaration of Apache `ProxyPass` for each path (and its descendants) we would like to host elsewhere:

```
<VirtualHost *>
  ServerName example.com
  DocumentRoot /var/www/example.com

  <IfModule mod_proxy.c>
    ProxyPreserveHost Off

    ProxyPass /cheese http://cheese.com/whitelabel/example.com
    ProxyPassReverse /cheese http://cheese.com/whitelabel/example.com

    ProxyPass /doc http://example.github.io/site-doc
    ProxyPassReverse /doc http://example.github.io/site-doc
  </IfModule>
</VirtualHost>
```

And that's pretty much it!

![](/images/2016/10/cucumbers.jpg)

# Configuring Proxy directives

I found [Apache `ProxyPass` documentation](https://httpd.apache.org/docs/current/mod/mod_proxy.html#proxypass) to be quite clear actually (or maybe I spent too much time reading it). We can manage to exclude folders from the proxying or match only specific patterns with `ProxyPassMatch`. I guess all we need is a use case before starting to use them 😊.

## `ProxyPreserveHost`

This setting has an influence on how our VirtualHost proxy server will advertise the `Host` HTTP header to the client.

With `ProxyPreserveHost On`:

```bash
curl --head http://example.com/cheese
HTTP/1.1 200 Found
Host: cheese.com
```

With `ProxyPreserveHost Off`:

```bash
curl --head http://example.com/cheese
HTTP/1.1 200 Found
Host: example.com
```

So in general we will want to have it set to `Off`, especially in the case of web browsers and relative link computation.

## `ProxyPassReverse`

This *reverse* directive indicates Apache how to treat *location* headers emitted by the backend of the proxy.

In other words, if the backend emits some headers like `Location` and  `Content-Location`, our proxy will *rewrite* them to match our VirtualHost.

Without `ProxyPassReverse`:

```bash
curl --head http://example.com/cheese/old-uri
HTTP/1.1 301 Redirect
Location: http://cheese.com/whitelabel/example.com/new-uri
```

With `ProxyPassReverse`:

```bash
curl --head http://example.com/cheese
HTTP/1.1 301 Redirect
Location: http://example.com/cheese/new-uri
```

## `ProxyPassReverseCookieDomain`

This is exactly the same principle as `ProxyPassReverse` but to rewrite the hostnames contained in any `Cookie` header emitted by the backend.

## `SSLProxyEngine`

This one will enable the proxy module to deal with signed requests. We could definitely have an HTTP to HTTPS or, better, HTTPS to HTTP – to secure insecure parts of our website. Or to secure them… with a different SSL certificate.

And that's precisely one advantage to use a reverse proxy in front of GitHub Pages to use our custom domain and our own certificate.

![](/images/2016/10/autumn-squash.jpg)

# Reverse Proxy over HTTPS

GitHub serves every [GitHub Pages websites over HTTPS](https://help.github.com/articles/securing-your-github-pages-site-with-https/) if they have been created after *June 15<sup>th</sup> 2016*. So we will have to make sure both our server can talk over SSL with GitHub by enabling `mod_ssl`.

```
# ∨∨∨∨∨∨ We add this line
LoadModule ssl_module /wherever/is/apache2/modules/mod_ssl.so
# ∧∧∧∧∧∧

<VirtualHost *>
  ServerName example.com

  <IfModule mod_proxy.c>
    ProxyPreserveHost Off

    # ∨∨∨∨∨∨ and this one too.
    SSLProxyEngine On
    # ∧∧∧∧∧∧

    # ∨∨∨∨∨∨ Look, the backend URL now uses the https scheme!
    ProxyPass /doc https://example.github.io/site-doc
    ProxyPassReverse /doc https://example.github.io/site-doc
  </IfModule>
</VirtualHost>
```

As an alternative, we can also run the following to globally enable `mod_ssl` :

```bash
$ a2enmod mod_ssl
```

By doing so, we do not need to write the `LoadModule` line.

If we cannot enable `mod_ssl`, well we are screwed so best is to raise a support ticket to our hosting service provider if there is a way to enable it.

![](/images/2016/10/horsetail-tea.jpg)

# Conclusion

Is the *reverse proxy* technique limited to [Apache httpd](https://httpd.apache.org/)? Of course not:

- **Nginx** has [proxy_pass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) and a [good reverse proxy tutorial](https://www.nginx.com/resources/admin-guide/reverse-proxy/);
- **Varnish** is more *powerful* but [slightly harder to dive into its documentation](https://varnish-cache.org/docs/5.0/);
- **Node.js** [express-http-proxy](https://www.npmjs.com/package/express-http-proxy) package will help mount proxy routes in our [Express application](https://expressjs.com/).
  I personally use it to _proxy authentication at the app level_ and query internal APIs.
