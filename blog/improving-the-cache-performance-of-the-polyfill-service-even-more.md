---
date: 2019-01-02T14:20:02.374Z
title: Improving the cache performance of The Polyfill Service even more
category: Polyfill.io
---
From December 11th 2018 to December 17th 2018, [polyfill.io](https://polyfill.io) served 1,250,322 requests, 99.98% of which were served from Fastly's cache. To achieve such a high cache-hit ratio, we employed some novel solutions using <abbr title="Varnish Configuration Language">VCL</abbr>. In this blog post I will explain how we went about achieving this result.

## What is polyfill.io and how does it work?

Polyfill.io is a service which serves polyfills for features which are missing in the requesting user-agent. It works by reading the request url to figure out which features the website is wanting to polyfill and then reading the user-agent header to see what features are missing in from the user-agent and serving polyfills for the missing features that were included in the request url.

## How the caching works

Polyfill.io uses [Varnish Cache](https://varnish-cache.org/intro/), specifically it uses [Fastly's Varnish Cache](https://www.fastly.com/blog/benefits-using-varnish). When a request is made to polyfill.io, the Varnish Cache server will handle the request, create a "hash key" and check if an object in it's cache has the corresponding "hash key", if it does then it responds with the cached object.

Varnish Cache is a programmable cache, which is great because it allows us define how to create the "hash key". We decided to use the <abbr title="Uniform Resource Locater">URL</abbr> path and query-parameters as the hash key because they are the interface to our <abbr title="Application Program Interface">API</abbr>. The rest of this post goes into how we made different request <abbr title="Uniform Resource Locater">URL</abbr>s end up being the same <abbr title="Uniform Resource Locater">URL</abbr> before Varnish Cache generates the hash key.

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

The API for configuring a polyfill bundle is available via the query paramter. I.E. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded` is configuring the polyfill bundle to contain polyfills for [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch), [`IntersectionObserver`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserver), and to call `polyfillsLoaded` once done.

There are many ways to configure this exact same polyfill bundle.\
A non-exhaustive list:

1. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?features=fetch,IntersectionObserver&callback=polyfillsLoaded`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
5. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill&zebra=striped`

We want to be able to have all these different URLs use the same hash key, the way to do that is to get them all to have the same URL before the Varnish Cache creates the key.

If you look at the list of URLs you might notice that some of them have the exact same query parameters but in a different order. We can re-order the query parameters with a function that Fastly provide called [`querystring.sort`](https://docs.fastly.com/vcl/functions/querystring-sort/). Using just this function will make URLs 1 and 2 become the same as well as 3 and 4.

<script type="application/json+fiddle">
{
  "origins": [
    "https://polyfill.io"
  ],
  "vcl": {
    "recv": "# Store original url for logging purposes.\ndeclare local var.original-url STRING;\nset var.original-url = req.url;\n\nset req.url = querystring.sort(req.url);\n\nlog \"Original url: \" var.original-url;\nlog \"Updated  url: \" req.url;"
  },
  "reqUrl": "/v3/polyfill.js?features=IntersectionObserver%2Cfetch&callback=polyfillsLoaded",
  "reqMethod": "GET",
  "purgeFirst": false,
  "enableCluster": true,
  "enableShield": false
}
</script>

Listing the URLs again, now with the query parameters sorted:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch&unknown=polyfill`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill&zebra=striped`

Looking again at the list of URLs you might notice that the difference between 2. and 3. is the order the comma-separated features in the features parameter. We would need to sort those features by some manner in order to make them identical. Neither Varnish Cache not Fastly offer a pre-built function to sort a string, VCL also does not have a way to loop through items either. The way we solved this issue was by creating a function which takes a comma-separated string and turn it into a URL where each item in the comma-separated string is a query parameter, that way we can use the same `querystring.sort` function that we used earlier, we then turn the query parameters back into a comma-separated string.

Here is what that function looks like:

```
sub sort_comma_separated_querystring_parameter {
	#  Store the url without the querystring into a temporary header.
	declare local var.url STRING;
	set var.url = querystring.remove("https://www.example.com");
	declare local var.parameter STRING;
	set var.parameter = req.http.Sort-Parameter;
	# If query parameter does not exist or is empty, set it to ""
	set var.parameter = if(var.parameter != "", var.parameter, "");
	# Replace all `&` characters with `^`, this is because `&` would break the parameter up into pieces.
	set var.parameter = regsuball(var.parameter, "&", "^");
	# Replace all `,` characters with `&` to break them into individual query parameters
	# Append `1-` infront of all the query parameters to make them simpler to transform later
	set var.parameter = "1-" regsuball(var.parameter, ",", "&1-");
	set var.url = var.url "?" var.parameter;
	set var.url = querystring.sort(var.url);
	# Grab all the query parameters from the sorted url
	set var.parameter = regsub(var.url, "(.*)\?(.*)", "\2");
	# Reverse all the previous transformations to get back the single `features` query parameter value
	set var.parameter = regsuball(var.parameter, "1-", "");
	set var.parameter = regsuball(var.parameter, "&", ",");
	set var.parameter = regsuball(var.parameter, "\^", "&");
	set req.http.Sorted-Parameter = var.parameter;
}
```

<small>[You can also view the code on GitHub](https://github.com/Financial-Times/polyfill-service/blob/714623bdfff470b865c0e6f7746db5f6908f3acc/fastly/vcl/main.vcl#L3-L25)</small>

Using this function will make URLs 1, 2, 3 and 4 identical.

Listing the URLs again, after this function has been used:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch&unknown=polyfill`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill&zebra=striped`

The 5th and 6th URLs also return the same polyfill bundle as the other URLs, this is because `zebra` is not part of the API for configuring a polyfill bundle and if `unknown` is not set then the API sets it to `polyfill`. We can remove query parameters which are not part of the API and set default values for the options which have not been set. There are lots of options in the API and the code is rather verbose so I will link to the [source on GitHub if you want to read it](https://github.com/Financial-Times/polyfill-service/blob/714623bdfff470b865c0e6f7746db5f6908f3acc/fastly/vcl/main.vcl#L27-L137).

Here is a shortened version of the function:

<script type="application/json+fiddle">
{
  "title": "default values for api",
  "origins": [
    "https://polyfill.io"
  ],
  "vcl": {
    "recv": "# Store original url for logging purposes.\ndeclare local var.original-url STRING;\nset var.original-url = req.url;\n\n# Store the url without the querystring into a variable for use later.\ndeclare local var.url STRING;\nset var.url = querystring.remove(req.url);\n\n# (?i) makes the regex case-insensitive\n# The regex will match only if their are characters after `unknown=` which are not an ampersand (&).\nif (req.url.qs ~ \"(?i)[^&=]*unknown=([^&]+)\") {\n  # Parameter has already been set, use the already set value.\n  # re.group.1 is the first regex capture group in the regex above.\n  set var.url = querystring.set(var.url, \"unknown\", re.group.1);\n} else {\n  # Parameter has not been set, use the default value\n  set var.url = querystring.set(var.url, \"unknown\", \"polyfill\");\n}\n\nset req.url = var.url;\n\nlog \"Original url: \" var.original-url;\nlog \"Updated  url: \" var.url;"
  },
  "reqUrl": "/v2/polyfill.js",
  "reqMethod": "GET",
  "purgeFirst": true,
  "enableCluster": false,
  "enableShield": false
}
</script>

With all these functions in place the end result is that all 6 of those URLs become identical, which means they will have the same hash key inside Varnish Cache and therefore all point to the same single cached response, increasing the cache-hit ratio.

## Normalising the user-agent header inside Varnish Cache

In the previous section I omitted the fact that one of the options in the API is to set the User-Agent in the URL via the `ua` query parameter. TODO FINISH THIS.

\----- The part below is not finished, probably no reason to read it -----

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
