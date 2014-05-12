.. index::
   single: Routing
   single: Components; Routing

Le Composant Routing
====================

   Le composant routing relie une requête HTTP à un ensemble de variables
   de configuration.

Installation
------------

Vous pouvez installer le composant de deux manières différentes :

* :doc:`Installez-le via Composer </components/using_components>` (``symfony/routing`` sur `Packagist`_);
* Utilisez le dépôt Git officiel (https://github.com/symfony/Routing).

Utilisation
-----------

Pour mettre en place un système basique de routing, vous avez besoin de trois parties :

* Une :class:`Symfony\\Component\\Routing\\RouteCollection`, contenant les définitions de routes (instances de la classe :class:`Symfony\\Component\\Routing\\Route`)
* Un :class:`Symfony\\Component\\Routing\\RequestContext`, contennat les informations de la requête
* Un :class:`Symfony\\Component\\Routing\\Matcher\\UrlMatcher`, qui effectue le lien entre une requête et une route

Voici un exemple rapide. Notez que nous partons du principe que vous avez déjà
configuré votre autoloader pour charger le composant Routing ::

    use Symfony\Component\Routing\Matcher\UrlMatcher;
    use Symfony\Component\Routing\RequestContext;
    use Symfony\Component\Routing\RouteCollection;
    use Symfony\Component\Routing\Route;

    $route = new Route('/foo', array('controller' => 'MyController'));
    $routes = new RouteCollection();
    $routes->add('route_name', $route);

    $context = new RequestContext($_SERVER['REQUEST_URI']);

    $matcher = new UrlMatcher($routes, $context);

    $parameters = $matcher->match('/foo');
    // array('controller' => 'MyController', '_route' => 'route_name')

.. note::

    Soyez vigilants en utilisant ``$_SERVER['REQUEST_URI']``, comme il
    peut contenir n'importe quels paramètres de l'URL, qui pourraient
    causer des problèmes avec la liaison avec une route. Une façon facile
    de régler le problème est d'utiliser le composant HttpFoundation
    comme expliqué :ref:`plus bas <components-routing-http-foundation>`.

Vous pouvez ajouter autant de routes que vous le souhaitez à une
:class:`Symfony\\Component\\Routing\\RouteCollection`.

La méthode :method:`RouteCollection::add() <Symfony\\Component\\Routing\\RouteCollection::add>`
prend deux arguments. Le premier est le nom de la route. Le second est un objet
de type class:`Symfony\\Component\\Routing\\Route`, qui attend une URL et
un tableau de quelques variables personnalisées dans son constructeur. Ce
tableau de variables peut être *n'importe quoi* qui pourrait être signifiant
pour votre application, et est disponible lorsqu'une route est trouvée.

Si aucun matching n'a été fait une :class:`Symfony\\Component\\Routing\\Exception\\ResourceNotFoundException`
sera jetée.

En plus de votre tableau de variables personnalisées, une clé ``_route``
est ajoutée, qui porte le nom de la route liée.

Définir les routes
~~~~~~~~~~~~~~~~~~

Une définition complète de route peut contenir sept parties :

1. The URL path route. This is matched against the URL passed to the `RequestContext`,
   and can contain named wildcard placeholders (e.g. ``{placeholders}``)
   to match dynamic parts in the URL.

2. An array of default values. This contains an array of arbitrary values
   that will be returned when the request matches the route.

3. An array of requirements. These define constraints for the values of the
   placeholders as regular expressions.

4. An array of options. These contain internal settings for the route and
   are the least commonly needed.

5. A host. This is matched against the host of the request. See
   :doc:`/components/routing/hostname_pattern` for more details.

6. An array of schemes. These enforce a certain HTTP scheme (``http``, ``https``).

7. An array of methods. These enforce a certain HTTP request method (``HEAD``,
   ``GET``, ``POST``, ...).

Take the following route, which combines several of these ideas::

   $route = new Route(
       '/archive/{month}', // path
       array('controller' => 'showArchive'), // default values
       array('month' => '[0-9]{4}-[0-9]{2}', 'subdomain' => 'www|m'), // requirements
       array(), // options
       '{subdomain}.example.com', // host
       array(), // schemes
       array() // methods
   );

   // ...

   $parameters = $matcher->match('/archive/2012-01');
   // array(
   //     'controller' => 'showArchive',
   //     'month' => '2012-01',
   //     'subdomain' => 'www',
   //     '_route' => ...
   //  )

   $parameters = $matcher->match('/archive/foo');
   // throws ResourceNotFoundException

In this case, the route is matched by ``/archive/2012-01``, because the ``{month}``
wildcard matches the regular expression wildcard given. However, ``/archive/foo``
does *not* match, because "foo" fails the month wildcard.

.. tip::

    If you want to match all URLs which start with a certain path and end in an
    arbitrary suffix you can use the following route definition::

        $route = new Route(
            '/start/{suffix}',
            array('suffix' => ''),
            array('suffix' => '.*')
        );

Using Prefixes
~~~~~~~~~~~~~~

You can add routes or other instances of
:class:`Symfony\\Component\\Routing\\RouteCollection` to *another* collection.
This way you can build a tree of routes. Additionally you can define a prefix
and default values for the parameters, requirements, options, schemes and the
host to all routes of a subtree using methods provided by the
``RouteCollection`` class::

    $rootCollection = new RouteCollection();

    $subCollection = new RouteCollection();
    $subCollection->add(...);
    $subCollection->add(...);
    $subCollection->addPrefix('/prefix');
    $subCollection->addDefaults(array(...));
    $subCollection->addRequirements(array(...));
    $subCollection->addOptions(array(...));
    $subCollection->setHost('admin.example.com');
    $subCollection->setMethods(array('POST'));
    $subCollection->setSchemes(array('https'));

    $rootCollection->addCollection($subCollection);

Set the Request Parameters
~~~~~~~~~~~~~~~~~~~~~~~~~~

The :class:`Symfony\\Component\\Routing\\RequestContext` provides information
about the current request. You can define all parameters of an HTTP request
with this class via its constructor::

    public function __construct(
        $baseUrl = '',
        $method = 'GET',
        $host = 'localhost',
        $scheme = 'http',
        $httpPort = 80,
        $httpsPort = 443,
        $path = '/',
        $queryString = ''
    )

.. _components-routing-http-foundation:

Normally you can pass the values from the ``$_SERVER`` variable to populate the
:class:`Symfony\\Component\\Routing\\RequestContext`. But If you use the
:doc:`HttpFoundation </components/http_foundation/index>` component, you can use its
:class:`Symfony\\Component\\HttpFoundation\\Request` class to feed the
:class:`Symfony\\Component\\Routing\\RequestContext` in a shortcut::

    use Symfony\Component\HttpFoundation\Request;

    $context = new RequestContext();
    $context->fromRequest(Request::createFromGlobals());

Generate a URL
~~~~~~~~~~~~~~

While the :class:`Symfony\\Component\\Routing\\Matcher\\UrlMatcher` tries
to find a route that fits the given request you can also build a URL from
a certain route::

    use Symfony\Component\Routing\Generator\UrlGenerator;

    $routes = new RouteCollection();
    $routes->add('show_post', new Route('/show/{slug}'));

    $context = new RequestContext($_SERVER['REQUEST_URI']);

    $generator = new UrlGenerator($routes, $context);

    $url = $generator->generate('show_post', array(
        'slug' => 'my-blog-post',
    ));
    // /show/my-blog-post

.. note::

    If you have defined a scheme, an absolute URL is generated if the scheme
    of the current :class:`Symfony\\Component\\Routing\\RequestContext` does
    not match the requirement.

Load Routes from a File
~~~~~~~~~~~~~~~~~~~~~~~

You've already seen how you can easily add routes to a collection right inside
PHP. But you can also load routes from a number of different files.

The Routing component comes with a number of loader classes, each giving
you the ability to load a collection of route definitions from an external
file of some format.
Each loader expects a :class:`Symfony\\Component\\Config\\FileLocator` instance
as the constructor argument. You can use the :class:`Symfony\\Component\\Config\\FileLocator`
to define an array of paths in which the loader will look for the requested files.
If the file is found, the loader returns a :class:`Symfony\\Component\\Routing\\RouteCollection`.

If you're using the ``YamlFileLoader``, then route definitions look like this:

.. code-block:: yaml

    # routes.yml
    route1:
        path:     /foo
        defaults: { _controller: 'MyController::fooAction' }

    route2:
        path:     /foo/bar
        defaults: { _controller: 'MyController::foobarAction' }

To load this file, you can use the following code. This assumes that your
``routes.yml`` file is in the same directory as the below code::

    use Symfony\Component\Config\FileLocator;
    use Symfony\Component\Routing\Loader\YamlFileLoader;

    // look inside *this* directory
    $locator = new FileLocator(array(__DIR__));
    $loader = new YamlFileLoader($locator);
    $collection = $loader->load('routes.yml');

Besides :class:`Symfony\\Component\\Routing\\Loader\\YamlFileLoader` there are two
other loaders that work the same way:

* :class:`Symfony\\Component\\Routing\\Loader\\XmlFileLoader`
* :class:`Symfony\\Component\\Routing\\Loader\\PhpFileLoader`

If you use the :class:`Symfony\\Component\\Routing\\Loader\\PhpFileLoader` you
have to provide the name of a PHP file which returns a :class:`Symfony\\Component\\Routing\\RouteCollection`::

    // RouteProvider.php
    use Symfony\Component\Routing\RouteCollection;
    use Symfony\Component\Routing\Route;

    $collection = new RouteCollection();
    $collection->add(
        'route_name',
        new Route('/foo', array('controller' => 'ExampleController'))
    );
    // ...

    return $collection;

Routes as Closures
..................

There is also the :class:`Symfony\\Component\\Routing\\Loader\\ClosureLoader`, which
calls a closure and uses the result as a :class:`Symfony\\Component\\Routing\\RouteCollection`::

    use Symfony\Component\Routing\Loader\ClosureLoader;

    $closure = function() {
        return new RouteCollection();
    };

    $loader = new ClosureLoader();
    $collection = $loader->load($closure);

Routes as Annotations
.....................

Last but not least there are
:class:`Symfony\\Component\\Routing\\Loader\\AnnotationDirectoryLoader` and
:class:`Symfony\\Component\\Routing\\Loader\\AnnotationFileLoader` to load
route definitions from class annotations. The specific details are left
out here.

The all-in-one Router
~~~~~~~~~~~~~~~~~~~~~

The :class:`Symfony\\Component\\Routing\\Router` class is a all-in-one package
to quickly use the Routing component. The constructor expects a loader instance,
a path to the main route definition and some other settings::

    public function __construct(
        LoaderInterface $loader,
        $resource,
        array $options = array(),
        RequestContext $context = null,
        array $defaults = array()
    );

With the ``cache_dir`` option you can enable route caching (if you provide a
path) or disable caching (if it's set to ``null``). The caching is done
automatically in the background if you want to use it. A basic example of the
:class:`Symfony\\Component\\Routing\\Router` class would look like::

    $locator = new FileLocator(array(__DIR__));
    $requestContext = new RequestContext($_SERVER['REQUEST_URI']);

    $router = new Router(
        new YamlFileLoader($locator),
        'routes.yml',
        array('cache_dir' => __DIR__.'/cache'),
        $requestContext
    );
    $router->match('/foo/bar');

.. note::

    If you use caching, the Routing component will compile new classes which
    are saved in the ``cache_dir``. This means your script must have write
    permissions for that location.

.. _Packagist: https://packagist.org/packages/symfony/routing
