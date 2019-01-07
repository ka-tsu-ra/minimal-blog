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
5. `polyfill.io/v3/polyfill.js?features=fetch,IntersectionObserver&callback=polyfillsLoaded&zebra=striped`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`

We want to be able to have all these different URLs use the same hash key, the way to do that is to get them all to have the same URL before the Varnish Cache creates the key.

If you look at the list of URLs you might notice that some of them have the exact same query parameters but in a different order. We can re-order the query parameters with a function that Fastly provide called [`querystring.sort`](https://docs.fastly.com/vcl/functions/querystring-sort/). Using just this function will make URLs 1 and 2 become the same as well as 3 and 4.

<script type="application/json+fiddle">
{
  "title": "Sorting Querystrings",
  "origins": [
    "https://polyfill.io"
  ],
  "vcl": {
    "recv": "# Store original url for logging purposes.\ndeclare local var.original-url STRING;\nset var.original-url = req.url;\n\nset req.url = querystring.sort(req.url);\n\nlog \"Original url: \" var.original-url;\nlog \"Updated  url: \" req.url;"
  },
  "reqUrl": "/v3/polyfill.js?features=fetch%2CIntersectionObserver&callback=polyfillsLoaded&zebra=striped",
  "reqMethod": "GET",
  "purgeFirst": true,
  "enableCluster": false,
  "enableShield": false
}
</script>

Listing the URLs again, now with the query parameters sorted:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=fetch,IntersectionObserver&zebra=striped`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`

Looking again at the list of URLs you might notice that the difference between 2. and 3. is the order the comma-separated features in the features parameter. We would need to sort those features by some manner in order to make them identical. Neither Varnish Cache not Fastly offer a pre-built function to sort a string, VCL also does not have a way to loop through items either. The way we solved this issue was by creating a function which takes a comma-separated string and turn it into a URL where each item in the comma-separated string is a query parameter, that way we can use the same `querystring.sort` function that we used earlier, we then turn the query parameters back into a comma-separated string.

Here is what that function looks like:

<script type="application/json+fiddle">
{
  "title": "Sorting CSVs",
  "origins": [
    "https://polyfill.io"
  ],
  "vcl": {
    "init": "sub sort_comma_separated_value {\n  # This function takes a CSV and tranforms it into a url where each\n  # comma-separated-value is a query-string parameter and then uses \n  # Fastly's querystring.sort function to sort the values. Once sorted\n  # it then turn the query-parameters back into a CSV.\n  # Set the CSV on the header `Sort-Value`.\n  # Returns the sorted CSV on the header `Sorted-Value`.\n\tdeclare local var.value STRING;\n\tset var.value = req.http.Sort-value;\n\n\t# If query value does not exist or is empty, set it to \"\"\n\tset var.value = if(var.value != \"\", var.value, \"\");\n\n\t# Replace all `&` characters with `^`, this is because `&` would break the value up into pieces.\n\tset var.value = regsuball(var.value, \"&\", \"^\");\n\n\t# Replace all `,` characters with `&` to break them into individual query values\n\t# Append `1-` infront of all the query values to make them simpler to transform later\n\tset var.value = \"1-\" regsuball(var.value, \",\", \"&1-\");\n\t\n\t# Create a url-like string in order for querystring.sort to work.\n\tset var.value = querystring.sort(\"https://www.example.com?\" var.value);\n\n\t# Grab all the query values from the sorted url\n\tset var.value = regsub(var.value, \"https://www.example.com\\?\", \"\");\n\t\n\t# Reverse all the previous transformations to get back the single `features` query value value\n\tset var.value = regsuball(var.value, \"1-\", \"\");\n\tset var.value = regsuball(var.value, \"&\", \",\");\n\tset var.value = regsuball(var.value, \"\\^\", \"&\");\n\n\tset req.http.Sorted-Value = var.value;\n}",
    "recv": "# Store original url for logging purposes.\ndeclare local var.original-url STRING;\nset var.original-url = req.url;\n\nif (req.url.qs ~ \"(?i)[^&=]*features=([^&]+)\") {\n  # Need to decode %2C into ,\n  set req.http.Sort-Value = urldecode(re.group.1);\n  call sort_comma_separated_value;\n  set req.url = querystring.set(req.url, \"features\", req.http.Sorted-Value);\n}\n\nset req.url = querystring.sort(req.url);\n\nlog \"Original url: \" var.original-url;\nlog \"Updated  url: \" req.url;"
  },
  "reqUrl": "/v3/polyfill.js?features=fetch%2CIntersectionObserver&callback=polyfillsLoaded&zebra=striped",
  "reqMethod": "GET",
  "purgeFirst": true,
  "enableCluster": false,
  "enableShield": false
}
</script>

<small>[You can also view the code on GitHub](https://github.com/Financial-Times/polyfill-service/blob/714623bdfff470b865c0e6f7746db5f6908f3acc/fastly/vcl/main.vcl#L3-L25)</small>

Using this function will make URLs 1, 2, 3 and 4 identical.

Listing the URLs again, after this function has been used:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch&zebra=striped`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`

The 5th URL has configured `zebra` to `striped`, `zebra` is not part of the API for configuring a polyfill bundle. Let's add a function to only keep query parameters in the URL which are actually part of the public API. We can achieve this with a function that Fastly provide called [`querystring.regfilter_except`](https://docs.fastly.com/vcl/functions/querystring-regfilter-except/). Using this function will make URLs 1, 2, 3, 4, and 5 become identical.

<script type="application/json+fiddle">
{
  "title": "Keep query parameters which are part of the API",
  "origins": [
    "https://polyfill.io"
  ],
  "vcl": {
    "recv": "# Store original url for logging purposes.\ndeclare local var.original-url STRING;\nset var.original-url = req.url;\n\n# Remove all querystring parameters which are not part of the public API.\nset req.url = querystring.regfilter_except(req.url, \"^(features|excludes|rum|unknown|flags|version|ua|callback|compression)$\");\n\nlog \"Original url: \" var.original-url;\nlog \"Updated  url: \" req.url;"
  },
  "reqUrl": "/v3/polyfill.js?features=fetch%2CIntersectionObserver&callback=polyfillsLoaded&zebra=striped",
  "reqMethod": "GET",
  "purgeFirst": false,
  "enableCluster": true,
  "enableShield": false
}
</script>

Listing the URLs again, after this function has been used:

1. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
2. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
3. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
4. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
5. `polyfill.io/v3/polyfill.js?callback=polyfillsLoaded&features=IntersectionObserver,fetch`
6. `polyfill.io/v3/polyfill.js?features=IntersectionObserver,fetch&callback=polyfillsLoaded&unknown=polyfill`


The 6th URL has set `unknown` to `polyfill` which just so happens to be the same as the default value for `unknown`. If we add the default values into the URL for the parameters which have not been set, we can make all the URLs in the list become identical. Remember, we use the URL as the hash key within Fastly, making these requests use the same URL internally will mean that they point to the same response in the cache, which is what we are trying to achieve.

<script type="application/json+fiddle">
{
  "title": "Setting default values, removing non-api parameters, sorting CSVs, and sorting querystring",
  "origins": [
    "https://polyfill.io"
  ],
  "vcl": {
    "recv": "# Store original url for logging purposes.\ndeclare local var.original-url STRING;\nset var.original-url = req.url;\n\t\ncall normalise_querystring_parameters_for_polyfill_bundle;\nset req.url = querystring.sort(req.url);\n\nlog \"Original url: \" var.original-url;\nlog \"Updated  url: \" req.url;",
    "init": "sub normalise_querystring_parameters_for_polyfill_bundle {\n\t# Remove all querystring parameters which are not part of the public API.\n\tset req.url = querystring.regfilter_except(req.url, \"^(features|excludes|rum|unknown|flags|version|ua|callback|compression)$\");\n\n\t# (?i) makes the regex case-insensitive\n\t# The regex will match only if their are characters after `features=` which are not an ampersand (&).\n\tif (req.url.qs ~ \"(?i)[^&=]*features=([^&]+)\") {\n\t\t# Parameter has already been set, use the already set value.\n\t\t# re.group.1 is the first regex capture group in the regex above.\n\t\tif (std.strlen(re.group.1) < 100) {\n\t\t\t# We add the value of the features parameter to this header\n\t\t\t# This is to be able to have sort_comma_separated_value sort the value\n\t\t\tset req.http.Sort-Value = urldecode(re.group.1);\n\t\t\tcall sort_comma_separated_value;\n\t\t\t# The header Sorted-Parameter now contains the sorted version of the features parameter.\n\t\t\tset req.url = querystring.set(req.url, \"features\", req.http.Sorted-Value);\n\t\t}\n\t} else {\n\t\t# Parameter has not been set, use the default value.\n\t\tset req.url = querystring.set(req.url, \"features\", \"default\");\n\t}\n\t\n\t# (?i) makes the regex case-insensitive\n\t# The regex will match only if their are characters after `excludes=` which are not an ampersand (&).\n\tif (req.url.qs ~ \"(?i)[^&=]*excludes=([^&]+)\") {\n\t\t# Parameter has already been set, use the already set value.\n\t\t# re.group.1 is the first regex capture group in the regex above.\n\t\tif (std.strlen(re.group.1) < 100) {\n\t\t\t# We add the value of the excludes parameter to this header\n\t\t\t# This is to be able to have sort_comma_separated_value sort the value\n\t\t\tset req.http.Sort-Value = urldecode(re.group.1);\n\t\t\tcall sort_comma_separated_value;\n\t\t\t# The header Sorted-Parameter now contains the sorted version of the excludes parameter.\n\t\t\tset req.url = querystring.set(req.url, \"excludes\", req.http.Sorted-Value);\n\t\t}\n\t} else {\n\t\t# If excludes is not set, set to default value \"\"\n\t\tset req.url = querystring.filter(req.url, \"excludes\");\n\t\tset req.url = if(req.url ~ \"\\?\", req.url \"&excludes=\", req.url \"?excludes=\");\n\t}\n\t\n\t# If rum is not set, set to default value \"0\"\n\tif (req.url.qs !~ \"(?i)[^&=]*rum=([^&]+)\") {\n\t\tset req.url = querystring.set(req.url, \"rum\", \"0\");\n\t}\n\t\n\t# If unknown is not set, set to default value \"polyfill\"\n\tif (req.url.qs !~ \"(?i)[^&=]*unknown=([^&]+)\") {\n\t\tset req.url = querystring.set(req.url, \"unknown\", \"polyfill\");\n\t}\n\n\t# If flags is not set, set to default value \"\"\n\tif (req.url.qs !~ \"(?i)[^&=]*flags=([^&]+)\") {\n\t\tset req.url = querystring.filter(req.url, \"flags\");\n\t\tset req.url = if(req.url ~ \"\\?\", req.url \"&flags=\", req.url \"?flags=\");\n\t}\n\n\t# If version is not set, set to default value \"\"\n\tdeclare local var.version STRING;\n\tif (req.url.qs !~ \"(?i)[^&=]*version=([^&]+)\") {\n\t\tset req.url = querystring.filter(req.url, \"version\");\n\t\tset req.url = if(req.url ~ \"\\?\", req.url \"&version=\", req.url \"?version=\");\n\t}\n\t\n\t# If ua is not set, normalise the User-Agent header based upon the version of the polyfill-library that has been requested.\n\tif (req.url.qs !~ \"(?i)[^&=]*ua=([^&]+)\") {\n\t\tif (req.url.qs ~ \"(?i)[^&=]*version=3\\.25\\.1(&|$)\") {\n\t\t\t# normalise_user_agent function is too large for Fastly Fiddle.\n\t\t\t# call normalise_user_agent;\n\t\t} else {\n\t\t\t# normalise_user_agent_latest function is too large for Fastly Fiddle.\n\t\t\t# call normalise_user_agent_latest;\n\t\t}\n\t\tset req.url = querystring.set(req.url, \"ua\", req.http.Normalized-User-Agent);\n\t}\n\n\t# If callback is not set, set to default value \"\"\n\tif (req.url.qs !~ \"(?i)[^&=]*callback=([^&]+)\") {\n\t\tset req.url = querystring.filter(req.url, \"callback\");\n\t\tset req.url = if(req.url ~ \"\\?\", req.url \"&callback=\", req.url \"?callback=\");\n\t}\n\t\n\t# If compression is not set, use the best compression that the user-agent supports.\n\tif (req.url.qs !~ \"(?i)[^&=]*compression=([^&]+)\") {\n\t\t# When Fastly adds Brotli into the Accept-Encoding normalisation we can replace this with: \n\t\t# `set req.url = querystring.set(req.url, \"compression\", req.http.Accept-Encoding || \"\")`\n\n\t\t# Before SP2, IE/6 doesn't always read and cache gzipped content correctly.\n\t\tif (req.http.Fastly-Orig-Accept-Encoding && req.http.User-Agent !~ \"MSIE 6\") {\n\t\t\tif (req.http.Fastly-Orig-Accept-Encoding ~ \"br\") {\n\t\t\t\tset req.url = querystring.set(req.url, \"compression\", \"br\");\n\t\t\t} elsif (req.http.Fastly-Orig-Accept-Encoding ~ \"gzip\") {\n\t\t\t\tset req.url = querystring.set(req.url, \"compression\", \"gzip\");\n\t\t\t} else {\n\t\t\t\tset req.url = querystring.set(req.url, \"compression\", \"\");\n\t\t\t}\n\t\t} else {\n\t\t\tset req.url = querystring.set(req.url, \"compression\", \"\");\n\t\t}\n\t}\n}\n\nsub sort_comma_separated_value {\n\t# This function takes a CSV and tranforms it into a url where each\n\t# comma-separated-value is a query-string parameter and then uses \n\t# Fastly's querystring.sort function to sort the values. Once sorted\n\t# it then turn the query-parameters back into a CSV.\n\t# Set the CSV on the header `Sort-Value`.\n\t# Returns the sorted CSV on the header `Sorted-Value`.\n\tdeclare local var.value STRING;\n\tset var.value = req.http.Sort-value;\n\n\t# If query value does not exist or is empty, set it to \"\"\n\tset var.value = if(var.value != \"\", var.value, \"\");\n\n\t# Replace all `&` characters with `^`, this is because `&` would break the value up into pieces.\n\tset var.value = regsuball(var.value, \"&\", \"^\");\n\n\t# Replace all `,` characters with `&` to break them into individual query values\n\t# Append `1-` infront of all the query values to make them simpler to transform later\n\tset var.value = \"1-\" regsuball(var.value, \",\", \"&1-\");\n\t\n\t# Create a querystring-like string in order for querystring.sort to work.\n\tset var.value = querystring.sort(\"?\" var.value);\n\n\t# Grab all the query values from the sorted url\n\tset var.value = regsub(var.value, \"\\?\", \"\");\n\t\n\t# Reverse all the previous transformations to get back the single `features` query value value\n\tset var.value = regsuball(var.value, \"1-\", \"\");\n\tset var.value = regsuball(var.value, \"&\", \",\");\n\tset var.value = regsuball(var.value, \"\\^\", \"&\");\n\n\tset req.http.Sorted-Value = var.value;\n}"
  },
  "reqUrl": "/v3/polyfill.js?features=IntersectionObserver%2Cfetch&callback=&zebra=striped",
  "reqMethod": "GET",
  "purgeFirst": true,
  "enableCluster": false,
  "enableShield": false
}
</script>

