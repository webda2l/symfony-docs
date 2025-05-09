Object Mapper
=============

Symfony provides a mapper component to transform a given object into another one,
simplifying tasks like converting DTOs (Data Transfer Objects) to entities or
vice-versa. It may also be useful when decoupling API input/output from internal
models helping with legacy code or hexagonal architectures implementations.

.. versionadded:: 7.2

    The ObjectMapper component is experimental. Experimental features are not
    covered by the backward compatibility promise.

Installation
------------

Run this command to install the ``object-mapper`` component before using it:

.. code-block:: terminal

    $ composer require symfony/object-mapper

Using the ObjectMapper
----------------------

Once installed, the object mapper service can be injected into any service or
controller where you need it:

::

    // src/Controller/UserController.php
    namespace App\Controller;

    use App\Dto\UserInput;
    use App\Entity\User;
    use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\ObjectMapper\ObjectMapperInterface;

    class UserController extends AbstractController
    {
        public function updateUser(UserInput $userInput, ObjectMapperInterface $objectMapper): Response
        {
            $user = new User();
            // Map properties from UserInput to User
            $objectMapper->map($userInput, $user);

            // ... persist $user and return response
            return new Response('User updated!');
        }
    }

Basic Mapping
-------------

The core functionality is provided by the ``map()`` method. It takes a source
object and maps its properties onto a target. The target can be either a
class name (a new instance will be created) or an existing object instance.

**Mapping to a New Object:**

Provide the target class name as the second argument:

::

    use App\Dto\ProductInput;
    use App\Entity\Product;
    use Symfony\Component\ObjectMapper\ObjectMapper;

    $productInput = new ProductInput();
    $productInput->name = 'Wireless Mouse';
    $productInput->sku = 'WM-1024';

    $mapper = new ObjectMapper();
    // Creates a new Product instance and maps properties from $productInput
    $product = $mapper->map($productInput, Product::class);

    // $product is now an instance of Product
    // with $product->name = 'Wireless Mouse' and $product->sku = 'WM-1024'

**Mapping to an Existing Object:**

Provide an existing object instance as the second argument to update it:

::

    use App\Dto\ProductUpdateInput;
    use App\Entity\Product;
    use Symfony\Component\ObjectMapper\ObjectMapper;

    $product = $productRepository->find(1);

    $updateInput = new ProductUpdateInput();
    $updateInput->price = 99.99;

    $mapper = new ObjectMapper();
    // Updates the existing $product instance
    $mapper->map($updateInput, $product);

    // $product->price is now 99.99

**Mapping from ``stdClass``:**

The source object can also be an instance of ``stdClass``:

::

    use App\Entity\Product;
    use Symfony\Component\ObjectMapper\ObjectMapper;

    $productData = new \stdClass();
    $productData->name = 'Keyboard';
    $productData->sku = 'KB-001';

    $mapper = new ObjectMapper();
    $product = $mapper->map($productData, Product::class);

    // $product is an instance of Product with properties mapped from $productData

Configuring Mapping with Attributes
-----------------------------------

The ObjectMapper uses PHP attributes to configure the mapping process. The
primary attribute is :class:`Symfony\\Component\\ObjectMapper\\Attribute\\Map`.

**Defining the Default Target Class:**

Apply ``#[Map]`` to the source class to define its default mapping target:

::

    // src/Dto/ProductInput.php
    namespace App\Dto;

    use App\Entity\Product;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: Product::class)]
    class ProductInput
    {
        public string $name = '';
        public string $sku = '';
    }

    // Now you can call map() without the second argument if ProductInput is the source:
    $mapper = new ObjectMapper();
    $product = $mapper->map($productInput); // Maps to Product automatically

**Configuring Property Mapping:**

Apply ``#[Map]`` to properties to customize their mapping behavior.

* ``target``: Specifies the name of the property in the target object.
* ``source``: Specifies the name of the property in the source object (useful
    when mapping is defined on the target, see below).
* ``if``: Defines a condition for mapping the property.
* ``transform``: Applies a transformation to the value before mapping.

::

    // src/Dto/OrderInput.php
    namespace App\Dto;

    use App\Entity\Order;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: Order::class)]
    class OrderInput
    {
        // Map 'customerEmail' from source to 'email' in target
        #[Map(target: 'email')]
        public string $customerEmail = '';

        // Do not map this property at all
        #[Map(if: false)]
        public string $internalNotes = '';

        // Only map 'discountCode' if it's a non-empty string
        #[Map(if: 'strlen')] // Uses the built-in strlen function as a condition
        public ?string $discountCode = null;
    }

