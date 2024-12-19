Streaming JSON
==============

Symfony is able of encoding PHP data structures to JSON streams and decoding
JSON streams back into PHP data structures.

To do so, it relies on the JsonStreamer component, which is designed for high
efficiency and can process large JSON data incrementally without needing to
load the entire content into memory.

.. tip::

    This component is ideal for handling APIs or interacting with third-party
    APIs. It transforms incoming JSON request payloads into PHP objects that
    your application can work with. Similarly, it converts processed PHP
    objects into a JSON stream for outgoing responses.

Installation
------------

To install the JsonStreamer component in applications that use
:ref:`Symfony Flex <symfony-flex>`, run this command:

.. code-block:: terminal

    $ composer require symfony/json-streamer

.. include:: /components/require_autoload.rst.inc

.. warning::

    The JsonStreamer component is :doc:`experimental </contributing/code/experimental>`
    and could be changed at any time without prior notice.

Encoding objects
----------------

Let's say that we have the following ``Cat`` class::

    // src/Dto/Cat.php
    namespace App\Dto;

    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;

    #[JsonStreamable]
    class Cat
    {
        public string $name;
        public string $age;
    }

.. warning::

    The JsonStreamer only works with PHP classes without constructor and
    composed by public properties only.

If you want to encode ``Cat`` objects to a JSON stream (e.g. to send them
via an API response), you can get the ``json_streamer.stream_writer`` service by using
the :class:`Symfony\\Component\\JsonStreamer\\StreamWriterInterface` parameter type
with the ``$jsonStreamWriter`` name, and use the :method:`Symfony\\Component\\JsonStreamer\\StreamWriterInterface::write`
method:

.. configuration-block::

    .. code-block:: php-symfony

        // src/Controller/CatController.php
        namespace App\Controller;

        use App\Dto\Cat;
        use App\Repository\CatRepository;
        use Symfony\Component\HttpFoundation\StreamedResponse;
        use Symfony\Component\JsonStreamer\StreamWriterInterface;
        use Symfony\Component\TypeInfo\Type;

        class CatController
        {
            public function retrieveCats(StreamWriterInterface $jsonStreamWriter, CatRepository $catRepository): StreamedResponse
            {
                $cats = $catRepository->findAll();
                $type = Type::list(Type::object(Cat::class));

                $json = $jsonStreamWriter->write($cats, $type);

                return new StreamedResponse($json);
            }
        }

    .. code-block:: php-standalone

        use App\Dto\Cat;
        use App\Repository\CatRepository;
        use Symfony\Component\HttpFoundation\StreamedResponse;
        use Symfony\Component\JsonStreamer\JsonStreamWriter;
        use Symfony\Component\TypeInfo\Type;

        // ...

        $jsonWriter = JsonStreamWriter::create();

        $cats = $catRepository->findAll();
        $type = Type::list(Type::object(Cat::class));

        $json = $jsonWriter->write($cats, $type);

        $response = new StreamedResponse($json);

        // ...

.. note::

    You can explicitly inject the ``json_streamer.stream_writer`` service by
    using the ``#[Target('json_streamer.stream_writer')]`` autowire attribute.

Decoding objects
----------------

Besides encoding objects to JSON, you can decode JSON to objects.

To do so, you can get the ``json_streamer.stream_reader`` service by using the
:class:`Symfony\\Component\\JsonStreamer\\StreamReaderInterface` parameter type
with the ``$jsonStreamReader`` name, and use the :method:`Symfony\\Component\\JsonStreamer\\StreamReaderInterface::read`
method:

