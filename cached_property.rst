`cached_property`
=================

The `cached_property` decorator is one of my favourite utilities in Django and
I frequrently use it across all of my projects. It is a decorator that can 
be used in place of the standard `@property` decorator and works pretty much
the same way except for one detail: **the result of the property function is
cached on the instance**.

This is useful if you have a computationally heavy method computing your
property's value or make requests to external services such as a database or
web API. It looks like this:

.. code:: python

   class MyObject(object):

       @cached_property
       def compute_heavy_method(self):
           ...
           return result


Where It Can Be Useful
----------------------

Let's imagine we have want to use a web API to retireve the name for a given
colour value. The colour is defined by a hex value, e.g. `#ffffff`. We
represent the colour itself as a class and provide a `name` property that
queries the external API to retrieve the name:

.. code:: python

    class Color(object):

        def __init__(self, hex):
            self.hex = hex

        def _request_colour_name(self, hex):
            print "Requesting #{}".format(hex)
            rsp = requests.get(API_ENDPOINT.format(hex))
            return rsp.json()[0].get("title")

        @property
        def name(self):
            return self._request_colour_name(self.hex)


Let's take a look at what that might look like when we execute it in a Python
shell:

.. code:: python

   >>> c = Color('ffffff')
   >>> c.name

   Requesting #ffffff
   white

   >>> c.name

   Requesting #ffffff
   white


As you can see, every call to `c.name` results in a web request to the API.
This can become problematic when the API's response time increases or the
network is experiencing problems.

Using the `cached_property` decorator, we can minimize the impact of these
requests on our example application. Since the result of the property method
is cached, we only have to make *a single* request to retrieve the name from
the API:

.. code:: python

    from django.utils.functional import cached_property

    class Color(object):
        ...

        @cached_property
        def name(self):
            return self._request_colour_name(self.hex)


Executing the same commands as above in a Python shell now results in this:

.. code:: python

    >>> c = Color('ffffff')
    >>> c.name

    Requesting #ffffff
    white

    >>> c.name

    white


You don't have to use the `cached_property` decorator for this. Implementing
the caching yourself would be pretty easy:

.. code:: python

    @property
    def name(self):
        if self._name is None:
            self._name = self._request_colour_name(self.hex)
        return self._name


However, a smart person one said to me "the best code is the code that you
don't write". I fully agree and would rather use `cached_property` instead of
writing it myself and include tests with it that verify the behaviour...and I'm
sure you do too 😉😉.


Beware Of Querysets
-------------------

You have to be careful when you are caching querysets. As you probably already
know, querysets in Django are lazily evaluated. That means a queryset instance
doesn't actually query the database until it's result is accessed. 

This means, that when you use a queryset within a `cached_property`
decoratorated method, you are only caching the queryset, not its resulting 
models. Especially for list of objects, you might want to force the evaluation
of the queryset by turning it into a `list` or `tuple`.

.. code:: python

    class VeryComplicatedView(object):

        @cached_property
        def get_followers(self):
            return list(User.objects.filter(followed_by=self.user))


References
----------

* `Django docs <https://docs.djangoproject.com/en/1.8/ref/utils/#django.utils.functional.cached_property>`_
* `Source <https://docs.djangoproject.com/en/1.8/_modules/django/utils/functional/#cached_property>`_
