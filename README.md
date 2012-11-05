django-rest-thumbnails
======================
Simple and scalable thumbnail generation via REST API

Why?
----
Most thumbnail libraries are broken for large scale sites by adopting one, or
both, of the following architectures:

1. Thumbnails of preset sizes are generated upon user upload and,
alternatively, via management commands
2. Thumbnails of arbitrary sizes are generated during template rendering

These options introduces a series of limitations, namely:

1. You can't generate many thumbnails upon user uploads since that blocks
workers and raises timeouts
2. You can't generate many thumbnails during template rendering for the same
reasons
3. You can't introduce extra-overhead by issuing hundreds of database/key-value
queries to retrieve image metadata
4. It's not always feasible to regenerate thumbnails for different sizes in
batch when you have hundreds of gigabytes of legacy data

This applications aims to solve it by separating concerns (requesting
thumbnails vs. generating thumbnails) via a HTTP API so you can leverage
fast proxies (e.g. Varnish) to cache and handle all the load, while being
flexible to let you generate thumbnails of arbitrary sizes on demand.

How to setup the server?
----
You setup a project to be your endpoint, like:

    from django.conf import settings
    from django.conf.urls.defaults import patterns, url, include

    urlpatterns = patterns('',
        url(r'^t/', include('restthumbnails.urls')),
    )

That's all. Now asking for a URL like:

    http://example.com/t/animals/kitten.jpg/100x100/crop/?secret=04c8f5c392a8d2b6ac86ad4e4c1dc5884a3ac317

Will resize and crop the file on `<MEDIA_ROOT>/animals/kitten.jpg` using the
default file storage and issue a permanent redirect to your media server.
Assuming your `MEDIA_URL` is `http://media.example.com`:

    http://media.example.com/animals/kitten_100x100_crop.jpg

That means you can setup a different worker process/server just to generate
thumbnails (freeing up your main site workers). It also means you can setup
a Varnish proxy and make it cache those responses, so only one request hits
the slow backend (the first one).

To avoid simultaneous requests to the same thumbnail dogpilling the server, the
view implements a lock (using `CACHE_BACKEND`). The intermediate requests will
simply return 404 while another worker is busy generating the thumbnail. Since
thumbnail generation will average <100ms on decent hardware, you can trade
consistency for high concurrency in large sites.

What about the client?
---------------------
There's a template tag to output these URLs automatically. Just add
`restthumbnails` to your `INSTALLED_APPS` and use the template tag as:

    {% load thumbnail %}

    {% thumbnail model.image_field "100x100" "crop"  as thumb %}
    <img src="{{ thumb.url }}"/>

Assuming `REST_THUMBNAILS_BASE_URL = "http://example.com/t/"` on settings, this
would output:

    <img src="http://example.com/t/animals/kitten.jpg/100x100/crop/?secret=04c8f5c392a8d2b6ac86ad4e4c1dc5884a3ac317"/>

The secret hash is derived from the `SECRET_KEY` setting, so if you are
serving the API from another server make sure these settings are in sync. This
is a mechanism to validate that requests come from trusted clients, so
malicious users can't bogg down your server by requesting thousands of kitten
images at 1280x1280.

You can define only one dimension:

    {% thumbnail model.image_field "500x" "crop" as fat_kitten %}

    {% thumbnail model.image_field "x500" "crop" as tall_kitten %}

You can also choose between 3 different methods:

- `crop`: Crop the image, maintaining the aspect ratio when possible.
- `scale`: Just scale the image until it matches one or all of the dimensions.
- `smart`: Smart crop the image by keeping the areas with the highest entropy.


Server settings
---------------

- `REST_THUMBNAILS_SOURCE_STORAGE` (Defaults to `DEFAULT_FILE_STORAGE`):
The Django file storage class used to open the source files (the path you
request on the URL).

- `REST_THUMBNAILS_STORAGE` (Defaults to `DEFAULT_FILE_STORAGE`):
The Django file storage class used to output the thumbnails.

- `REST_THUMBNAILS_SECRET_PARAM` (Defaults to `secret`):
The query string parameter appended to the URL.

- `REST_THUMBNAILS_LOCK_TIMEOUT` (Defaults to `30`):
The maximum amount of time workers are allowed to return 404 while a
thumbnail is generated by another worker. This is to avoid dogpilling only. If
the thumbnail fails to be generated because the source doesn't exist, it
will issue a permanent redirect anyway and let the media server handle
posterior requests.

Client settings
---------------

- `REST_THUMBNAILS_BASE_URL` (Defaults to `/`)
The base URL for your server. It can be relative to the same server (like
`/thumbnails`) or an absolute one (`http://thumbs.example.com/`)

- `REST_THUMBNAILS_VIEW_URL` (Defaults to '%(source)s/%(size)s/%(method)s/')
The view signature. You don't need to change this unless you hook the
`ThumbnailView` manually.

Thanks
------
The `processors` module is adapted from [`easy-thumbnails`](http://github.com/SmileyChris/easy-thumbnails/) by Chris Beaven,
under the BSD license.