.. configuration-block::

    .. code-block:: php-symfony

        // src/Service/TombolaService.php
        namespace App\Service;

        use App\Dto\Cat;
        use Symfony\Component\DependencyInjection\Attribute\Autowire;
        use Symfony\Component\JsonStreamer\StreamReaderInterface;
        use Symfony\Component\TypeInfo\Type;

        class TombolaService
        {
            private string $catsJsonFile;

            public function __construct(
                private StreamReaderInterface $jsonStreamReader,
                #[Autowire(param: 'kernel.root_dir')]
                string $rootDir,
            ) {
                $this->catsJsonFile = sprintf('%s/var/cats.json', $rootDir);
            }

            public function pickTheTenthCat(): ?Cat
            {
                $jsonResource = fopen($this->catsJsonFile, 'r');
                $type = Type::iterable(Type::object(Cat::class));

                /** @var iterable<Cat> $cats */
                $cats = $this->jsonStreamReader->read($jsonResource, $type);

                $i = 0;
                foreach ($cats as $cat) {
                    if ($i === 9) {
                        return $cat;
                    }

                    ++$i;
                }

                return null;
            }

            /**
             * @return list<string>
             */
            public function listEligibleCatNames(): array
            {
                $json = file_get_contents($this->catsJsonFile);
                $type = Type::iterable(Type::object(Cat::class));

                /** @var iterable<Cat> $cats */
                $cats = $this->jsonStreamReader->read($json, $type);

                return array_map(fn(Cat $cat) => $cat->name, iterator_to_array($cats));
            }
        }

    .. code-block:: php-standalone

        // src/Service/TombolaService.php
        namespace App\Service;

        use App\Dto\Cat;
        use Symfony\Component\JsonStreamer\JsonStreamReader;
        use Symfony\Component\JsonStreamer\StreamReaderInterface;
        use Symfony\Component\TypeInfo\Type;

        class TombolaService
        {
            private StreamReaderInterface $jsonStreamReader;
            private string $catsJsonFile;

            public function __construct(
                private string $catsJsonFile,
            ) {
                $this->jsonStreamReader = JsonStreamReader::create();
            }

            public function pickTheTenthCat(): ?Cat
            {
                $jsonResource = fopen($this->catsJsonFile, 'r');
                $type = Type::iterable(Type::object(Cat::class));

                /** @var iterable<Cat> $cats */
                $cats = $this->jsonStreamReader->read($jsonResource, $type);

                $i = 0;
                foreach ($cats as $cat) {
                    if ($i === 9) {
                        return $cat;
                    }

                    ++$i;
                }

                return null;
            }

            /**
             * @return list<string>
             */
            public function listEligibleCatNames(): array
            {
                $json = file_get_contents($this->catsJsonFile);
                $type = Type::iterable(Type::object(Cat::class));

                /** @var iterable<Cat> $cats */
                $cats = $this->jsonStreamReader->read($json, $type);

                return array_map(fn(Cat $cat) => $cat->name, iterator_to_array($cats));
            }
        }

.. note::

    You can explicitly inject the ``json_streamer.stream_reader`` service by
    using the ``#[Target('json_streamer.stream_reader')]`` autowire attribute.

The upper code demonstrates two different approaches to decoding JSON data
using the JsonStreamer:

* decoding from a stream (``pickTheTenthCat``)
* decoding from a string (``listEligibleCatNames``).

Both methods work with the same JSON data but differ in memory usage and
speed optimization.


Decoding from a stream
~~~~~~~~~~~~~~~~~~~~~~

In the ``pickTheTenthCat`` method, the JSON data is read as a stream using
:phpfunction:`fopen`. Streams are useful when working with large files
because the data is processed incrementally rather than loading the entire
file into memory.

To improve memory efficiency, the JsonStreamer creates `ghost objects`_
instead of fully instantiating objects. Ghosts objects are lightweight
placeholders that represent the objects but don't fully load their data
into memory until it's needed. This approach reduces memory usage, especially
for large datasets.

* Advantage: Efficient memory usage, suitable for very large JSON files.
* Disadvantage: Slightly slower than decoding a full string because data is
  loaded on-demand.

Decoding from a string
~~~~~~~~~~~~~~~~~~~~~~

In the ``listEligibleCatNames`` method, the entire JSON file is read into
a string using :phpfunction:`file_get_contents`. This string is then passed
to the decoder, which fully instantiates all the objects in the JSON data
upfront.