With all these functions in place the end result is that all 6 of those URLs become identical, which means they will have the same hash key inside Varnish Cache and therefore all point to the same single cached response, increasing the cache-hit ratio and decreasing the amount of requests that need to go all the way back to servers for polyfill.io.

## Normalising the user-agent header inside Varnish Cache

In the previous section I omitted the fact that one of the options in the API is to set the User-Agent in the URL via the `ua` query parameter. This is a very important feature with regards to caching because it means that we can have a different cached entry for each User-Agent making a request. If you are thinking that will be lots of cache entries and make the cache-hit ratio really low, you would be correct. The reason that would happen is because User-Agent values *vary a lot*, [whatismybrowser.com](https://developers.whatismybrowser.com/useragents/explore/) has collected 840,000 unique user-agents and keeps finding new ones every day.

Luckily for polyfill.io we only care about the User-Agent family, major, and minor version. In polyfill.io v1 and v2 we had an API endpoint that would take a User-Agent and return a version of it which was contained only the family, major, and minor version. This worked very well but introduced some complications in the VCL because the API endpoint was implemented in the polyfill.io server it meant that a request without a `ua` query parameter would need to first go to this endpoint to find out it's normalised user-agent value and then go to it's original destination to return a polyfill bundle.

In v3 we have implemented this API endpoint as a function in VCL, which has removed those complications around making two requests to the polyfill.io server. The way that we parse user-agents is by compiling the [uaparser.org](https://www.uaparser.org/) into VCL for the polyfill.io service and into JS for the [polyfill-library](https://github.com/Financial-Times/polyfill-library) npm package.

Unfortunately the code for parsing user-agents is too large to embed in a fiddle, you can view it at this link instead [fiddle.fastlydemo.net/fiddle/f857ab79](https://fiddle.fastlydemo.net/fiddle/f857ab79).

Normalising the User-Agent helps reduce the millions of different User-Agents down to the thousands of User-Agents that [uaparser.org](https://www.uaparser.org/) detects them as. We can improve on this still, polyfill.io supports 15 User-Agent families: Android, BlackBerry, Chrome, Edge, Edge Mobile, Firefox, Firefox Mobile, Internet Explorer, Internet Explorer Mobile, iOS Safari, iOS Chrome, Opera, Opera Mini, Opera Mobile, Samsung. If we detect that the User-Agent family is one we do not support, we can give it a generic unsupported name, such as `other` to ensure that all unsupported User-Agents generate the same internal URL and point to the same cached object.

Unfortunately the code for this is also too large to embed in a fiddle, you can view it at this link instead [fiddle.fastlydemo.net/fiddle/6a4eb0a1](https://fiddle.fastlydemo.net/fiddle/6a4eb0a1).

This change is the one that brought about the biggest improvement in our cache-hit ratio. Instead of getting a different cache-entry for every browser family and version, we only get different cache-entries for browser family and versions we support. As of writing this blog post, that works out to be roughly 300 different cache entries (Chrome has roughly 70 releases, Edge has 6 etc).