By default, if a property exists in the source but not in the target, it's
ignored. If a property exists in both and has no ``#[Map]`` attribute, it's
mapped directly if the names match.

**Conditional Mapping with Services:**

For complex conditions, you can use a dedicated service implementing
:class:`Symfony\\Component\\ObjectMapper\\ConditionCallableInterface`. The service
name (its class name by default) is passed to the ``if`` parameter:

::

    // src/ObjectMapper/IsShippableCondition.php
    namespace App\ObjectMapper;

    use App\Dto\OrderInput;
    use App\Entity\Order; // Target type hint
    use Symfony\Component\ObjectMapper\ConditionCallableInterface;

    /**
     * @implements ConditionCallableInterface<OrderInput, Order>
     */
    final class IsShippableCondition implements ConditionCallableInterface
    {
        public function __invoke(mixed $value, object $source, ?object $target): bool
        {
            // Example: Only map shipping address if order total is above 50
            return $source->total > 50;
        }
    }

::

    // src/Dto/OrderInput.php
    namespace App\Dto;

    use App\Entity\Order;
    use App\ObjectMapper\IsShippableCondition;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: Order::class)]
    class OrderInput
    {
        public float $total = 0.0;

        #[Map(if: IsShippableCondition::class)]
        public ?string $shippingAddress = null;
    }

For this to work, ``IsShippableCondition`` must be registered as a service.

.. _object_mapper-conditional-property-target:

**Conditional Property Mapping based on Target:**

When a source class can be mapped to multiple target classes, you might want
to map a specific property only when mapping to a particular target. You can
achieve this using the :class:`Symfony\\Component\\ObjectMapper\\Condition\\TargetClass`
condition within the ``if`` parameter on a property's ``#[Map]`` attribute.

This is useful for creating different representations (e.g., public vs. admin)
from the same source object, offering an alternative to serialization groups.

::

    // src/Entity/User.php
    namespace App\Entity;

    use App\Dto\AdminUserProfile;
    use App\Dto\PublicUserProfile;
    use Symfony\Component\ObjectMapper\Attribute\Map;
    use Symfony\Component\ObjectMapper\Condition\TargetClass;

    // This User entity can be mapped to two different DTOs
    #[Map(target: PublicUserProfile::class)]
    #[Map(target: AdminUserProfile::class)]
    class User
    {
        // Map 'lastLoginIp' to 'ipAddress' ONLY when the target is AdminUserProfile
        #[Map(target: 'ipAddress', if: new TargetClass(AdminUserProfile::class))]
        public ?string $lastLoginIp = '192.168.1.100';

        // Map 'registrationDate' to 'memberSince' for both targets
        #[Map(target: 'memberSince')]
        public \DateTimeImmutable $registrationDate;

        public function __construct() {
            $this->registrationDate = new \DateTimeImmutable();
        }
    }

    // src/Dto/PublicUserProfile.php
    namespace App\Dto;
    class PublicUserProfile {
        public \DateTimeImmutable $memberSince;
        // No $ipAddress property here
    }

    // src/Dto/AdminUserProfile.php
    namespace App\Dto;
    class AdminUserProfile {
        public \DateTimeImmutable $memberSince;
        public ?string $ipAddress = null; // Mapped from lastLoginIp
    }

    // Usage:
    $user = new User();
    $mapper = new ObjectMapper();

    $publicProfile = $mapper->map($user, PublicUserProfile::class);
    // No IP address available

    $adminProfile = $mapper->map($user, AdminUserProfile::class);
    // $adminProfile->ipAddress = '192.168.1.100'

Transforming Values
-------------------

Use the ``transform`` option within ``#[Map]`` to modify a value before it's
assigned to the target property. The value can be a callable (like a built-in
PHP function name, a static method, or an anonymous function) or a service
implementing :class:`Symfony\\Component\\ObjectMapper\\TransformCallableInterface`.

**Using Callables:**

::

    // src/Dto/ProductInput.php
    namespace App\Dto;

    use App\Entity\Product;
    use App\Util\PriceFormatter;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: Product::class)]
    class ProductInput
    {
        // Use a static method from another class for formatting
        #[Map(target: 'displayPrice', transform: [PriceFormatter::class, 'format'])]
        public float $price = 0.0;

        // Can also use simple built-ins
        #[Map(transform: 'intval')]
        public string $stockLevel = '100';
    }

