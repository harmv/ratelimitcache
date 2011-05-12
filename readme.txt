ratelimitcache for Django
=========================
By Simon Willison - http://simonwillison.net/
a fork by Chris LM

A rate limiter that uses Django's cache framework, with no requirement for a 
persistent data store.

More information: http://simonwillison.net/2009/Jan/7/ratelimitcache/

Installation:
    
    Place the ratelimitcache.py on your Python path.
    
    Configure your CACHE_BACKEND setting. For best results, use the memcached 
    backend - the other backends do not provide an atomic counter increment 
    and so may suffer from less effective limiting due to race conditions.
    
    Cache documentation: http://docs.djangoproject.com/en/dev/topics/cache/

Demo:
    cd demo/
    ./manage.py runserver 8008
    
    Now browse to:
        http://localhost:8008/
        http://localhost:8008/debug/
        http://localhost:8008/login/

Basic usage (max 20 requests every 3 minutes):

    from ratelimitcache import ratelimit
    
    @ratelimit(minutes = 3, requests = 20)
    def myview(request):
        # ...
        return HttpResponse('...')

Protecting a login form, i.e rate limit on IP address and attempted username:
    
    from ratelimitcache import ratelimit_post
    
    @ratelimit_post(minutes = 3, requests = 10, key_field = 'username')
    def login(request):
        # ...
        return HttpResponse('...')

You can also use it directly in urls.py. Here's how you would use it with 
the login() view function provided by Django:
    
    from ratelimitcache import ratelimit_post
    from django.contrib.auth.views import login
    
    urlpatterns = patterns('',
        #...
        (r'^login/$', ratelimit_post(
            minutes = 3, requests = 10, key_field = 'username'
        )(login)),
    )

Custom behaviour, e.g. logging when the rate limit condition fails:
    
    from ratelimitcache import ratelimit
    from my_logging_app.models import Log
    import datetime, pprint
    
    class ratelimit_with_logging(ratelimit):
        def disallowed(self, request):
            Log.objects.create(
                ip_address = request.META.get('REMOTE_ADDR'),
                path = request.path,
                counters = pprint.pformat(
                    self.get_counters(reqest)
                ),
                created = datetime.datetime.now()
            )
            return HttpResponseForbidden('Rate limit exceeded')
    
    @ratelimit_with_logging(minutes = 3, requests = 20)
    def myview(request):
        # ...
        return HttpResponse('...')
