---
date: 2019-01-02T14:20:02.374Z
title: Improving the cache performance of The Polyfill Service even more
category: Polyfill.io
---
From December 11th 2018 to December 17th 2018, [polyfill.io](https://polyfill.io) served 1,250,322 requests, 99.98% of which were served from Fastly's cache. To achieve such a high cache-hit ratio, we employed some novel solutions using <abbr title="Varnish Configuration Language">VCL</abbr>. In this blog post I will explain how we went about achieving this result.

## What is polyfill.io and how does it work?

polyfill.io is a service which serves polyfills for features which are missing in the requesting user-agent. It works by reading the request url to figure out which features the website is wanting to polyfill and then reading the user-agent header to see what features are missing in from the user-agent and serving polyfills for the missing features that were included in the request url.

## Normalising the user-agent header

User-Agent headers vary a lot and new values are seen everyday, however most of the contents in a header is not of importance to polyfill.io, we only need to know the family and the major, minor and patch version of the user-agent. Versions 1 and 2 of polyfill.io had an endpoint which would take the user-agent as a query parameter and return a normalised user-agent header in the format \`user-agent-family/major-version.minor-version.patch-version.\`.

E.G:

```
http https://polyfill.io/v2/normalizeUa\?ua\=Mozilla/5.0%20\(Macintosh\;%20Intel%20Mac%20OS%20X%2010_12_6\)%20AppleWebKit/537.36%20\(KHTML,%20like%20Gecko\)%20Chrome/71.0.3578.98%20Safari/537.36 -p h | grep "Normalized-User-Agent"
```

```
Normalized-User-Agent: chrome/71.0.0
```

Having it as an endpoint on the backend server required custom <abbr title="Varnish Configuration Language">VCL</abbr> which rewrites requests to the polyfill bundle endpoints without a normalised user-agent header to instead go to the normalising endpoint and when it got a response, would issue a <abbr title="Varnish Configuration Language">VCL</abbr> restart for the origin request but with the addition of the normalised user-agent header.

Having restarts be a core part of the system makes the normal request to response flow become more complicated.

In version 3 of polyfill.io we decided to move the user-agent normalisation code from the backend and into the CDN via <abbr title="Varnish Configuration Language">VCL</abbr>. This bought back the simple request to response flow and meant that a single request to the CDN would make at most only one request to the backend. This alone wouldn't make a big difference to the cache-hit ratio, what did make a big difference was taking the normalised user agent and checking if it is a user-agent that polyfill.io supports. Polyfill.io supports 16 user-agents, if the user-agent is not one of those 16 we set the value to `other/0.0.0`. This meant that the url used for identifying a polyfill-bundle contained a user-agent family that was one we supported, or the value `other`, this is extremely useful for increasing the cache-hit ratio because if the user-agent is unsupported, it will be served the same bundle as any other unsupported user-agent so we might as well have that bundle stored under a url that would serve both requests.

The user agents that are supported are:

* Chrome
* Android
* BlackBerry
* Edge
* Edge Mobile
* Internet Explorer
* Internet Explorer Mobile
* Safari
* iOS Safari
* iOS Chrome
* Firefox
* Firefox Mobile
* Opera
* Opera Mobile
* Opera Mini
* Samsung Mobile

## Normalising the API.

TODO