::

    // src/Util/PriceFormatter.php
    namespace App\Util;
    class PriceFormatter {
        public static function format(float $value, object $source): string {
            return number_format($value, 2, '.', '');
        }
    }

**Using Transformer Services:**

Similar to conditions, complex transformations can be encapsulated in services
implementing :class:`Symfony\\Component\\ObjectMapper\\TransformCallableInterface`.

::

    // src/ObjectMapper/FullNameTransformer.php
    namespace App\ObjectMapper;

    use App\Dto\UserInput;
    use App\Entity\User;
    use Symfony\Component\ObjectMapper\TransformCallableInterface;

    /**
     * @implements TransformCallableInterface<UserInput, User>
     */
    final class FullNameTransformer implements TransformCallableInterface
    {
        public function __invoke(mixed $value, object $source, ?object $target): mixed
        {
            return trim($source->firstName . ' ' . $source->lastName);
        }
    }

::

    // src/Dto/UserInput.php
    namespace App\Dto;

    use App\Entity\User;
    use App\ObjectMapper\FullNameTransformer;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: User::class)]
    class UserInput
    {
        // This property's value will be generated by the transformer
        #[Map(target: 'fullName', transform: FullNameTransformer::class)]
        public string $firstName = '';

        public string $lastName = '';
    }

**Class-Level Transformation:**

You can apply a ``transform`` callable at the class level. This callable is
executed *after* the target object is initially created (if the target is a
class name we use ``newInstanceWithoutConstructor``) but *before* properties
are mapped. Its responsibility is often to return a correctly initialized
instance, potentially replacing the one created by the mapper.
The returned value *must* be an object of the target type or else an
exception will be thrown.

::

    // src/Dto/LegacyUserData.php
    namespace App\Dto;

    use App\Entity\User;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    // Use a static factory method on the target User class for instantiation
    #[Map(target: User::class, transform: [User::class, 'createFromLegacy'])]
    class LegacyUserData
    {
        public int $userId = 0;
        public string $name = '';
    }

::

    // src/Entity/User.php
    namespace App\Entity;
    class User {
        public string $name = '';
        private int $legacyId = 0;

        private function __construct() {} // Private constructor

        public static function createFromLegacy(mixed $value, object $source): self
        {
            // $value is the initially created (empty) User object
            // $source is the LegacyUserData object
            $user = new self();
            $user->legacyId = $source->userId;
            // Property mapping will happen *after* this method returns $user
            return $user;
        }
    }

Mapping Multiple Targets
------------------------

A source class can be configured to map to multiple different target classes.
Apply the ``#[Map]`` attribute multiple times at the class level, typically
using the ``if`` condition to determine which target is appropriate based on the
source object's state or other logic.

::

    // src/Dto/EventInput.php
    namespace App\Dto;

    use App\Entity\OnlineEvent;
    use App\Entity\PhysicalEvent;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: OnlineEvent::class, if: [self::class, 'isOnline'])]
    #[Map(target: PhysicalEvent::class, if: [self::class, 'isPhysical'])]
    class EventInput
    {
        public string $type = 'online'; // e.g., 'online' or 'physical'
        public string $title = '';

        /**
         * In class-level conditions, $value is null.
         */
        public static function isOnline(?mixed $value, object $source): bool
        {
            return $source->type === 'online';
        }

        public static function isPhysical(?mixed $value, object $source): bool
        {
            return $source->type === 'physical';
        }
    }

    // src/Entity/OnlineEvent.php / PhysicalEvent.php ...

    // Usage:
    $eventInput = new EventInput();
    $eventInput->type = 'physical';
    $mapper = new ObjectMapper();
    $event = $mapper->map($eventInput); // Automatically maps to PhysicalEvent

Mapping Based on Target Properties (Source Mapping)
---------------------------------------------------

Sometimes, it's more convenient to define the mapping on the *target* class,
specifying where its properties should come from in the *source* class. This is
done using the ``source`` parameter within the ``#[Map]`` attribute on the target's
properties.

Note that if both the ``source`` and the ``target`` classes define the ``#[Map]``
attribute, the ``source`` takes precedence.

::

    // src/Api/Payload.php - Represents data from an external API
    namespace App\Api;
    class Payload
    {
        public string $product_name = ''; // Source property name uses snake_case
        public float $price_amount = 0.0;
    }

