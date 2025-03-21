Tree Helper
===========

The Tree Helper allows you to build and display tree structures in the console.

.. versionadded:: 7.3

    The ``TreeHelper`` class was introduced in Symfony 7.3.

Rendering a Tree
----------------

The :method:`Symfony\\Component\\Console\\Helper\\TreeHelper::createTree` method creates a tree structure from an array and returns a :class:`Symfony\\Component\\Console\\Helper\\Tree`
object that can be rendered in the console.

Building a Tree from an Array
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

You can build a tree from an array by passing the array to the :method:`Symfony\\Component\\Console\\Helper\\TreeHelper::createTree`
method::

    use Symfony\Component\Console\Helper\TreeHelper;

    $tree = TreeHelper::createTree($io, null, [
        'src' =>  [
            'Command',
            'Controller' => [
                'DefaultController.php',
            ],
            'Kernel.php',
        ],
        'templates' => [
            'base.html.twig',
        ],
    ]);

    $tree->render();

The above code will output the following tree:

.. code-block:: text

    ├── src
    │   ├── Command
    │   ├── Controller
    │   │   └── DefaultController.php
    │   └── Kernel.php
    └── templates
        └── base.html.twig

Manually Creating a Tree
~~~~~~~~~~~~~~~~~~~~~~~~

You can manually create a tree by creating a new instance of the :class:`Symfony\\Component\\Console\\Helper\\Tree` class and adding nodes to it::

    use Symfony\Component\Console\Helper\TreeHelper;
    use Symfony\Component\Console\Helper\TreeNode;

    $node = TreeNode::fromValues([
        'Command',
        'Controller' => [
            'DefaultController.php',
        ],
        'Kernel.php',
    ]);
    $node->addChild('templates');
    $node->addChild('tests');

    $tree = TreeHelper::createTree($io, $node);
    $tree->render();

Customizing the Tree Style
--------------------------

Built-in Tree Styles
~~~~~~~~~~~~~~~~~~~~

The tree helper provides a few built-in styles that you can use to customize the output of the tree.

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::default`

    .. code-block:: text

        ├── config
        │   ├── packages
        │   └── routes
        │      ├── framework.yaml
        │      └── web_profiler.yaml
        ├── src
        │   ├── Command
        │   ├── Controller
        │   │   └── DefaultController.php
        │   └── Kernel.php
        └── templates
           └── base.html.twig

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::box`

    .. code-block:: text

        ┃╸ config
        ┃  ┃╸ packages
        ┃  ┗╸ routes
        ┃     ┃╸ framework.yaml
        ┃     ┗╸ web_profiler.yaml
        ┃╸ src
        ┃  ┃╸ Command
        ┃  ┃╸ Controller
        ┃  ┃  ┗╸ DefaultController.php
        ┃  ┗╸ Kernel.php
        ┗╸ templates
           ┗╸ base.html.twig

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::doubleBox`

    .. code-block:: text

        ╠═ config
        ║  ╠═ packages
        ║  ╚═ routes
        ║    ╠═ framework.yaml
        ║    ╚═ web_profiler.yaml
        ╠═ src
        ║  ╠═ Command
        ║  ╠═ Controller
        ║  ║  ╚═ DefaultController.php
        ║  ╚═ Kernel.php
        ╚═ templates
          ╚═ base.html.twig

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::compact`

    .. code-block:: text

        ├ config
        │ ├ packages
        │ └ routes
        │   ├ framework.yaml
        │   └ web_profiler.yaml
        ├ src
        │ ├ Command
        │ ├ Controller
        │ │ └ DefaultController.php
        │ └ Kernel.php
        └ templates
          └ base.html.twig

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::light`

    .. code-block:: text

        |-- config
        |   |-- packages
        |   `-- routes
        |       |-- framework.yaml
        |       `-- web_profiler.yaml
        |-- src
        |   |-- Command
        |   |-- Controller
        |   |   `-- DefaultController.php
        |   `-- Kernel.php
        `-- templates
            `-- base.html.twig

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::minimal`

    .. code-block:: text

        . config
        . . packages
        . . routes
        .   . framework.yaml
        .   . web_profiler.yaml
        . src
        . . Command
        . . Controller
        . . . DefaultController.php
        . . Kernel.php
        . templates
          . base.html.twig

:method:`Symfony\\Component\\Console\\Helper\\TreeStyle::rounded`

    .. code-block:: text

        ├─ config
        │  ├─ packages
        │  ╰─ routes
        │     ├─ framework.yaml
        │     ╰─ web_profiler.yaml
        ├─ src
        │  ├─ Command
        │  ├─ Controller
        │  │  ╰─ DefaultController.php
        │  ╰─ Kernel.php
        ╰─ templates
           ╰─ base.html.twig

Making a Custom Tree Style
~~~~~~~~~~~~~~~~~~~~~~~~~~

You can create your own tree style by passing the characters to the constructor
of the :class:`Symfony\\Component\\Console\\Helper\\TreeStyle` class::

    use Symfony\Component\Console\Helper\TreeHelper;
    use Symfony\Component\Console\Helper\TreeStyle;

    $customStyle = new TreeStyle('🟣 ', '🟠 ', '🔵 ', '🟢 ', '🔴 ', '🟡 ');

    // Pass the custom style to the createTree method

    $tree = TreeHelper::createTree($io, null, [
        'src' =>  [
            'Command',
            'Controller' => [
                'DefaultController.php',
            ],
            'Kernel.php',
        ],
        'templates' => [
            'base.html.twig',
        ],
    ], $customStyle);

    $tree->render();

The above code will output the following tree:

.. code-block:: text

    🔵 🟣 🟡 src
    🔵 🟢 🟣 🟡 Command
    🔵 🟢 🟣 🟡 Controller
    🔵 🟢 🟢 🟠 🟡 DefaultController.php
    🔵 🟢 🟠 🟡 Kernel.php
    🔵 🟠 🟡 templates
    🔵 🔴 🟠 🟡 base.html.twig
