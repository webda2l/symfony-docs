How to Implement CSRF Protection
================================

CSRF, or `Cross-site request forgery`_, is a type of attack where a malicious actor
tricks a user into performing actions on a web application without their knowledge
or consent.

The attack is based on the trust that a web application has in a user's browser
(e.g. on session cookies). Here's a real example of a CSRF attack: a malicious
actor could create the following website:

.. code-block:: html

    <html>
        <body>
            <form action="https://example.com/settings/update-email" method="POST">
                <input type="hidden" name="email" value="malicious-actor-address@some-domain.com"/>
            </form>
            <script>
                document.forms[0].submit();
            </script>

            <!-- some content here to distract the user -->
        </body>
    </html>

If you visit this website (e.g. by clicking on some email link or some social
network post) and you were already logged in on the ``https://example.com`` site,
the malicious actor could change the email address associated to your account
(effectively taking over your account) without you even being aware of it.

An effective way of preventing CSRF attacks is to use anti-CSRF tokens. These are
unique tokens added to forms as hidden fields. The legit server validates them to
ensure that the request originated from the expected source and not some other
malicious website.

Anti-CSRF tokens can be managed either in a stateful way: they're put in the
session and are unique for each user and for each kind of action, or in a
stateless way: they're generated on the client-side.

Installation
------------

Symfony provides all the needed features to generate and validate the anti-CSRF
tokens. Before using them, install this package in your project:

.. code-block:: terminal

    $ composer require symfony/security-csrf

Then, enable/disable the CSRF protection with the ``csrf_protection`` option
(see the :ref:`CSRF configuration reference <reference-framework-csrf-protection>`
for more information):

.. configuration-block::

    .. code-block:: yaml

        # config/packages/framework.yaml
        framework:
            # ...
            csrf_protection: ~

    .. code-block:: xml

        <!-- config/packages/framework.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony
                https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">

            <framework:config>
                <framework:csrf-protection enabled="true"/>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/framework.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $framework): void {
            $framework->csrfProtection()
                ->enabled(true)
            ;
        };

By default, the tokens used for CSRF protection are stored in the session.
That's why a session is started automatically as soon as you render a form
with CSRF protection.

.. _caching-pages-that-contain-csrf-protected-forms:

This leads to many strategies to help with caching pages that include CSRF
protected forms, among them:

* Embed the form inside an uncached :doc:`ESI fragment </http_cache/esi>` and
  cache the rest of the page contents;
* Cache the entire page and load the form via an uncached AJAX request;
* Cache the entire page and use :ref:`hinclude.js <templates-hinclude>` to
  load the CSRF token with an uncached AJAX request and replace the form
  field value with it.

The most effective way to cache pages that need CSRF protected forms is to use
stateless CSRF tokens, see below.

.. _csrf-protection-forms:

CSRF Protection in Symfony Forms
--------------------------------

:doc:`Symfony Forms </forms>` include CSRF tokens by default and Symfony also
checks them automatically for you. So, when using Symfony Forms, you don't have
to do anything to be protected against CSRF attacks.

.. _form-csrf-customization:

By default Symfony adds the CSRF token in a hidden field called ``_csrf_token``, but
this can be customized (1) globally for all forms and (2) on a form-by-form basis.
Globally, you can configure it under the ``framework.form`` option:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/framework.yaml
        framework:
            # ...
            form:
                csrf_protection:
                    enabled: true
                    field_name: 'custom_token_name'

    .. code-block:: xml

        <!-- config/packages/framework.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony
                https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">

            <framework:config>
                <framework:form>
                    <framework:csrf-protection enabled="true" field-name="custom_token_name"/>
                </framework:form>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/framework.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $framework) {
            $framework->form()->csrfProtection()
                ->enabled(true)
                ->fieldName('custom_token_name')
            ;
        };

On a form-by-form basis, you can configure the CSRF protection in the ``setDefaults()``
method of each form::

    // src/Form/TaskType.php
    namespace App\Form;

    // ...
    use App\Entity\Task;
    use Symfony\Component\OptionsResolver\OptionsResolver;

    class TaskType extends AbstractType
    {
        // ...

        public function configureOptions(OptionsResolver $resolver): void
        {
            $resolver->setDefaults([
                'data_class'      => Task::class,
                // enable/disable CSRF protection for this form
                'csrf_protection' => true,
                // the name of the hidden HTML field that stores the token
                'csrf_field_name' => '_token',
                // an arbitrary string used to generate the value of the token
                // using a different string for each form improves its security
                // when using stateful tokens (which is the default)
                'csrf_token_id'   => 'task_item',
            ]);
        }

        // ...
    }