This approach is faster because all the objects are created immediately,
making subsequent operations on the data quicker. However, it uses more
memory since the entire file content and all objects are loaded at once.

* Advantage: Faster processing, suitable for small to medium-sized JSON files.
* Disadvantage: Higher memory usage, not ideal for large JSON files.

.. tip::

    Prefer stream decoding when working with large JSON files to conserve
    memory.

    Prefer string decoding instead when performance is more critical and the
    JSON file size is manageable.

Enabling PHPDoc reading
-----------------------

The JsonStreamer component can be able to process advanced PHPDoc type
definitions, such as generics, and read/generate JSON for complex PHP
objects.

For example, let's consider this ``Shelter`` class that defines a generic
``TAnimal`` type, which can be a ``Cat`` or a ``Dog``::

    // src/Dto/Shelter.php
    namespace App\Dto;

    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;

    /**
     * @template TAnimal of Cat|Dog
     */
    #[JsonStreamable]
    class Shelter
    {
        /**
         * @var list<TAnimal>
         */
        public array $animals;
    }


To enable PHPDoc interpretation, run the following command:

.. code-block:: terminal

    $ composer require phpstan/phpdoc-parser

Then, when encoding/decoding an instance of the ``Shelter`` class, you can
specify the concrete type information, and the JsonStreamer will deal with the
correct JSON structure::

    use App\Dto\Cat;
    use App\Dto\Shelter;
    use Symfony\Component\TypeInfo\Type;

    $json = <<<JSON
    {
      "animals": [
        {"name": "Eva", "age": 29},
        {...}
      ]
    }
    JSON;

    // maps the TAnimal template in Shelter to the Cat concrete type
    $type = Type::generic(Type::object(Shelter::class), Type::object(Cat::class));

    $catShelter = $jsonStreamReader->read($json, $type); // will be populated with Cat instances

Configuring encoding/decoding
-----------------------------

While it's not recommended to change to object shape and values during
encoding and decoding, it is sometimes unavoidable.

Configuring the encoded name
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

It is possible to configure the JSON key associated to a property thanks to
the :class:`Symfony\\Component\\JsonStreamer\\Attribute\\StreamedName`
attribute::

    // src/Dto/Duck.php
    namespace App\Dto;

    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;
    use Symfony\Component\JsonStreamer\Attribute\StreamedName;

    #[JsonStreamable]
    class Duck
    {
        #[StreamedName('@id')]
        public string $id;
    }

By doing so, the ``Duck::$id`` property will be mapped to the ``@id`` JSON key::

    use App\Dto\Duck;
    use Symfony\Component\TypeInfo\Type;

    // ...

    $duck = new Duck();
    $duck->id = '/ducks/daffy';

    echo (string) $jsonStreamWriter->write($duck, Type::object(Duck::class));

    // This will output:
    // {
    //   "@id": "/ducks/daffy"
    // }

Configuring the encoded value
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

If you need to manipulate the value related to a property during the encoding
process, you can use the :class:`Symfony\\Component\\JsonStreamer\\Attribute\\ValueTransformer`
attribute. Its ``nativeToStream`` property takes a callable, or a :ref:`value transformer service id <json-streamer-transform-with-services>`.

When a callable is specified, it must either be a public static method or
non-anonymous function with the following signature::

    $transformer = function (mixed $data, array $options = []): mixed { /* ... */ };

Then, you just have to specify the function identifier in the attribute::

    // src/Dto/Duck.php
    namespace App\Dto;

    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;
    use Symfony\Component\JsonStreamer\Attribute\ValueTransformer;

    #[JsonStreamable]
    class Duck
    {
        #[ValueTransformer(nativeToStream: 'strtoupper')]
        public string $name;

        #[ValueTransformer(nativeToStream: [self::class, 'formatHeight'])]
        public int $height;

        public static function formatHeight(int $value, array $options = []): string
        {
            return sprintf('%.2fcm', $value / 100);
        }
    }

