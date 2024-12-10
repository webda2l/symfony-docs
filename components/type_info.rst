The TypeInfo Component
======================

The TypeInfo component extracts type information from PHP elements like properties,
arguments and return types.

This component provides:

* A powerful ``Type`` definition that can handle unions, intersections, and generics
  (and can be extended to support more types in the future);
* A way to get types from PHP elements such as properties, method arguments,
  return types, and raw strings.

Installation
------------

.. code-block:: terminal

    $ composer require symfony/type-info

.. include:: /components/require_autoload.rst.inc

Usage
-----

This component gives you a :class:`Symfony\\Component\\TypeInfo\\Type` object that
represents the PHP type of anything you built or asked to resolve.

There are two ways to use this component. First one is to create a type manually thanks
to the :class:`Symfony\\Component\\TypeInfo\\Type` static methods as following::

    use Symfony\Component\TypeInfo\Type;

    Type::int();
    Type::nullable(Type::string());
    Type::generic(Type::object(Collection::class), Type::int());
    Type::list(Type::bool());
    Type::intersection(Type::object(\Stringable::class), Type::object(\Iterator::class));

    // Many others are available and can be
    // found in Symfony\Component\TypeInfo\TypeFactoryTrait

Resolvers
~~~~~~~~~

The second way of using the component is to use ``TypeInfo`` to resolve a type
based on reflection or a simple string, this is aimed towards libraries that wants to
describe a class or anything that has a type easily::

    use Symfony\Component\TypeInfo\Type;
    use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

    class Dummy
    {
        public function __construct(
            public int $id,
        ) {
        }
    }

    // Instantiate a new resolver
    $typeResolver = TypeResolver::create();

    // Then resolve types for any subject
    $typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'id')); // returns an "int" Type instance
    $typeResolver->resolve('bool'); // returns a "bool" Type instance

    // Types can be instantiated thanks to static factories
    $type = Type::list(Type::nullable(Type::bool()));

    // Type instances have several helper methods

    // for collections, it returns the type of the item used as the key;
    // in this example, the collection is a list, so it returns an "int" Type instance
    $keyType = $type->getCollectionKeyType();

    // you can chain the utility methods (e.g. to introspect the values of the collection)
    // the following code will return true
    $isValueNullable = $type->getCollectionValueType()->isNullable();

Each of these calls will return you a ``Type`` instance that corresponds to the
static method used. You can also resolve types from a string (as shown in the
``bool`` parameter of the previous example)

PHPDoc parsing
~~~~~~~~~~~~~~

But most times you won't have clean typed properties or you want a more precise type
thank to advanced PHPDoc, to do that you would want a string resolver based on that PHPDoc.
First you will require ``phpstan/phpdoc-parser`` package from composer to support string
revolving. Then you would do as following::

    use Symfony\Component\TypeInfo\TypeResolver\TypeResolver;

    class Dummy
    {
        public function __construct(
            public int $id,
            /** @var string[] $tags */
            public array $tags,
        ) {
        }
    }

    $typeResolver = TypeResolver::create();
    $typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'id')); // returns an "int" Type
    $typeResolver->resolve(new \ReflectionProperty(Dummy::class, 'id')); // returns a collection with "int" as key and "string" as values Type

Advanced usages
~~~~~~~~~~~~~~~

There is many methods to manipulate and check types depending on your needs within the TypeInfo components.

If you need a check a simple Type::

    // You need to check if a Type
    $type = Type::int(); // with a simple int type
    // You can check if a given type comply with a given identifier
    $type->isIdentifiedBy(TypeIdentifier::INT); // true
    $type->isIdentifiedBy(TypeIdentifier::STRING); // false

    $type = Type::union(Type::string(), Type::int()); // with an union of int and string types
    // You can now see that the second check will pass to true since we have an union with a string type
    $type->isIdentifiedBy(TypeIdentifier::INT); // true
    $type->isIdentifiedBy(TypeIdentifier::STRING); // true

    class DummyParent {}
    class Dummy extends DummyParent implements DummyInterface {}
    $type = Type::object(Dummy::class); // with an object Type
    // You can check is the Type is an object, or even if it's a given class
    $type->isIdentifiedBy(TypeIdentifier::OBJECT); // true
    $type->isIdentifiedBy(Dummy::class); // true
    // Or inherits/implements something
    $type->isIdentifiedBy(DummyParent::class); // true
    $type->isIdentifiedBy(DummyInterface::class); // true

Sometimes you want to check for more than one thing at a time so a callable may be better to check everything::

    class Foo
    {
        private int $integer;
        private string $string;
        private ?float $float;
    }

    $reflClass = new \ReflectionClass(Foo::class);

    $resolver = TypeResolver::create();
    $integerType = $resolver->resolve($reflClass->getProperty('integer'));
    $stringType = $resolver->resolve($reflClass->getProperty('string'));
    $floatType = $resolver->resolve($reflClass->getProperty('float'));

    // your callable to check whatever you need
    // here we want to validate a given type is a non nullable number
    $isNonNullableNumber = function (Type $type): bool {
        if ($type->isNullable()) {
            return false;
        }

        if ($type->isIdentifiedBy(TypeIdentifier::INT) || $type->isIdentifiedBy(TypeIdentifier::FLOAT)) {
            return true;
        }

        return false;
    };

    $integerType->isSatisfiedBy($isNonNullableNumber); // true
    $stringType->isSatisfiedBy($isNonNullableNumber); // false
    $floatType->isSatisfiedBy($isNonNullableNumber); // false
