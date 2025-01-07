Slug
====

.. versionadded:: 7.3

    The ``Slug`` constraint was introduced in Symfony 7.3.

Validates that a value is a slug. By default, a slug is a string that matches
the following regular expression: ``/^[a-z0-9]+(?:-[a-z0-9]+)*$/``.

.. include:: /reference/constraints/_empty-values-are-valid.rst.inc

==========  ===================================================================
Applies to  :ref:`property or method <validation-property-target>`
Class       :class:`Symfony\\Component\\Validator\\Constraints\\Slug`
Validator   :class:`Symfony\\Component\\Validator\\Constraints\\SlugValidator`
==========  ===================================================================

Basic Usage
-----------

The ``Slug`` constraint can be applied to a property or a getter method:

.. configuration-block::

    .. code-block:: php-attributes

        // src/Entity/Author.php
        namespace App\Entity;

        use Symfony\Component\Validator\Constraints as Assert;

        class Author
        {
            #[Assert\Slug]
            protected string $slug;
        }

    .. code-block:: yaml

        # config/validator/validation.yaml
        App\Entity\Author:
            properties:
                slug:
                    - Slug: ~

    .. code-block:: xml

        <!-- config/validator/validation.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <constraint-mapping xmlns="http://symfony.com/schema/dic/constraint-mapping"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="http://symfony.com/schema/dic/constraint-mapping https://symfony.com/schema/dic/constraint-mapping/constraint-mapping-1.0.xsd">

            <class name="App\Entity\Author">
                <property name="slug">
                    <constraint name="Slug"/>
                </property>
            </class>
        </constraint-mapping>

    .. code-block:: php

        // src/Entity/Author.php
        namespace App\Entity;

        use Symfony\Component\Validator\Constraints as Assert;
        use Symfony\Component\Validator\Mapping\ClassMetadata;

        class Author
        {
            // ...

            public static function loadValidatorMetadata(ClassMetadata $metadata): void
            {
                $metadata->addPropertyConstraint('slug', new Assert\Slug());
            }
        }

Examples of valid values:

* foobar
* foo-bar
* foo123
* foo-123bar

Uppercase characters would result in an violation of this constraint.

Options
-------

``regex``
~~~~~~~~~

**type**: ``string`` default: ``/^[a-z0-9]+(?:-[a-z0-9]+)*$/``

This option allows you to modify the regular expression pattern that the input
will be matched against via the :phpfunction:`preg_match` PHP function.

If you need to use it, you might also want to take a look at the :doc:`Regex constraint <Regex>`.

``message``
~~~~~~~~~~~

**type**: ``string`` **default**: ``This value is not a valid slug``

This is the message that will be shown if this validator fails.

You can use the following parameters in this message:

=================  ==============================================================
Parameter          Description
=================  ==============================================================
``{{ value }}``    The current (invalid) value
=================  ==============================================================

.. include:: /reference/constraints/_groups-option.rst.inc

.. include:: /reference/constraints/_payload-option.rst.inc
