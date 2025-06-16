The JsonPath Component
======================

.. versionadded:: 7.3

    The JsonPath component was introduced in Symfony 7.3 as an
    :doc:`experimental feature </contributing/code/experimental>`.

The JsonPath component provides a powerful way to query and extract data from
JSON structures. It implements the `RFC 9535 - JSONPath`_
standard, allowing you to navigate complex JSON data with ease.

Just as the :doc:`DomCrawler component </components/dom_crawler>` allows you to navigate and query HTML/XML documents
using XPath, the JsonPath component provides a similar experience to traverse and search JSON structures
using JSONPath expressions. The component also offers an abstraction layer for data extraction.

Installation
------------

You can install the component in your project using Composer:

.. code-block:: terminal

    $ composer require symfony/json-path

.. include:: /components/require_autoload.rst.inc

Usage
-----

To start querying a JSON document, first create a :class:`Symfony\\Component\\JsonPath\\JsonCrawler`
object from a JSON string. For the following examples, we'll use this sample
"bookstore" JSON data::

    use Symfony\Component\JsonPath\JsonCrawler;

    $json = <<<'JSON'
    {
        "store": {
            "book": [
                {
                    "category": "reference",
                    "author": "Nigel Rees",
                    "title": "Sayings of the Century",
                    "price": 8.95
                },
                {
                    "category": "fiction",
                    "author": "Evelyn Waugh",
                    "title": "Sword of Honour",
                    "price": 12.99
                },
                {
                    "category": "fiction",
                    "author": "Herman Melville",
                    "title": "Moby Dick",
                    "isbn": "0-553-21311-3",
                    "price": 8.99
                },
                {
                    "category": "fiction",
                    "author": "John Ronald Reuel Tolkien",
                    "title": "The Lord of the Rings",
                    "isbn": "0-395-19395-8",
                    "price": 22.99
                }
            ],
            "bicycle": {
                "color": "red",
                "price": 399
            }
        }
    }
    JSON;

    $crawler = new JsonCrawler($json);

Once you have the crawler instance, use its :method:`Symfony\\Component\\JsonPath\\JsonCrawler::find` method to start querying
the data. This method always returns an array of matching values.

Querying with Expressions
-------------------------

The primary way to query the JSON is by passing a JSONPath expression string
to the :method:`Symfony\\Component\\JsonPath\\JsonCrawler::find` method.

Accessing a Specific Property
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Use dot-notation for object keys and square brackets for array indices. The root
of the document is represented by ``$``::

    // Get the title of the first book in the store
    $titles = $crawler->find('$.store.book[0].title');

    // $titles is ['Sayings of the Century']

While dot-notation is common, JSONPath offers alternative syntaxes which can be more flexible. For instance, bracket notation (`['...']`) is required if a key contains spaces or special characters::

    // This is equivalent to the previous example and works with special keys
    $titles = $crawler->find('$["store"]["book"][0]["title"]');

    // You can also build the query programmatically for more complex scenarios
    use Symfony\Component\JsonPath\JsonPath;

    $path = (new JsonPath())->key('store')->key('book')->first()->key('title');
    $titles = $crawler->find($path);

Searching with the Descendant Operator
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The descendant operator (``..``) recursively searches for a given key, allowing
you to find values without specifying the full path::

    // Get all authors from anywhere in the document
    $authors = $crawler->find('$..author');

    // $authors is ['Nigel Rees', 'Evelyn Waugh', 'Herman Melville', 'John Ronald Reuel Tolkien']

Filtering Results
~~~~~~~~~~~~~~~~~

JSONPath includes a powerful filter syntax (``?(<expression>)``) to select items
based on a condition. The current item within the filter is referenced by ``@``::

    // Get all books with a price less than 10
    $cheapBooks = $crawler->find('$.store.book[?(@.price < 10)]');

    /*
    $cheapBooks contains two book objects:
    the one by "Nigel Rees" and the one by "Herman Melville"
    */

Building Queries Programmatically
---------------------------------

For more dynamic or complex query building, you can use the fluent API provided
by the :class:`Symfony\\Component\\JsonPath\\JsonPath` class. This allows you
to construct a query object step-by-step. The ``JsonPath`` object can then be
passed to the crawler's :method:`Symfony\\Component\\JsonPath\\JsonCrawler::find` method.

