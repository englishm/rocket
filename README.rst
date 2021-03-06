======
Rocket
======

Rocket is a library for quickly implementing a client-side API. 

Rocket is licensed under the `Apache Licence, Version 2.0 
<http://www.apache.org/licenses/LICENSE-2.0.html>`_


In short, Rocket is...
======================

Rocket is a collection of modules that make interacting with remote API's easy.
Rocket comes with a code generation system for using an IDL to describe the
remote source's API. 

There is complexity in implementing a remote API so Rocket tries to handle all
of that for you and let you focus on what's unique about this particular API.

Let's say we want to connect to the Awesome API and fetch the top item off the
'cool stuff' list with HTTP GET. The IDL for this API might look like below:

::

    FUNCTIONS = {
        'get_awesome_list': {
            'get': [
                ('list_name', str, []),
                ('page', int, ['optional']),
                ('size', str, ['optional']),
            ],
        }

This is what using the the Awesome API rocket would look like:

::

    from r_awesome import AwesomeAPI
    awesome_api = AwesomeAPI(api_key='some key', api_secret_key='SHHHHH')

    response = awesome_api.get_awesome_list.get('cool stuff', page=1, size=1)

I have been able to implement entire API's with multiple namespaces in an afternoon
using this method and found the total lines of codes to often be below half of the
other implementation offerings.


Modules
=======

Rocket comes with multiple API implementations, but they are not installed
by default. Each module has a setup.py for installing but Rocket doesn't install them
by default.

Currently packaged:
`Sailthru <https://github.com/exfm/rocket/tree/master/modules/r_sailthru/>`_,
`Songkick <https://github.com/exfm/rocket/tree/master/modules/r_songkick/>`_,
`Echonest <https://github.com/exfm/rocket/tree/master/modules/r_echonest/>`_,
`Twilio <https://github.com/exfm/rocket/tree/master/modules/r_twilio/>`_, 
`Twitter <https://github.com/exfm/rocket/tree/master/modules/r_twitter/>`_, 
`Exfm <https://github.com/exfm/rocket/tree/master/modules/r_exfm/>`_.

If you're only looking to use one of these modules, then you can skip the rest of 
the documentation.

People looking to learn more about how Rocket works should continue reading and then
checkout `r_simple <https://github.com/exfm/rocket/tree/master/modules/r_simple/>`_ 
in the modules directory.


Rocket's design
===============

Rocket uses an interface description language, specified by Rocket implementors,
to generate Python code that implements each API call. 

Rocket is designed to help programmers focus on the details of an API
implementation, rather than the details of implementing an API.

If a remote API consists of only one namespace, called ``email``, which goes over
HTTP POST and takes two arguments email and vars, with vars being optional,
your API implementation will look close to this.

That's as minimal as I've been able to get for describing an API. Perhaps
a different structure could be used, but the idea remains the same. To
describe the API in a language agnostic way and generate code that implements
it.

An API implementer would then subclass Rocket, like ``class Sailthru(Rocket)``
and then override ``__init__()`` to pass that ``FUNCTIONS`` list to rocket for
code generation.

An API user would not have to worry about this stuff, as it is behind the
scenes of the Rocket framework.
    

Callbacks
=========

To implement a Rocket you subclass Rocket and then make use of callbacks
to adjust functionality. Most common are adjustments to how query urls are
made, how errors are handled when you receive http 200 responses regardless
of errors or how query arguments are handled before you send them to the
remote source.

r_sailthru is a good example of how ``check_error``, ``build_query_args`` and
``gen_query_url`` can be used. Here is how each one works.

::

    def check_error(self, response):
        """Checks if the given API response is an error, and then raises
        the appropriate exception.
        """
        if type(response) is dict and response.has_key('error'):
            raise rocket.RocketAPIException(response['error'], response['errormsg'])

We see that check_error receives a response, which was parsed from json 
into a python dict called response. We found that it had an error key,
so we raise an exception containing the error info found in the dict.

::

    def build_query_args(self, *args, **kwargs):
        """Overrides Rocket's build_query_arg to set signing_alg to
        sign_sorted_values
        """
        return super(Sailthru, self).build_query_args(signing_alg=sign_sorted_values,
                                                      *args, **kwargs)

The sailthru API requires signing our requests, but Rocket makes no
assumptions on signing by default. We override ``build_query_args`` to
call ``build_query_args`` with ``sign_sorted_values`` for it's signing
algorithm. ``sign_sorted_values``, along with some other choices, are
implemented in ``rocket.auth`` module.

::

    def gen_query_url(self, url, function, format=None, method=None, get_args=None):
        """Sailthru urls look like 'url/function'.

        Example: http://api.sailthru.com/email
        """
        return '%s/%s' % (url, function)

The callback handles the data known about the call and generates the
URL string that handles the call. Each API is different here, so this
callback allows the flexibility of looking at the relevant information
and generating what you think it is.


Namespace handling
==================

Sometimes namespaces are complicated and instead of being simple like
'email' they have some complexity like ``group/subgroup.method``. Or 
perhaps variables even turn up in the url like ``something/{user_id}/feed.json``
Rocket handles this by offering several functions to handle how that string
is formatted into something that is compatible with the code generation.

It's easy enough to think of this functions as a *namespace pair
generator*. We'll see this again in the next section.

Let's look at one: ``rocket.proxies.gen_ns_pair_multi_delim``.

:: 

    def gen_ns_pair_multi_delim(ns, delims=['\/', '\.']):
        """..."""
        def title_if_lower(nnss):
            if not nnss.isupper():
                return nnss.title()
            return nnss
    
        groups = re.split('|'.join(delims), ns) 
        ns_fun = ''.join(groups)
        ns_title = ''.join([title_if_lower(g) for g in groups])
        return (ns_fun, ns_title)

    
The purpose of this function is to generate namespace keys from the
string found in the ``FUNCTIONS`` list. If we see ``SMS/Messages``, like 
found in ``r_twilio``, we translate this to ``SMSMessages`` and 
``SMSMessages`` which are then used for ``twilio.SMSMessages.post(...)``
and ``SMSMessagesProxy``, as attached to the Rocket.

We make use of this function by passing it in as part of Rocket's
``__init__()``.

::

    class Twilio(rocket.Rocket):
        """..."""
        def __init__(self, *args, **kwargs):
            super(Twilio, self).__init__(FUNCTIONS,
                                         gen_namespace_pair=gen_namespace_pair,
                                         ...)
    
Often enough, you won't need these overrides, but you'll be happy 
rocket handles a few of them easily when they come up. 

Rocket doesn't implement the most flexible by default because it aims to keep
performance light unless additional handling is desired.


URL's with Variables
====================

Variables sometimes turn up in the way URL's are constructed. Like perhaps a
feed system with ``api.songkick.com/api/3.0/artists/<artist_id>/calendar.json``.
Rocket handles url's with variables with two helper functions.

Imagine we have this ``FUNCTIONS`` list.

::

    FUNCTIONS = {
        'artists/{artist_id}/calendar': {
            'get': [
                ('artist_id', str, []),
            ],
        }

Rocket generates access to this namespace by replacing the ``{variable}`` with 
an underscore. We see this as ``Artists_CalendarProxy`` and 
``artists_calendar.get().``

This is done by using proxies.gen_ns_pair_multi_vars as the *namespace pair
generator*. This function can handle multiple delimiters, like '/', and
handles variables where a regex can describe them. In this case, I'm using
Rocket's default which is ``'{(\w+)}'``.

Rocket then implements gen_query_url to fill in the variable's values with
values from the caller. This means ``{artist_id}`` gets replaced with the artist's
id.

::

    artist_id = '258948'
    songkick.artists_calendar.get(artist_id)

This gets translated to a URL like: 
``api.songkick.com/api/3.0/artists/258948/calendar.json``.


Code generation using proxies
=============================

Rocket has a module called proxies which contain some functions for
generating callable objects from IDL's. The Proxy class represents
a namespace. It then generatescode representing 'get' or 'post', as 
found in ``FUNCTIONS``, and attaches them to the Proxy classes. This
is how Rocket maps particular funcitons into an API's namespace.

During Rocket's ``__init__()`` process, it calls ``generate_proxies(FUNCTIONS)``
and receives back a map of Proxy classes, each with ``get()`` or ``post()``
functions attached to them, as describes in ``FUNCTIONS``. These proxy
classes are then attached to our Rocket and we now have generated python
code that's ready for use.

The Rocket itself is what maps this data into http calls. Becaues of
this, to implement a remote API is to implement a Rocket. A use 
then instantiates your implementation and uses the generated functions
from your implementation's ``FUNCTIONS`` list.

See ``rocket.proxies`` or ``Rocket.__init__()`` for more details.


Http handling
=============

Rocket's ``http_handling.py`` module contains a few functions for handling
rocket's http interactions. The main function here is ``urlread()`` which
takes some arguments for tweaking the call, like which http method
(GET, POST, DELETE) to use or if ``basic_auth`` is turned on.

Functionality for file handling will be in there soon but is not complete.


Auth
====

Auth currently contains some functions for signing API requests and
basic_auth. For request signatures, ``sign_args`` and ``sign_sorted_values`` 
are available. Often enough a timestamp can be used to limit the 
lifespan of the signature.

``sign_args`` takes the request arguments, the secret key and a hashing
algorithm (defaults to md5). This algorithm concatenates strings of
the arguments, like ``arg1=val1arg2=val2``, and generates the key like:

::
  
    # get string of args like 'arg1=val1arg2=val2'
    s = _join_kv_pairs(args, hash_alg=hash_alg)
    # note: this algorithm *postfixes* s with the key
    hash_input = s + api_secret_key
    return hash_alg(hash_input).hexdigest()

``sign_sorted_values`` is similar, but it's signature string is a sorted
list of the request's values, like 'avalue1value2zebra1' and prefixes
this string with the secret key for it's signature.

Each API is different. :)

::

    # extact flattened list of values found in args
    values = _extract_param_values(args)
    arranged_args = sorted(values)
    s = ''.join(arranged_args)
    # note: this algorithm *prefixes* s with the key
    hash_input = api_secret_key + s 
    return hash_alg(hash_input).hexdigest()


Install It
==========

::

    python ./setup.py install

pip / easy_install support on the way


Author
======

James Dennis <james@extension.fm>