::

    // src/Entity/Product.php - Our application's entity
    namespace App\Entity;

    use App\Api\Payload;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    // Define that Product can be mapped *from* Payload
    #[Map(source: Payload::class)]
    class Product
    {
        // Define where 'name' should get its value from in the Payload source
        #[Map(source: 'product_name')]
        public string $name = ''; // Target property uses camelCase

        // Define where 'price' should get its value from
        #[Map(source: 'price_amount')]
        public float $price = 0.0;
    }

    // Usage:
    $payload = new Payload();
    $payload->product_name = 'Super Widget';
    $payload->price_amount = 123.45;

    $mapper = new ObjectMapper();
    // Map *from* the payload *to* the Product class
    $product = $mapper->map($payload, Product::class);

    // $product->name = 'Super Widget'
    // $product->price = 123.45

When using ``source`` mapping, the ``ObjectMapper`` detects this configuration if
the source object itself doesn't have ``#[Map]`` attributes pointing outwards. It
then reads the ``#[Map(source: ...)]`` attributes from the target class instead.

Handling Recursion
------------------

The ObjectMapper automatically handles recursive relationships. If an object graph
contains cycles (e.g., a ``User`` has a ``manager`` which is another ``User``,
who might manage the first user), the mapper will reuse the already mapped target
instances to avoid infinite loops.

::

    // src/Entity/User.php
    namespace App\Entity;
    use App\Dto\UserDto;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(target: UserDto::class)]
    class User {
        public string $name = '';
        public ?User $manager = null;
    }

::

    // src/Dto/UserDto.php
    namespace App\Dto;
    use Symfony\Component\ObjectMapper\Attribute\Map;

    #[Map(source: \App\Entity\User::class)] // Can also define mapping here
    class UserDto {
        public string $name = '';
        public ?UserDto $manager = null;
    }

    // Usage:
    $manager = new User();
    $manager->name = 'Alice';
    $employee = new User();
    $employee->name = 'Bob';
    $employee->manager = $manager;
    // Manager's manager is the employee:
    $manager->manager = $employee;

    $mapper = new ObjectMapper();
    $employeeDto = $mapper->map($employee, UserDto::class);

    // Recursion is handled correctly:
    // $employeeDto->name === 'Bob'
    // $employeeDto->manager->name === 'Alice'
    // $employeeDto->manager->manager === $employeeDto

Custom Mapping Logic (MapStruct-like Approach)
----------------------------------------------

For very complex mapping scenarios or if you prefer separating mapping rules from
your DTOs/Entities, you can implement a custom mapping strategy using the
:class:`Symfony\\Component\\ObjectMapper\\Metadata\\ObjectMapperMetadataFactoryInterface`.
This allows defining mapping rules within dedicated "mapper" services, similar
to the approach used by libraries like MapStruct in the Java ecosystem.

First, create your custom metadata factory. The following example reads mapping
rules defined via ``#[Map]`` attributes on a dedicated mapper *service* class,
specifically on its ``map`` method for property mappings and on the class itself
for the source-to-target relationship:

::

    namespace App\ObjectMapper\Metadata;

    use Symfony\Component\ObjectMapper\Attribute\Map;
    use Symfony\Component\ObjectMapper\Metadata\Mapping;
    use Symfony\Component\ObjectMapper\Metadata\ObjectMapperMetadataFactoryInterface;
    use Symfony\Component\ObjectMapper\ObjectMapperInterface;

    /**
     * A Metadata factory that implements basics similar to MapStruct.
     * Reads mapping configuration from attributes on a dedicated mapper service.
     */
    final class MapStructMapperMetadataFactory implements ObjectMapperMetadataFactoryInterface
    {
        /**
         * @param class-string<ObjectMapperInterface> $mapperClass The FQCN of the mapper service class
         */
        public function __construct(private readonly string $mapperClass)
        {
            if (!is_a($this->mapperClass, ObjectMapperInterface::class, true)) {
                throw new \RuntimeException(sprintf('Mapper class "%s" must implement "%s".', $this->mapperClass, ObjectMapperInterface::class));
            }
        }

        public function create(object $object, ?string $property = null, array $context = []): array
        {
            try {
                $refl = new \ReflectionClass($this->mapperClass);
            } catch (\ReflectionException $e) {
                throw new \RuntimeException("Failed to reflect mapper class: " . $e->getMessage(), 0, $e);
            }

            $mapConfigs = [];
            $sourceIdentifier = $property ?? $object::class;

            // Read attributes from the map method (for property mapping) or the class (for class mapping)
            $attributesSource = $property ? $refl->getMethod('map') : $refl;
            foreach ($attributesSource->getAttributes(Map::class, \ReflectionAttribute::IS_INSTANCEOF) as $attribute) {
                $map = $attribute->newInstance();

                // Check if the attribute's source matches the current property or source class
                if ($map->source === $sourceIdentifier) {
                    $mapConfigs[] = new Mapping($map->target, $map->source, $map->if, $map->transform);
                }
            }

            // If it's a property lookup and no specific mapping was found, map to the same property
            if ($property && empty($mapConfigs)) {
                $mapConfigs[] = new Mapping(target: $property, source: $property);
            }

            return $mapConfigs;
        }
    }

