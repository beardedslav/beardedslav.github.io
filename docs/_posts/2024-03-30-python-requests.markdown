---
layout: post
title:  "Requests, httpx, backwards compatibility and how I realised dynamic typing is not always a good thing"
date:   2024-03-30 16:00:00 +0100
categories: python
---
A few days ago a colleague highlighted some warnings in our logs refering to unverified HTTPS requests.

```
InsecureRequestWarning: Unverified HTTPS request is being made to host 'REDACTED'. Adding certificate verification is strongly advised. See: https://urllib3.readthedocs.io/en/latest/advanced-usage.html#tls-warnings
```

We are using `requests` which uses `urllib3` under the hood, so seeing the error come from `urllib3` made sense.

The fix seemed simple, just make sure the `verify` parameter is set to `True` when making the request. We were already using it and setting it to a config value, however
1. the config value was not set at all in our parameter store,
2. the default value was `False`

Ok, let's just set the value in the parameter store and we're sorted, right? We did and suddenly we started seeing this in our logs
```
Could not find a suitable TLS CA certificate bundle, invalid path: true
```

Wait, why would setting the value to `True` cause this error?
After looking at [the bit of the code](https://github.com/psf/requests/blob/8dd3b26bf59808de24fd654699f592abf6de581e/src/requests/adapters.py#L299) responsible for raising `OSError` we figured out the verify parameter was being passed as a string instead of a boolean. Turns out [SSL/TLS certificate validation](https://github.com/psf/requests/blob/8dd3b26bf59808de24fd654699f592abf6de581e/src/requests/adapters.py#L274) in `requests` behaves differently depending on the type of the `verify` parameter.

```python
def cert_verify(self, conn, url, verify, cert):
    """
    ...
    :param verify: Either a boolean, in which case it controls whether we verify
        the server's TLS certificate, or a string, in which case it must be a path
        to a CA bundle to use
    ...
    """
```

The config parameter was defined like this:

```python
VERIFY_SSL: str = config("VERIFY_SSL", default=False)
```

For now let's ignore the fact that the default value was a boolean despite the type of the parameter being a string.
Ok, let's just change the type to `bool`

```python
VERIFY_SSL: bool = config("VERIFY_SSL", default=False)
```

Problem solved! However the more I looked at `HTTPAdapter.cert_verify` code the more I didn't like it. Taking a different code path depending on the type of the parameter is not a good idea, and it's only the docstring that explains this. On top of that `requests` doesn't use type hints, however this can be addressed by using [`typeshed`](https://github.com/python/typeshed). Still, this doesn't solve the issue of the code smell.

I remembered I used [`httpx`](https://www.python-httpx.org/) in a take-home exercise I did when I was applying for jobs last year. Since `httpx` is a more modern library I hoped for a more elegant approach to the same issue, however [I could not have been more wrong!](https://github.com/encode/httpx/blob/392dbe45f086d0877bd288c5d68abf860653b680/httpx/_client.py#L598). But why? Turns out most likely because it [tries to maintain a high degree of compatibility with `requests`](https://www.python-httpx.org/compatibility/).

So how is this all related to backwards compatibility and dynamic typing?

## Backwards compatibility

Backwards compatibility sounds like a great idea, however if anybody's code depends on a bug or a bad design decision in your library/package/framework/API, fixing it might (and most likely will) break it. 20 years ago [parts of Windows NT 4.0 and Windows 2000 source code were leaked](https://web.archive.org/web/20110805100108/http://www.microsoft.com/presspass/press/2004/Feb04/02-12windowssource.mspx) and as you can see [in this article](http://atdt.freeshell.org/k5/story_2004_2_15_71552_7795.html) certain bugs in *undocumented* Windows code could not be fixed because they were depended on by programs created both by Microsoft and third parties. Of course there's a massive value in keeping old code compatible with the newer version of your product, but if you keep doing this your users will have no incentive to upgrade their code and you will have to maintain the old code forever. [Look at Python 2 and 3](https://www.python.org/doc/sunset-python-2/), it took 12 years from releasing Python 3.0 to ending supprt for Python 2.7, [and not everyone moved to Python 3 in time](https://www.efinancialcareers.co.uk/news/2021/02/jpmorgan-still-has-its-python-2-issue).

## Dynamic typing

The problem I encountered
