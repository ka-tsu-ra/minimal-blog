---
date: 2019-01-02T14:20:02.374Z
title: Improving the cache performance of The Polyfill Service even more
category: Polyfill.io
---
From December 11th 2018 to December 17th 2018, [polyfill.io](https://polyfill.io) served 1,250,322 requests, 99.98% of which were served from Fastly's cache. To achieve such a high cache-hit ratio, we employed some novel solutions using <abbr title="Varnish Configuration Language">VCL</abbr>. In this blog post I will explain how we went about achieving this result.

## What is polyfill.io and how does it work?

Polyfill.io is a service which serves polyfills for features which are missing in the requesting User-Agent. It works in three broad steps:
1. It reads the request url to figure out which features the website is wanting to polyfill
2. It reads the User-Agent header to see what features it is missing
3. It serves polyfills for the missing features that were included in the request url

## How the caching works

Polyfill.io uses [Varnish Cache](https://varnish-cache.org/intro/), specifically it uses [Fastly's Varnish Cache](https://www.fastly.com/blog/benefits-using-varnish). When a request is made to polyfill.io, the Varnish Cache server will handle the request, create a hash key and check if an object in its cache has the corresponding hash key. If it does, then polyfill.io responds with the cached object.

Varnish Cache is a programmable cache, which is great because it allows us to define how the hash key is created. We decided to use the <abbr title="Uniform Resource Locater">URL</abbr> path and query-parameters as the hash key because they are the interface to our <abbr title="Application Program Interface">API</abbr>. The rest of this post goes into how we made different request <abbr title="Uniform Resource Locater">URL</abbr>s end up being the same <abbr title="Uniform Resource Locater">URL</abbr> before Varnish Cache generates the hash key.

The code used to generate the hash key:

```
sub vcl_hash {
	...
	set req.hash += req.url;
	# We include return(hash) to stop the function falling through to the default VCL built into varnish, which for vcl_hash will add req.url and req.http.Host to the hash.
	return(hash);
}
```

<small>[You can also view the code on GitHub](https://github.com/Financial-Times/polyfill-service/blob/714623bdfff470b865c0e6f7746db5f6908f3acc/fastly/vcl/polyfill-service.vcl#L88-L98)</small>

## Normalising the query parameters

Polyfill.io users specify what features they need in their request's query parameters. For example this request:
```
polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded
```
is configuring the polyfill bundle to contain polyfills for [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch), [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver), and to call `polyfillsLoaded` once done.

Due to flexibility in the way urls can be formulated, many other string formulations would configure this exact same polyfill bundle.\
A non-exhaustive list:

1. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?features=fetch,IntersectionObserver&callback=polyfillsLoaded`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
5. `polyfill.io/v3/polyfill.js?features=fetch,IntersectionObserver&callback=polyfillsLoaded&zebra=striped`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`

We want all of these different URLs to point to the same response in the cache. To do this they need to use the same hash key. The way we achieve that is to convert them all to the same URL before the Varnish Cache creates the key.

Some of the URLs in the list have the exact same query parameters, only in different orders. We can re-order the query parameters with a function that Fastly provide called [`querystring.sort`](https://docs.fastly.com/vcl/functions/querystring-sort/). This function alone will turn URLs 1 and 2 into the same URL. It will do the same for URLs 3 and 4.

[You can view the code for this at fiddle.fastlydemo.net/fiddle/2dca4d1c](https://fiddle.fastlydemo.net/fiddle/2dca4d1c/embedded)

Listing the URLs again, now with the query parameters sorted:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver&zebra=striped`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`

There are still more differences that we can normalise:

The order of the comma-separated features in the features parameter isn't the same for URLs 2 and 3. We would need to sort those features by some manner in order to make them identical. Neither Varnish Cache nor Fastly offer a pre-built function to sort a string, VCL also does not have a way to loop through items either, which makes this a bit trickier to solve.

The way we solved this issue was by creating a function which takes a comma-separated string and turns it into a URL where each item in the comma-separated string is a lone-standing query parameter. Then we can use the same `querystring.sort` function that we used earlier, and finally, we turn the query parameters back into a comma-separated string.

[You can view the code for this at fiddle.fastlydemo.net/fiddle/a2950143](https://fiddle.fastlydemo.net/fiddle/a2950143/embedded)

Using this function will make URLs 1, 2, 3 and 4 identical.

Listing the URLs again, after this function has been used:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch&zebra=striped`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`

The 5th URL has configured `zebra` to `striped`, but `zebra` is not part of the API for configuring a polyfill bundle. Let's add a function to only keep query parameters in the URL which are actually part of the public API. We can achieve this with a function that Fastly provide called [`querystring.regfilter_except`](https://docs.fastly.com/vcl/functions/querystring-regfilter-except/). Using this function will make URLs 1, 2, 3, 4, and 5 become identical.

[You can view the code for this at fiddle.fastlydemo.net/fiddle/784c6fc6](https://fiddle.fastlydemo.net/fiddle/784c6fc6/embedded)

Listing the URLs again, after this function has been used:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`


The 6th URL has set `unknown` to `polyfill`, which just happens to be the same as the default value for `unknown`. If we add the default values into the URL for the parameters which have not been set, we can make all the URLs in the list become identical. Remember, we use the URL as the hash key within Fastly, so making these requests use the same URL internally will mean that they point to the same response in the cache, which is what we are trying to achieve.

[You can view the code for this at fiddle.fastlydemo.net/fiddle/8771bc22](https://fiddle.fastlydemo.net/fiddle/8771bc22/embedded)

With all these functions in place the end result is that all 6 of those URLs become identical, which means they will have the same hash key inside Varnish Cache and therefore all point to the same single cached response, increasing the cache-hit ratio and decreasing the amount of requests that need to go all the way back to servers for polyfill.io.

## Normalising the User-Agent header inside Varnish Cache

It made sense to explain the normalisation of the query paramaters first, but this is actually where we made the biggest step up in the cache-hit ratio. In the previous section I omitted the fact that one of the options in the API is to set the User-Agent in the URL via the `ua` query parameter. This is a very important feature with regard to caching because it means that we can have a different cached entry for each User-Agent making a request. However it also means that there will be a lot of cache entries and it will make the cache-hit ratio really low. The reason that would happen is because User-Agent values vary *a lot*. [whatismybrowser.com](https://developers.whatismybrowser.com/useragents/explore/) has collected 840,000 unique User-Agents and keeps finding new ones every day.

Luckily for polyfill.io we only care about the User-Agent family, major, and minor version. In polyfill.io v1 and v2 we had an API endpoint that would take a User-Agent and return a version of it which only had the family, major, and minor version. This worked very well but introduced some complications in the VCL. Since the API endpoint was implemented in the polyfill.io server it meant that a request without a `ua` query parameter would first need to go to this ua-specific endpoint to find out what its normalised User-Agent value was, and then go to its original destination to return a polyfill bundle.

In v3 we have implemented this API endpoint as a function in VCL, which has removed the complications around making two requests to the polyfill.io server. The way that we parse User-Agents now is by compiling the [uaparser.org](https://www.uaparser.org/) into VCL for the polyfill.io service and into JS for the [polyfill-library](https://github.com/Financial-Times/polyfill-library) npm package.

[You can view the code for this at fiddle.fastlydemo.net/fiddle/f857ab79](https://fiddle.fastlydemo.net/fiddle/f857ab79/embedded)

Normalising the User-Agent helps reduce the millions of different User-Agents down to the thousands of User-Agents that [uaparser.org](https://www.uaparser.org/) detects them as. We can still improve on this - currently polyfill.io supports 15 User-Agent families: Android, BlackBerry, Chrome, Edge, Edge Mobile, Firefox, Firefox Mobile, Internet Explorer, Internet Explorer Mobile, iOS Safari, iOS Chrome, Opera, Opera Mini, Opera Mobile and Samsung. If we detect that the User-Agent family is one we do not support, we can give it a generic unsupported name, such as `other` to ensure that all unsupported User-Agents generate the same internal URL and point to the same cached object.

[You can view the code for this at fiddle.fastlydemo.net/fiddle/6a4eb0a1](https://fiddle.fastlydemo.net/fiddle/6a4eb0a1/embedded)

As I said, this change is the one that brought about the biggest improvement in our cache-hit ratio. Instead of getting a different cache-entry for every browser family and version, we only get different cache-entries for browser family and versions we support. As of writing this blog post, that works out to be roughly 300 different cache entries (Chrome has roughly 70 releases, Edge has 6 etc).
