This tiny Java library is intended for use with Netty 4, to route HTTP requests
to your Netty handlers. It is based on
`Jauter <https://github.com/xitrum-framework/jauter>`_.

See `test <https://github.com/xitrum-framework/netty-router/tree/master/src/test/scala/io/netty/handler/codec/http>`_
for example.

Use with Maven
~~~~~~~~~~~~~~

::

  <dependency>
    <groupId>tv.cntt</groupId>
    <artifactId>netty-router</artifactId>
    <version>1.0</version>
  </dependency>

Tip: You should also add `Javassist <http://javassist.org/>`_, it boosts Netty 4+ speed.

::

  <dependency>
    <groupId>org.javassist</groupId>
    <artifactId>javassist</artifactId>
    <version>3.18.2-GA</version>
  </dependency>

Create router
~~~~~~~~~~~~~

::

  import io.netty.handler.codec.http.HttpMethod;
  import io.netty.handler.codec.http.Router;

  Router router = new Router()
    .pattern(HttpMethod.GET,  "/articles",     IndexHandler.class)
    .pattern(HttpMethod.GET,  "/articles/:id", ShowHandler.class)
    .pattern(HttpMethod.POST, "/articles",     CreateHandler.class);

Instead of using handler class, you can use handler instance:

::

  DeleteHandler deleteHandler = new DeleteHandler();
  router.pattern(HttpMethod.DELETE, "/articles/:id", deleteHandler);

Add router to pipeline
~~~~~~~~~~~~~~~~~~~~~~

Add ``router`` to your Netty inbound pipeline, after the HTTP request decoder.
When a path is matched:

* ``router`` will create a new instance of the matched handler class, and add it
  to the pipeline, right after ``router`` itself. If you use handler instance as
  ``deleteHandler`` above, ``router`` doesn't have to create a new instance.
* ``router`` will add path params and query params to the request as headers.
* ``router`` will passes the current HTTP request to your handler.

::

  public class ExampleInitializer extends ChannelInitializer<SocketChannel> {
    private static final router = new Router()
      .pattern(HttpMethod.GET, "/",             new ExampleHandler())
      .pattern(HttpMethod.GET, "/articles/:id", ExampleHandler.class)

    public void initChannel(SocketChannel ch) {
      ChannelPipeline p = ch.pipeline();
      p.addLast(new HttpServerCodec);
      p.addLast(router.name(), router);  // Must use router.name()
    }
  }

Extract params from request
~~~~~~~~~~~~~~~~~~~~~~~~~~~

When the request is routed to your handler by ``router``, you can extract path
params and query params from its headers:

::

  // Use path params first, then fall back to query params.
  // Usually, you want to use this method most of the time.
  String Router.param(req)

  // Both path params and query params are returned.
  // Empty list is returned if there are no such params.
  List<String> Router.params(req)

  Map<String, String>       Router.pathParams(req)
  Map<String, List<String>> Router.queryParams(req)

If you want to clean headers set by the router:

::

  Router.cleanHeaders(req);

404 Not Found handler
~~~~~~~~~~~~~~~~~~~~~

If no matched handler is found, by default the router will respond
``404 Not Found``.

If you want to pass your own 404 Not Found handler:

::

  ChannelInboundHandler my404Handler = ...;
  Router                router       = new Router(my404Handler);

EventExecutorGroup
~~~~~~~~~~~~~~~~~~

By default your routed handler will run by Netty's IO thread. If your handler
takes time to run, e.g. blocking, Netty may not be able to accept new requests
or reply responses. In that case, you may specify your own
`EventExecutorGroup <http://netty.io/4.0/api/io/netty/util/concurrent/EventExecutorGroup.html>`_.

::

  int                poolSize     = Runtime.getRuntime().availableProcessors() * 2;
  EventExecutorGroup myThreadPool = new DefaultEventExecutorGroup(poolSize);
  Router             router       = new Router(myThreadPool);

If you want to specify both ``EventExecutorGroup`` and 404 Not Found handler:

::

  Router router = new Router(myThreadPool, my404Handler);

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
path. This feature is useful if you want to create web frameworks:

::

  // Optimize speed by precreating.
  // Optimize memory by sharing for all requests.
  IndexHandler cachedInstance = new IndexHandler();

  Router router = new Router<Object>()
    .pattern("/articles",     cachedInstance)
    .pattern("/articles/:id", ShowHandler.class);

  // These are the same:
  router.path(cachedInstance);
  router.path(IndexHandler.class);