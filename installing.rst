Installing the Library
======================

Installing RollerworksSearch is trivial. By using Composer to install
the dependencies you don't have to worry about compatibility or autoloading.

`Composer`_ is a dependency management library for PHP, which you can use
to download the RollerworksSearch library, and at your choice any extensions.

Start by `downloading Composer`_ anywhere onto your local computer.
And install RollerworksSearch with Composer by running the following:

.. code-block:: bash

    $ php composer.phar require "rollerworks/search:^2.0"

From the directory where your ``composer.json`` file is located.

Now, Composer will automatically download all required files, and install them
for you. After this you can start integrating RollerworksSearch with your application.

.. note::

    All code examples assume you are using the class auto-loader provided by Composer.

    .. code-block:: php

        require 'vendor/autoload.php';

        // ...

Extensions
----------

The ``rollerworks/search`` core library itself does not provide any mechanise
for searching in a storage engine (like Doctrine or ElasticSearch). Instead they
are provided as separate extensions you can easily install.

Framework integration libraries (provided by Rollerworks) are designed to provide
a clear-cut and ready to use solution. Whenever you install an addition extension,
the integration automatically enables the support for it.

.. note::

    Only extensions provided by Rollerworks are fully integrated, for extensions
    provided by third party developers you need to enable these manually.

Framework integration
---------------------

RollerworksSearch provides a fully featured integration for:

* The :doc:`Symfony Framework <integration/symfony>`,
* :doc:`Doctrine DBAL and ORM <integration/symfony_bundle>`.

And support for ElasticSearch, Silex, Zend Framework, and Laravel coming soon.

Further reading
---------------

* :doc:`Using the SearchProcessor <processing_searches>`
* :doc:`Composing SearchConditions <composing_search_conditions>`

.. _`Composer`: http://getcomposer.org/
.. _`downloading Composer`: https://getcomposer.org/download/