Next, define your mapper service class. This class implements
``ObjectMapperInterface`` but typically delegates the actual mapping back to a
standard ``ObjectMapper`` instance configured with the custom metadata factory.
Mapping rules are defined using ``#[Map]`` attributes on this class and its
``map`` method:

::

    namespace App\ObjectMapper;

    use App\Dto\LegacyUser;
    use App\Dto\UserDto;
    use App\ObjectMapper\Metadata\MapStructMapperMetadataFactory;
    use Symfony\Component\ObjectMapper\Attribute\Map;
    use Symfony\Component\ObjectMapper\ObjectMapper;
    use Symfony\Component\ObjectMapper\ObjectMapperInterface;

    // Define the source-to-target mapping at the class level
    #[Map(source: LegacyUser::class, target: UserDto::class)]
    class LegacyUserMapper implements ObjectMapperInterface
    {
        private readonly ObjectMapperInterface $objectMapper;

        // Inject the standard ObjectMapper or necessary dependencies
        public function __construct(?ObjectMapperInterface $objectMapper = null)
        {
            // Create an ObjectMapper instance configured with *this* mapper's rules
            $metadataFactory = new MapStructMapperMetadataFactory(self::class);
            $this->objectMapper = $objectMapper ?? new ObjectMapper($metadataFactory);
        }

        // Define property-specific mapping rules on the map method
        #[Map(source: 'fullName', target: 'name')] // Map LegacyUser::fullName to UserDto::name
        #[Map(source: 'creationTimestamp', target: 'registeredAt', transform: [\DateTimeImmutable::class, 'createFromFormat'])]
        #[Map(source: 'status', if: false)] // Ignore the 'status' property from LegacyUser
        public function map(object $source, object|string|null $target = null): object
        {
            // Delegate the actual mapping to the configured ObjectMapper
            return $this->objectMapper->map($source, $target);
        }
    }

Finally, use your custom mapper service:

::

    use App\Dto\LegacyUser;
    use App\ObjectMapper\LegacyUserMapper;

    $legacyUser = new LegacyUser();
    $legacyUser->fullName = 'Jane Doe';
    $legacyUser->status = 'active'; // This will be ignored

    // Instantiate your custom mapper service
    $mapperService = new LegacyUserMapper();

    // Use the map method of your service
    $userDto = $mapperService->map($legacyUser); // Target (UserDto) is inferred from #[Map] on LegacyUserMapper

This approach keeps mapping logic centralized within dedicated services, which
can be beneficial for complex applications or when adhering to specific architectural
patterns.

Advanced Configuration
----------------------

The ``ObjectMapper`` constructor accepts optional arguments for further customization:

* ``ObjectMapperMetadataFactoryInterface $metadataFactory``: Allows providing a custom metadata factory, as shown in the MapStruct example. Defaults to :class:`Symfony\\Component\\ObjectMapper\\Metadata\\ReflectionObjectMapperMetadataFactory`, which reads ``#[Map]`` attributes directly from source/target classes.
* ``?PropertyAccessorInterface $propertyAccessor``: An instance of the PropertyAccess component to customize how properties are read from the source and written to the target. Useful for accessing private properties or using getters/setters.
* ``?ContainerInterface $transformCallableLocator``: A PSR-11 container (service locator) used to resolve service IDs provided in the ``transform`` option of ``#[Map]``.
* ``?ContainerInterface $conditionCallableLocator``: A PSR-11 container used to resolve service IDs provided in the ``if`` option of ``#[Map]``.

When using the ``ObjectMapperInterface`` service provided by the Symfony framework,
these dependencies (like the property accessor and service locators for callables)
are typically configured automatically via the service container.
