scrapy-ninja-proxies
=======================

.. image:: https://img.shields.io/pypi/v/scrapy-rotating-proxies.svg
   :target: https://pypi.python.org/pypi/scrapy-rotating-proxies
   :alt: PyPI Version

.. image:: https://travis-ci.org/TeamHG-Memex/scrapy-rotating-proxies.svg?branch=master
   :target: http://travis-ci.org/TeamHG-Memex/scrapy-rotating-proxies
   :alt: Build Status


This package provides a Scrapy_ middleware to use rotating proxies, automatically provided within scrapy.ninja_ services subscription.

.. _Scrapy: https://scrapy.org/
.. _scrapy.ninja: https://scrapy.ninja/

License is MIT.

Installation
------------

::

    pip3 install git+https://github.com/fliot/scrapy-ninja-proxies.git

Usage
-----

Add ``ROTATING_NINJA_KEY`` option (something like 'KJHGFSERTYUIO87654323ERFGHUIO876543'), obtained from your **scrapy.ninja subscription**, to your settings.py::

    ROTATING_NINJA_KEY = 'KJHGFSERTYUIO87654323ERFGHUIO876543'



Then add rotating_proxies middlewares to your DOWNLOADER_MIDDLEWARES::

    DOWNLOADER_MIDDLEWARES = {
        # ...
        'rotating_proxies.middlewares.RotatingProxyMiddleware': 610,
        'rotating_proxies.middlewares.BanDetectionMiddleware': 620,
        # ...
    }

After this all requests will be proxied using one of the proxies obtained though your proxy services scrapy.ninja subscription.

**Being scrapy.ninja subscribers, your active/working proxy list is automatically updated each 15 minutes.**

Requests with "proxy" set in their meta are not handled by
scrapy-ninja-proxies. To disable proxying for a request set
``request.meta['proxy'] = None``; to set proxy explicitly use
``request.meta['proxy'] = "<my-proxy-address>"``.


Concurrency
-----------

By default, all default Scrapy concurrency options (``DOWNLOAD_DELAY``,
``AUTHTHROTTLE_...``, ``CONCURRENT_REQUESTS_PER_DOMAIN``, etc) become
per-proxy for proxied requests when RotatingProxyMiddleware is enabled.
For example, if you set ``CONCURRENT_REQUESTS_PER_DOMAIN=2`` then
spider will be making at most 2 concurrent connections to each proxy,
regardless of request url domain.

Customization
-------------

``scrapy-ninja-proxies`` keeps track of working and non-working proxies,
and re-checks non-working from time to time.

Detection of a non-working proxy is site-specific.
By default, ``scrapy-ninja-proxies`` uses a simple heuristic:
if a response status code is not 200, response body is empty or if
there was an exception then proxy is considered dead.

You can override ban detection method by passing a path to
a custom BanDectionPolicy in ``ROTATING_PROXY_BAN_POLICY`` option, e.g.::

    # settings.py
    ROTATING_PROXY_BAN_POLICY = 'myproject.policy.MyBanPolicy'

The policy must be a class with ``response_is_ban``
and ``exception_is_ban`` methods. These methods can return True
(ban detected), False (not a ban) or None (unknown). It can be convenient
to subclass and modify default BanDetectionPolicy::

    # myproject/policy.py
    from rotating_proxies.policy import BanDetectionPolicy

    class MyPolicy(BanDetectionPolicy):
        def response_is_ban(self, request, response):
            # use default rules, but also consider HTTP 200 responses
            # a ban if there is 'captcha' word in response body.
            ban = super(MyPolicy, self).response_is_ban(request, response)
            ban = ban or b'captcha' in response.body
            return ban

        def exception_is_ban(self, request, exception):
            # override method completely: don't take exceptions in account
            return None

Instead of creating a policy you can also implement ``response_is_ban``
and ``exception_is_ban`` methods as spider methods, for example::

    class MySpider(scrapy.Spider):
        # ...

        def response_is_ban(self, request, response):
            return b'banned' in response.body

        def exception_is_ban(self, request, exception):
            return None

It is important to have these rules correct because action for a failed
request and a bad proxy should be different: if it is a proxy to blame
it makes sense to retry the request with a different proxy.

Non-working proxies could become alive again after some time.
``scrapy-rotating-proxies`` uses a randomized exponential backoff for these
checks - first check happens soon, if it still fails then next check is
delayed further, etc. Use ``ROTATING_PROXY_BACKOFF_BASE`` to adjust the
initial delay (by default it is random, from 0 to 5 minutes). The randomized
exponential backoff is capped by ``ROTATING_PROXY_BACKOFF_CAP``.

Settings
--------

* ``ROTATING_NINJA_RENEW_INTERVAL`` - renewing proxy list interval in seconds,
  1200 by default;
* ``ROTATING_PROXY_LIST`` - additionnal proxy list (if ROTATING_NINJA_KEY
  is None, it will be the only available proxies)
* ``ROTATING_PROXY_LOGSTATS_INTERVAL`` - stats logging interval in seconds,
  30 by default;
* ``ROTATING_PROXY_CLOSE_SPIDER`` - When True, spider is stopped if
  there are no alive proxies. If False (default), then when there is no
  alive proxies all dead proxies are re-checked.
* ``ROTATING_PROXY_PAGE_RETRY_TIMES`` - a number of times to retry
  downloading a page using a different proxy. After this amount of retries
  failure is considered a page failure, not a proxy failure.
  Think of it this way: every improperly detected ban cost you
  ``ROTATING_PROXY_PAGE_RETRY_TIMES`` alive proxies. Default: 5.

  It is possible to change this option per-request using
  ``max_proxies_to_try`` request.meta key - for example, you can use a higher
  value for certain pages if you're sure they should work.
* ``ROTATING_PROXY_BACKOFF_BASE`` - base backoff time, in seconds.
  Default is 300 (i.e. 5 min).
* ``ROTATING_PROXY_BACKOFF_CAP`` - backoff time cap, in seconds.
  Default is 3600 (i.e. 60 min).
* ``ROTATING_PROXY_BAN_POLICY`` - path to a ban detection policy.
  Default is ``'rotating_proxies.policy.BanDetectionPolicy'``.


Contributing
------------

* source code: https://github.com/TeamHG-Memex/scrapy-rotating-proxies
* bug tracker: https://github.com/TeamHG-Memex/scrapy-rotating-proxies/issues

To run tests, install tox_ and run ``tox`` from the source checkout.

.. _tox: https://tox.readthedocs.io/en/latest/

----

.. image:: https://hyperiongray.s3.amazonaws.com/define-hg.svg
    :target: https://www.hyperiongray.com/?pk_campaign=github&pk_kwd=scrapy-rotating-proxies
    :alt: define hyperiongray