For example, by configuring the ``Duck`` class like above, the ``name`` and
``height`` values will be transformed during encoding::

    use App\Dto\Duck;
    use Symfony\Component\TypeInfo\Type;

    // ...

    $duck = new Duck();
    $duck->name = 'daffy';
    $duck->height = 5083;

    echo (string) $jsonStreamWriter->write($duck, Type::object(Duck::class));

    // This will output:
    // {
    //   "name": "DAFFY",
    //   "height": "50.83cm"
    // }

Configuring the decoded value
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can as well manipulate the value related to a property during decoding
using the :class:`Symfony\\Component\\JsonStreamer\\Attribute\\ValueTransformer`
attribute. Its ``streamToNative`` property can take either a callable or a :ref:`value transformer service id <json-streamer-transform-with-services>`.

When a callable is specified, it must either be a public static method or
a non-anonymous function with the following signature::

    $valueTransformer = function (mixed $data, array $options = []): mixed { /* ... */ };

You can specify a function identifier in the attribute as follows::

    // src/Dto/Duck.php
    namespace App\Dto;

    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;
    use Symfony\Component\JsonStreamer\Attribute\ValueTransformer;

    #[JsonStreamable]
    class Duck
    {
        #[ValueTransformer(streamToNative: [self::class, 'retrieveFirstName'])]
        public string $firstName;

        #[ValueTransformer(streamToNative: [self::class, 'retrieveLastName'])]
        public string $lastName;

        public static function retrieveFirstName(string $normalized, array $options = []): string
        {
            return explode(' ', $normalized)[0];
        }

        public static function retrieveLastName(string $normalized, array $options = []): string
        {
            return explode(' ', $normalized)[1];
        }
    }

For instance, the above configuration for the ``Duck`` class will transform
the `name` property from the input JSON during decoding::

    use App\Dto\Duck;
    use Symfony\Component\TypeInfo\Type;

    // ...

    $duck = $jsonStreamReader->read(
        '{"name": "Daffy Duck"}',
        Type::object(Duck::class),
    );

    // The $duck variable will contain:
    // object(Duck)#1 (1) {
    //   ["firstName"] => string(5) "Daffy"
    //   ["lastName"] => string(4) "Duck"
    // }

.. _json-streamer-transform-with-services:

Transform value using services
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When static methods or functions are not enough, you can transform the value
thanks to a value transformer service.

To do so, create a service implementing the :class:`Symfony\\Component\\JsonStreamer\\ValueTransformer\\ValueTransformerInterface`::

    // src/Transformer/DogUrlTransformer.php
    namespace App\Transformer;

    use Symfony\Component\JsonStreamer\ValueTransformer\ValueTransformerInterface;
    use Symfony\Component\Routing\Generator\UrlGeneratorInterface;
    use Symfony\Component\TypeInfo\Type;

    class DogUrlTransformer implements ValueTransformerInterface
    {
        public function __construct(
            private UrlGeneratorInterface $urlGenerator,
        ) {
        }

        public function transform(mixed $value, array $options = []): string
        {
            if (!is_int($value)) {
                throw new \InvalidArgumentException(sprintf('The value must be "int", "%s" given.', get_debug_type($value)));
            }

            return $this->urlGenerator->generate('show_dog', ['id' => $value]);
        }

        public static function getStreamValueType(): Type
        {
            return Type::string();
        }
    }

.. note::

    The ``getStreamValueType`` method should return the type of what the value
    will be in the JSON stream.

And then, configure the :class:`Symfony\\Component\\JsonStreamer\\Attribute\\ValueTransformer`
attribute to use that service::

    // src/Dto/Dog.php
    namespace App\Dto;

    use App\Transformer\DogUrlTransformer;
    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;
    use Symfony\Component\JsonStreamer\Attribute\StreamedName;
    use Symfony\Component\JsonStreamer\Attribute\ValueTransformer;

    #[JsonStreamable]
    class Dog
    {
        #[StreamedName('url')]
        #[ValueTransformer(nativeToStream: DogUrlTransformer::class)]
        public int $id;
    }

