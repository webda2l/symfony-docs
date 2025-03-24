Referencing Entities with Abstract Classes and Interfaces
=========================================================

In applications where functionality is segregated with minimal concrete dependencies
between the various layers, such as monoliths which are split into multiple modules,
it might be hard to prevent hard dependencies on entities between modules.

Doctrine 2.2 includes a new utility called the ``ResolveTargetEntityListener``,
that functions by intercepting certain calls inside Doctrine and rewriting
``targetEntity`` parameters in your metadata mapping at runtime. It means that
you are able to use an interface or abstract class in your mappings and expect
correct mapping to a concrete entity at runtime.

This functionality allows you to define relationships between different entities
without making them hard dependencies.

.. tip::

    Starting with Symfony 7.3, this functionality also works with the ``EntityValueResolver``.
    See :ref:`doctrine-entity-value-resolver-resolve-target-entities` for more details.

Background
----------

Suppose you have an application which provides two modules; an Invoice module which
provides invoicing functionality, and a Customer module that contains customer management
tools. You want to keep dependencies between these modules separated, because they should
not be aware of the other module's implementation details.

In this case, you have an ``Invoice`` entity with a relationship to the interface
``InvoiceSubjectInterface``. This is not recognized as a valid entity by Doctrine.
The goal is to get the ``ResolveTargetEntityListener`` to replace any mention of the interface
with a real object that implements that interface.

Set up
------

This article uses the following two basic entities (which are incomplete for
brevity) to explain how to set up and use the ``ResolveTargetEntityListener``.

A Customer entity::

    // src/Entity/Customer.php
    namespace App\Entity;

    use App\Entity\CustomerInterface as BaseCustomer;
    use App\Model\InvoiceSubjectInterface;
    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity]
    #[ORM\Table(name: 'customer')]
    class Customer extends BaseCustomer implements InvoiceSubjectInterface
    {
        // In this example, any methods defined in the InvoiceSubjectInterface
        // are already implemented in the BaseCustomer
    }

An Invoice entity::

    // src/Entity/Invoice.php
    namespace App\Entity;

    use App\Model\InvoiceSubjectInterface;
    use Doctrine\ORM\Mapping as ORM;

    /**
     * Represents an Invoice.
     */
    #[ORM\Entity]
    #[ORM\Table(name: 'invoice')]
    class Invoice
    {
        #[ORM\ManyToOne(targetEntity: InvoiceSubjectInterface::class)]
        protected InvoiceSubjectInterface $subject;
    }

An InvoiceSubjectInterface::

    // src/Model/InvoiceSubjectInterface.php
    namespace App\Model;

    /**
     * An interface that the invoice Subject object should implement.
     * In most circumstances, only a single object should implement
     * this interface as the ResolveTargetEntityListener can only
     * change the target to a single object.
     */
    interface InvoiceSubjectInterface
    {
        // List any additional methods that your InvoiceBundle
        // will need to access on the subject so that you can
        // be sure that you have access to those methods.

        public function getName(): string;
    }

Next, you need to configure the ``resolve_target_entities`` option, which tells the DoctrineBundle
about the replacement:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/doctrine.yaml
        doctrine:
            # ...
            orm:
                # ...
                resolve_target_entities:
                    App\Model\InvoiceSubjectInterface: App\Entity\Customer

    .. code-block:: xml

        <!-- config/packages/doctrine.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:doctrine="http://symfony.com/schema/dic/doctrine"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/doctrine
                https://symfony.com/schema/dic/doctrine/doctrine-1.0.xsd">

            <doctrine:config>
                <doctrine:orm>
                    <!-- ... -->
                    <doctrine:resolve-target-entity interface="App\Model\InvoiceSubjectInterface">App\Entity\Customer</doctrine:resolve-target-entity>
                </doctrine:orm>
            </doctrine:config>
        </container>

    .. code-block:: php

        // config/packages/doctrine.php
        use App\Entity\Customer;
        use App\Model\InvoiceSubjectInterface;
        use Symfony\Config\DoctrineConfig;

        return static function (DoctrineConfig $doctrine): void {
            $orm = $doctrine->orm();
            // ...
            $orm->resolveTargetEntity(InvoiceSubjectInterface::class, Customer::class);
        };

Final Thoughts
--------------

With the ``ResolveTargetEntityListener``, you are able to decouple your
modules, keeping them usable by themselves, but still being able to
define relationships between different objects. By using this method,
your modules will end up being easier to maintain independently.
