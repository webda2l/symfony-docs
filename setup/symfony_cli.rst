.. _symfony-server:
.. _symfony-local-web-server:

Symfony CLI
===========

The Symfony CLI is a developer tool to help you build, run, and manage your
Symfony applications directly from your terminal. It's designed to boost your
productivity with smart features like:

* Local web server optimized for development
* Seamless integration with Platform.sh services
* Docker support with automatic environment variable management
* Multiple PHP version management
* Background worker management
* And much more!

Installation
------------

The Symfony CLI is available as a standalone executable that supports Linux,
macOS, and Windows.

Download and install it following the instructions on `symfony.com/download`_.

Shell Autocompletion
~~~~~~~~~~~~~~~~~~~~

The Symfony CLI supports autocompletion for Bash, Zsh, and Fish shells. This
helps you type commands faster and discover available options:

.. code-block:: terminal

    # Install autocompletion (do this only once)
    $ symfony completion bash | sudo tee /etc/bash_completion.d/symfony

    # For Zsh users
    $ symfony completion zsh > ~/.symfony_completion && echo "source ~/.symfony_completion" >> ~/.zshrc

    # For Fish users
    $ symfony completion fish | source

After installation, restart your terminal to enable autocompletion. The CLI will
also provide autocompletion for ``composer`` and ``console`` commands when it
detects a Symfony project.

.. note::

    You can view and contribute to the Symfony CLI source in the
    `symfony-cli/symfony-cli GitHub repository`_.

Creating New Symfony Applications
---------------------------------

The Symfony CLI includes a powerful project creation command that helps you
start new projects quickly:

.. code-block:: terminal

    # Create a new Symfony project (latest stable version)
    $ symfony new my_project

    # Create a project with the latest LTS (Long Term Support) version
    $ symfony new my_project --version=lts

    # Create a project based on a specific Symfony version
    $ symfony new my_project --version=6.4

    # Create a project using the development version
    $ symfony new my_project --version=next

    # Create a minimal project (microkernel)
    $ symfony new my_project --version=6.4

    # Create a demo application
    $ symfony new my_project --demo

.. tip::

    Pass the ``--cloud`` option to initialize a Platform.sh project at the same
    time the Symfony project is created. Pass the ``--webapp`` option if you
    prefer a project where Twig is already configured.

Running the Local Web Server
----------------------------

The Symfony CLI includes a powerful local web server for development. This
server is not intended for production use but provides features that make
development more productive:

* HTTPS support with automatic certificate generation
* HTTP/2 support
* Automatic PHP version selection
* Integration with Docker services
* Built-in proxy for custom domain names

Getting Started
~~~~~~~~~~~~~~~

To serve a Symfony project with the local server:

.. code-block:: terminal

    $ cd my-project/
    $ symfony server:start

      [OK] Web server listening on http://127.0.0.1:8000
      ...

    # Browse the site using your default browser
    $ symfony open:local

Running the server this way displays log messages in the console. To run it in
the background:

.. code-block:: terminal

    $ symfony server:start -d

    # Continue working and running other commands...

    # View the latest log messages
    $ symfony server:log

    # Stop the background server
    $ symfony server:stop

.. tip::

    On macOS, you might see a warning about accepting incoming network
    connections. To fix this, sign the Symfony binary:

    .. code-block:: terminal

        $ sudo codesign --force --deep --sign - $(whereis -q symfony)

Enabling HTTPS/TLS
~~~~~~~~~~~~~~~~~~

Running your application over HTTPS locally helps detect mixed content issues
early and allows using features that require secure connections:

.. code-block:: terminal

    # Install the certificate authority (only once)
    $ symfony server:ca:install

    # Now start your server - it will use HTTPS automatically
    $ symfony server:start

.. tip::

    For WSL (Windows Subsystem for Linux) users, manually import the certificate
    authority in Windows:

    .. code-block:: terminal

        $ explorer.exe `wslpath -w $HOME/.symfony5/certs`

    Then double-click on ``default.p12`` to import it.

PHP Management
--------------

The Symfony CLI provides powerful PHP management features, allowing you to use
different PHP versions for different projects.

Selecting PHP Version
~~~~~~~~~~~~~~~~~~~~~

Create a ``.php-version`` file at your project root:

.. code-block:: terminal

    $ cd my-project/

    # Use a specific PHP version
    $ echo 8.2 > .php-version

    # Use any PHP 8.x version available
    $ echo 8 > .php-version

To see all available PHP versions:

.. code-block:: terminal

    $ symfony local:php:list

.. tip::

    You can create a ``.php-version`` file in a parent directory to set the same
    PHP version for multiple projects.

Custom PHP Configuration
~~~~~~~~~~~~~~~~~~~~~~~~

Override PHP settings per project by creating a ``php.ini`` file at the project
root:

.. code-block:: ini

    ; php.ini
    [Date]
    date.timezone = Europe/Paris

    [PHP]
    memory_limit = 256M

Using PHP Commands
~~~~~~~~~~~~~~~~~~

