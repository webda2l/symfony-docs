Validating Twig Template Syntax
===============================

The Twig Bridge provides a custom `Twig` constraint that allows validating
whether a given string contains valid Twig syntax.

This is particularly useful when template content is user-generated or
configurable, and you want to ensure it can be safely rendered by the Twig engine.

Installation
------------

This constraint is part of the `symfony/twig-bridge` package. Make sure it's installed:

.. code-block:: terminal

    $ composer require symfony/twig-bridge

Usage
-----

To use the `Twig` constraint, annotate the property that should contain a valid
Twig template::

    use Symfony\Bridge\Twig\Validator\Constraints\Twig;

    class Template
    {
        #[Twig]
        private string $templateCode;
    }

If the template contains a syntax error, a validation error will be thrown.

Constraint Options
------------------

**message**
Customize the default error message when the template is invalid::

    // ...
    class Template
    {
        #[Twig(message: 'Your template contains a syntax error.')]
        private string $templateCode;
    }

**skipDeprecations**
By default, this option is set to `true`, which means Twig deprecation warnings
are ignored during validation.

If you want validation to fail when deprecated features are used in the template,
set this to `false`::

    // ...
    class Template
    {
        #[Twig(skipDeprecations: false)]
        private string $templateCode;
    }

This can be helpful when enforcing stricter template rules or preparing for major
Twig version upgrades.