You can also customize the rendering of the CSRF form field by creating a custom
:doc:`form theme </form/form_themes>` and using ``csrf_token`` as the prefix of
the field (e.g. define ``{% block csrf_token_widget %} ... {% endblock %}`` to
customize the entire form field contents).

.. _csrf-protection-in-login-forms:

CSRF Protection in Login Form and Logout Action
-----------------------------------------------

Read the following:

* :ref:`CSRF Protection in Login Forms <form_login-csrf>`;
* :ref:`CSRF protection for the logout action <reference-security-logout-csrf>`.

.. _csrf-protection-in-html-forms:

Generating and Checking CSRF Tokens Manually
--------------------------------------------

Although Symfony Forms provide automatic CSRF protection by default, you may
need to generate and check CSRF tokens manually for example when using regular
HTML forms not managed by the Symfony Form component.

Consider a HTML form created to allow deleting items. First, use the
:ref:`csrf_token() Twig function <reference-twig-function-csrf-token>` to
generate a CSRF token in the template and store it as a hidden form field:

.. code-block:: html+twig

    <form action="{{ url('admin_post_delete', { id: post.id }) }}" method="post">
        {# the argument of csrf_token() is the ID of this token #}
        <input type="hidden" name="token" value="{{ csrf_token('delete-item') }}">

        <button type="submit">Delete item</button>
    </form>

Then, get the value of the CSRF token in the controller action and use the
:method:`Symfony\\Bundle\\FrameworkBundle\\Controller\\AbstractController::isCsrfTokenValid`
method to check its validity, passing the same token ID used in the template::

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    // ...

    public function delete(Request $request): Response
    {
        $submittedToken = $request->getPayload()->get('token');

        // 'delete-item' is the same value used in the template to generate the token
        if ($this->isCsrfTokenValid('delete-item', $submittedToken)) {
            // ... do something, like deleting an object
        }
    }

.. _csrf-controller-attributes:

Alternatively you can use the
:class:`Symfony\\Component\\Security\\Http\\Attribute\\IsCsrfTokenValid`
attribute on the controller action::

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Security\Http\Attribute\IsCsrfTokenValid;
    // ...

    #[IsCsrfTokenValid('delete-item', tokenKey: 'token')]
    public function delete(): Response
    {
        // ... do something, like deleting an object
    }

Suppose you want a CSRF token per item, so in the template you have something like the following:

.. code-block:: html+twig

    <form action="{{ url('admin_post_delete', { id: post.id }) }}" method="post">
        {# the argument of csrf_token() is a dynamic id string used to generate the token #}
        <input type="hidden" name="token" value="{{ csrf_token('delete-item-' ~ post.id) }}">

        <button type="submit">Delete item</button>
    </form>

The :class:`Symfony\\Component\\Security\\Http\\Attribute\\IsCsrfTokenValid`
attribute also accepts an :class:`Symfony\\Component\\ExpressionLanguage\\Expression`
object evaluated to the id::

    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Security\Http\Attribute\IsCsrfTokenValid;
    // ...

    #[IsCsrfTokenValid(new Expression('"delete-item-" ~ args["post"].getId()'), tokenKey: 'token')]
    public function delete(Post $post): Response
    {
        // ... do something, like deleting an object
    }

.. versionadded:: 7.1

    The :class:`Symfony\\Component\\Security\\Http\\Attribute\\IsCsrfTokenValid`
    attribute was introduced in Symfony 7.1.

CSRF Tokens and Compression Side-Channel Attacks
------------------------------------------------

`BREACH`_ and `CRIME`_ are security exploits against HTTPS when using HTTP
compression. Attackers can leverage information leaked by compression to recover
targeted parts of the plaintext. To mitigate these attacks, and prevent an
attacker from guessing the CSRF tokens, a random mask is prepended to the token
and used to scramble it.

Stateless CSRF Tokens
---------------------

.. versionadded:: 7.2

    Stateless anti-CSRF protection was introduced in Symfony 7.2.

By default CSRF tokens are stateful, which means they're stored in the session.
But some token ids can be declared as stateless using the ``stateless_token_ids``
option:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/csrf.yaml
        framework:
            # ...
            csrf_protection:
                stateless_token_ids: ['submit', 'authenticate', 'logout']

    .. code-block:: xml

        <!-- config/packages/csrf.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony
                https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">

            <framework:config>
                <framework:csrf-protection>
                    <framework:stateless-token-id>submit</framework:stateless-token-id>
                    <framework:stateless-token-id>authenticate</framework:stateless-token-id>
                    <framework:stateless-token-id>logout</framework:stateless-token-id>
                </framework:csrf-protection>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/csrf.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $framework): void {
            $framework->csrfProtection()
                ->statelessTokenIds(['submit', 'authenticate', 'logout'])
            ;
        };

Stateless CSRF tokens use a CSRF protection that doesn't need the session. This
means that you can cache the entire page and still have CSRF protection.

When a stateless CSRF token is checked for validity, Symfony verifies the
``Origin`` and the ``Referer`` headers of the incoming HTTP request.

If either of these headers match the target origin of the application (its domain
name), the CSRF token is considered valid. This relies on the app being able to
know its own target origin. Don't miss configuring your reverse proxy if you're
behind one. See :doc:`/deployment/proxies`.

Using a Default Token ID
~~~~~~~~~~~~~~~~~~~~~~~~

While stateful CSRF tokens are better seggregated per form or action, stateless
ones don't need many token identifiers. In the previous example, ``authenticate``
and ``logout`` are listed because they're the default identifiers used by the
Symfony Security component. The ``submit`` identifier is then listed so that
form types defined by the application can use it by default. The following
configuration - which applies only to form types declared using autofiguration
(the default way to declare *your* services) - will make your form types use the
``submit`` token identifier by default:

.. configuration-block::

    .. code-block:: yaml

        # config/packages/csrf.yaml
        framework:
            form:
                csrf_protection:
                    token_id: 'submit'

    .. code-block:: xml

        <!-- config/packages/csrf.xml -->
        <?xml version="1.0" encoding="UTF-8" ?>
        <container xmlns="http://symfony.com/schema/dic/services"
            xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xmlns:framework="http://symfony.com/schema/dic/symfony"
            xsi:schemaLocation="http://symfony.com/schema/dic/services
                https://symfony.com/schema/dic/services/services-1.0.xsd
                http://symfony.com/schema/dic/symfony
                https://symfony.com/schema/dic/symfony/symfony-1.0.xsd">

            <framework:config>
                <framework:form>
                    <framework:csrf-protection token-id="submit"/>
                </framework:form>
            </framework:config>
        </container>

    .. code-block:: php

        // config/packages/csrf.php
        use Symfony\Config\FrameworkConfig;

        return static function (FrameworkConfig $framework): void {
            $framework->form()
                ->csrfProtection()
                    ->tokenId('submit')
            ;
        };

Forms configured with a token identifier listed in the above ``stateless_token_ids``
option will use the stateless CSRF protection.

Generating CSRF Token Using Javascript
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In addition to the ``Origin`` and ``Referer`` headers, stateless CSRF protection
also checks a cookie and a header (named ``csrf-token`` by default, see the
:ref:`CSRF configuration reference <reference-framework-csrf-protection>`).

These extra checks are part of defense-in-depth strategies provided by the
stateless CSRF protection. They are optional and they require
`some JavaScript`_ to be activated. This JavaScript is responsible for generating
a crypto-safe random token when a form is submitted, then putting the token in
the hidden CSRF field of the form and submitting it also as a cookie and header.
On the server-side, the CSRF token is validated by checking the cookie and header
values. This "double-submit" protection relies on the same-origin policy
implemented by browsers and is strengthened by regenerating the token at every
form submission - which prevents cookie fixation issues - and by using
``samesite=strict`` and ``__Host-`` cookies, which make them domain-bound and
HTTPS-only.

Note that the default snippet of JavaScript provided by Symfony requires that
the hidden CSRF form field is either named ``_csrf_token``, or that it has the
``data-controller="csrf-protection"`` attribute. You can of course take
inspiration from this snippet to write your own, provided you follow the same
protocol.

As a last measure, a behavioral check is added on the server-side to ensure that
the validation method cannot be downgraded: if and only if a session is already
available, successful "double-submit" is remembered and is then required for
subsequent requests. This prevents attackers from exploiting potentially reduced
validation checks once cookie and/or header validation has been confirmed as
effective (they're optional by default as explained above).

.. note::

    Enforcing successful "double-submit" for every requests is not recommended as
    as it could lead to a broken user experience. The opportunistic approach
    described above is preferred because it allows the application to gracefully
    degrade to ``Origin`` / ``Referer`` checks when JavaScript is not available.

.. _`Cross-site request forgery`: https://en.wikipedia.org/wiki/Cross-site_request_forgery
.. _`BREACH`: https://en.wikipedia.org/wiki/BREACH
.. _`CRIME`: https://en.wikipedia.org/wiki/CRIME
.. _`some JavaScript`: https://github.com/symfony/recipes/blob/main/symfony/stimulus-bundle/2.20/assets/controllers/csrf_protection_controller.js