.. tip::

    The value transformers will be intensively called during the
    decoding and encoding. So be sure to keep them as fast as possible
    (avoid calling external APIs or the database for example).

Configure keys and values dynamically
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The JsonStreamer leverages services implementing the :class:`Symfony\\Component\\JsonStreamer\\Mapping\\PropertyMetadataLoaderInterface`
to determine the shape and values of objects during encoding/decoding.

These services are highly flexible and can be decorated to handle dynamic
configurations, offering much greater power compared to using attributes::

    namespace App\Streamer\SensitivePropertyMetadataLoader;

    use App\Dto\SensitiveInterface;
    use App\Streamer\ValueTransformer\EncryptorValueTransformer;
    use Symfony\Component\DependencyInjection\Attribute\AsDecorator;
    use Symfony\Component\DependencyInjection\Attribute\AutowireDecorated;
    use Symfony\Component\JsonStreamer\Mapping\PropertyMetadata;
    use Symfony\Component\JsonStreamer\Mapping\PropertyMetadataLoaderInterface;
    use Symfony\Component\TypeInfo\Type;

    #[AsDecorator('json_streamer.write.property_metadata_loader')]
    class SensitivePropertyMetadataLoader implements PropertyMetadataLoaderInterface
    {
        public function __construct(
            #[AutowireDecorated]
            private PropertyMetadataLoaderInterface $decorated,
        ) {
        }

        public function load(string $className, array $options = [], array $context = []): array
        {
            $propertyMetadataMap = $this->decorated->load($className, $options, $context);

            if (!is_a($className, SensitiveInterface::class, true)) {
                return $propertyMetadataMap;
            }

            // you can configure value transformers
            foreach ($propertyMetadataMap as $jsonKey => $metadata) {
                if (in_array($metadata->getName(), $className::getPropertiesToEncrypt(), true)) {
                    $propertyMetadataMap[$jsonKey] = $metadata
                        ->withType(Type::string())
                        ->withAdditionalNativeToStreamValueTransformer(EncryptorValueTransformer::class);
                }
            }

            // you can remove existing properties
            foreach ($propertyMetadataMap as $jsonKey => $metadata) {
                if (in_array($metadata->getName(), $className::getPropertiesToRemove(), true)) {
                    unset($propertyMetadataMap[$jsonKey]);
                }
            }

            // you can rename JSON keys
            foreach ($propertyMetadataMap as $jsonKey => $metadata) {
                $propertyMetadataMap[md5($jsonKey)] = $propertyMetadataMap[$jsonKey];
                unset($propertyMetadataMap[$jsonKey]);
            }

            // you can add virtual properties
            $propertyMetadataMap['is_sensitive'] = new PropertyMetadata(
                name: 'theNameWontBeUsed',
                type: Type::bool(),
                nativeToStreamValueTransformers: [fn() => true],
            );

            return $propertyMetadataMap;
        }
    }

However, this flexibility comes with complexity. Decorating property metadata
loaders requires a deep understanding of the system.

For most use cases, the attributes approach is sufficient, and the dynamic
capabilities of property metadata loaders should be reserved for scenarios
where their additional power is genuinely necessary.

Marking objects as streamable
-----------------------------

The ``JsonStreamable`` attribute is used to mark a class as streamable.
While this attribute is not mandatory, it is highly recommended because it
plays a crucial role during the cache warm-up process by generating the
files necessary for encoding and decoding operations, and thereby improving
performance.

It includes two properties: ``asObject`` and ``asList``. These properties
define the structure in which the marked class should be prepared during the
cache warm-up process::

    use Symfony\Component\JsonStreamer\Attribute\JsonStreamable;

    #[JsonStreamable(asObject: true, asList: true)]
    class StreamableData
    {
        // ...
    }

.. _ghost objects: https://en.wikipedia.org/wiki/Lazy_loading#Ghost