Use ``symfony php`` to ensure commands run with the correct PHP version:

.. code-block:: terminal

    # Runs with the system's default PHP
    $ php -v

    # Runs with the project's PHP version
    $ symfony php -v

    # This also works for Composer
    $ symfony composer install

Local Domain Names
------------------

Instead of using ``127.0.0.1:8000``, you can use custom domain names like
``https://my-app.wip``.

Setting up the Local Proxy
~~~~~~~~~~~~~~~~~~~~~~~~~~

1. Configure your system to use the Symfony proxy:

   * Open your system's proxy settings
   * Set ``http://127.0.0.1:7080/proxy.pac`` as the automatic proxy configuration URL

2. Start the proxy:

   .. code-block:: terminal

       $ symfony proxy:start

3. Attach a domain to your project:

   .. code-block:: terminal

       $ cd my-project/
       $ symfony proxy:domain:attach my-app

   Your application is now available at ``https://my-app.wip``

.. tip::

    View all local domains and their configuration at http://127.0.0.1:7080

You can also use wildcards:

.. code-block:: terminal

    $ symfony proxy:domain:attach "*.my-app"

This allows accessing subdomains like ``https://api.my-app.wip`` or
``https://admin.my-app.wip``.

.. _symfony-server-docker:

Docker Integration
------------------

The local server automatically detects Docker services and exposes their
connection information as environment variables.

Automatic Service Detection
~~~~~~~~~~~~~~~~~~~~~~~~~~~

With this ``compose.yaml``:

.. code-block:: yaml

    services:
        database:
            image: mysql:8
            ports: [3306]

The server automatically creates these environment variables:

* ``DATABASE_URL``
* ``DATABASE_HOST``
* ``DATABASE_PORT``

Supported services include MySQL, PostgreSQL, Redis, RabbitMQ, Elasticsearch,
MongoDB, and more.

.. tip::

    Run ``symfony var:export`` to see all exposed environment variables.

Service Naming
~~~~~~~~~~~~~~

If your service names don't match Symfony conventions, use labels:

.. code-block:: yaml

    services:
        db:
            image: postgres:15
            ports: [5432]
            labels:
                com.symfony.server.service-prefix: 'DATABASE'

Managing Long-Running Processes
-------------------------------

Use the Symfony CLI to manage long-running processes like Webpack watchers:

.. code-block:: terminal

    # Start webpack watcher in the background
    $ symfony run -d npx encore dev --watch

    # View logs
    $ symfony server:log

    # Check status
    $ symfony server:status

.. _symfony-server_configuring-workers:

Configuring Workers
~~~~~~~~~~~~~~~~~~~

Define processes that should start automatically with the server in
``.symfony.local.yaml``:

.. code-block:: yaml

    # .symfony.local.yaml
    workers:
        # Built-in Encore integration
        npm_encore_watch: ~

        # Messenger consumer with file watching
        messenger_consume_async:
            cmd: ['symfony', 'console', 'messenger:consume', 'async']
            watch: ['config', 'src', 'templates', 'vendor']

        # Custom commands
        build_spa:
            cmd: ['npm', 'run', 'watch']

        # Auto-start Docker Compose
        docker_compose: ~

Advanced Configuration
----------------------

The ``.symfony.local.yaml`` file provides advanced configuration options:

.. code-block:: yaml

    # Custom domains
    proxy:
        domains:
            - app
            - admin.app

    # HTTP server settings
    http:
        document_root: public/
        passthru: index.php
        port: 8000
        preferred_port: 8001
        allow_http: false
        no_tls: false
        p12: path/to/custom-cert.p12

Platform.sh Integration
-----------------------

The Symfony CLI provides seamless integration with `Platform.sh`_:

.. code-block:: terminal

    # Open Platform.sh web UI
    $ symfony cloud:web

    # Deploy to Platform.sh
    $ symfony cloud:deploy

    # Create a new environment
    $ symfony cloud:env:create feature-xyz

For more Platform.sh features, see the `Platform.sh documentation`_.

Troubleshooting
---------------

**Server doesn't start**: Check if the port is already in use:

.. code-block:: terminal

    $ symfony server:status
    $ symfony server:stop  # If a server is already running

**HTTPS not working**: Ensure the CA is installed:

.. code-block:: terminal

    $ symfony server:ca:install

**Docker services not detected**: Check that Docker is running and environment
variables are properly exposed:

.. code-block:: terminal

    $ docker compose ps
    $ symfony var:export --debug

**Proxy domains not working**: 

* Clear your browser cache
* Check proxy settings in your system
* For Chrome, visit ``chrome://net-internals/#proxy`` and click "Re-apply settings"

.. _`symfony.com/download`: https://symfony.com/download
.. _`symfony-cli/symfony-cli GitHub repository`: https://github.com/symfony-cli/symfony-cli
.. _`Platform.sh`: https://symfony.com/cloud/
.. _`Platform.sh documentation`: https://docs.platform.sh/frameworks/symfony.html