The main advantage of the programmatic builder is that it automatically handles
the correct escaping of keys and values, preventing syntax errors::

    use Symfony\Component\JsonPath\JsonPath;

    $path = (new JsonPath())
        ->key('store')       // Selects the 'store' key
        ->key('book')        // Then the 'book' key
        ->index(1);          // Then the item at index 1 (the second book)

    // The created $path object is equivalent to the string '$["store"]["book"][1]'
    $book = $crawler->find($path);

    // $book contains the book object for "Sword of Honour"

The :class:`Symfony\\Component\\JsonPath\\JsonPath` class provides several methods to build your query:

* :method:`Symfony\\Component\\JsonPath\\JsonPath::key`
    Adds a key selector. The key name will be properly escaped::

        // Creates the path '$["key\"with\"quotes"]'
        $path = (new JsonPath())->key('key"with"quotes');

* :method:`Symfony\\Component\\JsonPath\\JsonPath::deepScan`
    Adds the descendant operator ``..`` to perform a recursive search from the
    current point in the path::

        // Get all prices in the store: '$["store"]..["price"]'
        $path = (new JsonPath())->key('store')->deepScan()->key('price');

* :method:`Symfony\\Component\\JsonPath\\JsonPath::all`
    Adds the wildcard operator ``[*]`` to select all items in an array or object::

        // Creates the path '$["store"]["book"][*]'
        $path = (new JsonPath())->key('store')->key('book')->all();

* :method:`Symfony\\Component\\JsonPath\\JsonPath::index`
    Adds an array index selector.

* :method:`Symfony\\Component\\JsonPath\\JsonPath::first` / :method:`Symfony\\Component\\JsonPath\\JsonPath::last`
    Shortcuts for ``index(0)`` and ``index(-1)`` respectively::

        // Get the last book: '$["store"]["book"][-1]'
        $path = (new JsonPath())->key('store')->key('book')->last();

* :method:`Symfony\\Component\\JsonPath\\JsonPath::slice`
    Adds an array slice selector ``[start:end:step]``::

        // Get books from index 1 up to (but not including) index 3
        // Creates the path '$["store"]["book"][1:3]'
        $path = (new JsonPath())->key('store')->key('book')->slice(1, 3);

        // Get every second book from the first four books
        // Creates the path '$["store"]["book"][0:4:2]'
        $path = (new JsonPath())->key('store')->key('book')->slice(0, 4, 2);

* :method:`Symfony\\Component\\JsonPath\\JsonPath::filter`
    Adds a filter expression. The expression string is the part that goes inside
    the ``?()`` syntax::

        // Get expensive books: '$["store"]["book"][?(@.price > 20)]'
        $path = (new JsonPath())
            ->key('store')
            ->key('book')
            ->filter('@.price > 20');

Advanced Querying
-----------------

For a complete overview of advanced operators like wildcards and functions within
filters, please refer to the `Querying with Expressions`_ section above. All these
features are supported and can be combined with the programmatic builder where
appropriate (e.g., inside a ``filter()`` expression).

Error Handling
--------------

The component will throw specific exceptions for invalid input or queries:

* :class:`Symfony\\Component\\JsonPath\\Exception\\InvalidArgumentException`: Thrown if the input to the ``JsonCrawler`` constructor is not a valid JSON string.
* :class:`Symfony\\Component\\JsonPath\\Exception\\InvalidJsonStringInputException`: Thrown during a ``find()`` call if the JSON string is malformed (e.g., syntax error).
* :class:`Symfony\\Component\\JsonPath\\Exception\\JsonCrawlerException`: Thrown for errors within the JsonPath expression itself, such as using an unknown function::

    use Symfony\Component\JsonPath\Exception\InvalidJsonStringInputException;
    use Symfony\Component\JsonPath\Exception\JsonCrawlerException;

    try {
        // Example of malformed JSON
        $crawler = new JsonCrawler('{"store": }');
        $crawler->find('$..*');
    } catch (InvalidJsonStringInputException $e) {
        // ... handle error
    }

    try {
        // Example of an invalid query
        $crawler->find('$.store.book[?unknown_function(@.price)]');
    } catch (JsonCrawlerException $e) {
        // ... handle error
    }

.. _`RFC 9535 - JSONPath`: https://datatracker.ietf.org/doc/html/rfc9535
