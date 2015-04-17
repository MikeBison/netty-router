Netty-Router is a tiny Java library intended for use with Netty 4, to route HTTP
requests to your Netty handlers.

See `test <https://github.com/sinetja/netty-router/tree/master/src/test/scala/io/netty/handler/codec/http>`_
for example.

Netty-Router is based on `Jauter <https://github.com/sinetja/jauter>`_.
See also `Sinetja <https://github.com/sinetja/sinetja>`_.

Javadoc:

* `Netty-Router <http://sinetja.github.io/netty-router>`_
* `Netty <http://netty.io/4.0/api/index.html>`_

Create router
~~~~~~~~~~~~~

::

  import io.netty.handler.codec.http.router.Router;

  Router router = new Router()
    .GET      ("/articles",     IndexHandler.class)
    .GET      ("/articles/:id", ShowHandler.class)
    .POST     ("/articles",     CreateHandler.class)
    .GET      ("/download/:*",  DownloadHandler.class)      // ":*" must be the last token
    .GET_FIRST("/articles/new", NewHandler.class)           // This will be matched first
    .ANY_LAST ("/:*",           NotFound404Handler.class);  // This will be matched last, any method

Slashes at both ends are ignored, so these are the same:

::

  router.GET("articles",   IndexHandler.class)
  router.GET("/articles",  IndexHandler.class)
  router.GET("/articles/", IndexHandler.class)

You can remove routes by target or by path:

::

  router.removeTarget(IndexHandler.class)
  router.removePath("/articles")

Instead of using handler class, you can use handler instance. In this case,
the handler class should be annotated with
`Sharable <http://netty.io/4.0/api/io/netty/channel/ChannelHandler.Sharable.html>`_:

::

  // Optimize speed by precreating handler.
  // Optimize memory by sharing one handler for all requests.
  DeleteHandler deleteHandler = new DeleteHandler();
  router.DELETE("/articles/:id", deleteHandler);

Create handler
~~~~~~~~~~~~~~

Add ``handler`` to your Netty inbound pipeline, after the HTTP request decoder.
When a path is matched:

* ``handler`` will create a new instance of the matched handler class, and add it
  to the pipeline, right after ``handler`` itself. If you use handler instance as
  ``deleteHandler`` above, ``handler`` doesn't have to create a new instance.
* ``handler`` will pass a ``Routed`` (see below) to your handler.

::

  import io.netty.handler.codec.http.router.Router;
  import io.netty.handler.codec.http.router.Handler;

  public class PipelineInitializer extends ChannelInitializer<SocketChannel> {
    private static final router = new Router()
      .GET("/",             new ExampleHandler())
      .GET("/articles/:id", ExampleHandler.class)

    private static final Handler handler = new Handler(router);

    public void initChannel(SocketChannel ch) {
      ChannelPipeline p = ch.pipeline()
        .addLast(new HttpServerCodec)
        // Must use router.name() so that the router can add the
        // routed handler right after itself later
        .addLast(handler.name(), handler);
    }
  }

You may also want to add `BadClientSilencer <https://github.com/sinetja/netty-router/tree/master/src/main/java/io/netty/handler/codec/http/router/BadClientSilencer.java>`_.

Routed contains:

::

  boolean                   notFound()  // See "404 Not Found handler" below
  HttpRequest               request()
  String                    path()
  Map<String, String>       pathParams()
  Map<String, List<String>> queryParams()

``Routed`` is a `ReferenceCounted <http://netty.io/4.0/api/io/netty/util/ReferenceCounted.html>`_
via the wrapped request. If your handler is not a `SimpleChannelInboundHandler <http://netty.io/4.0/api/io/netty/util/ReferenceCounted.html>`_,
which does release automatically, you may need to manually call its ``release``.

Extract params from request
~~~~~~~~~~~~~~~~~~~~~~~~~~~

There are some utility methods in ``Routed``:

::

  // Returns path param first, then falls back to the first query param, or null.
  // Usually, you want to use this method most of the time.
  String       param(name)

  // Both path params and query params are returned.
  // Empty list is returned if there are no such params.
  List<String> params(name)

  // Returns the first query param, or null.
  List<String> queryParam(name)

404 Not Found handler
~~~~~~~~~~~~~~~~~~~~~

If no matched handler is found, by default the router will respond
``404 Not Found``.

If you want to pass your own 404 Not Found handler:

::

  router.notFound(My404Handler.class);

You know if your handler is called because of 404 Not Found or not, by checking
``Routed#notFound()``.

You can also use instance (your handler must be `sharable <http://netty.io/4.0/api/io/netty/channel/ChannelHandler.Sharable.html>`_):

::

  ChannelInboundHandler my404Handler = new My404Handler();
  router.notFound(my404Handler);

EventExecutorGroup
~~~~~~~~~~~~~~~~~~

By default your routed handler will run by Netty's IO thread. If your handler
takes time to run, e.g. blocking, Netty may not be able to accept new requests
or reply responses. In that case, you may specify your own
`EventExecutorGroup <http://netty.io/4.0/api/io/netty/util/concurrent/EventExecutorGroup.html>`_.

::

  int                poolSize     = Runtime.getRuntime().availableProcessors() * 2;
  EventExecutorGroup myThreadPool = new DefaultEventExecutorGroup(poolSize);
  router.group(myThreadPool);

Create reverse route
~~~~~~~~~~~~~~~~~~~~

::

  router.path(HttpMethod.GET, IndexHandler.class);
  // => "/articles"

You can skip HTTP method if there's no confusion:

::

  router.path(CreateHandler.class);
  // => "/articles"

You can specify params as map:

::

  // Things in params will be converted to String
  Map<Object, Object> params = new HashMap<Object, Object>();
  params.put("id", 123);
  router.path(ShowHandler.class, params);
  // => "/articles/123"

Convenient way to specify params:

::

  router.path(ShowHandler.class, "id", 123);
  // => "/articles/123"

You can specify an instance in pattern, but use the instance's class to create
path.

::

  IndexHandler cachedInstance = new IndexHandler();

  Router router = new Router<Object>()
    .pattern("/articles",     cachedInstance)
    .pattern("/articles/:id", ShowHandler.class);

  // These are the same:
  router.path(cachedInstance);
  router.path(IndexHandler.class);

Use with Maven
~~~~~~~~~~~~~~

::

  <dependency>
    <groupId>tv.cntt</groupId>
    <artifactId>netty-router</artifactId>
    <version>1.10</version>
  </dependency>

Tip:
In your Netty project, you should also add `Javassist <http://www.csg.ci.i.u-tokyo.ac.jp/~chiba/javassist/>`_,
it boosts Netty speed.

::

  <dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.19.0-GA</version>
  </dependency>
