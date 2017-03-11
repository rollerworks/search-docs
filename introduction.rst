Introduction
============

In this chapter we will take a short tour of the various components, which put
together form RollerworksSearch as a whole. You will learn key terminology
used throughout the rest of this manual and will gain an understanding of the
classes you will work with.

.. tip::

    RollerworksSearch can be used as both a processing library for search requests,
    and as a software client to compose search-conditions that will be provided to
    a third-party (web) API that uses RollerworksSearch for processing.

    Some of the actual functionality is provided by extensions that you need to
    install in addition to the core library. Installing these extensions as simple,
    as can be. See the :doc:`installation <installing>` chapter for more details.

Code Samples
------------

Code examples are used throughout the manual to clarify what is written in text.
They will sometimes be usable as-is, but they should always be taken as
outline/pseudo code only.

A code sample will look like this:

.. code-block:: php

    class AClass
    {
        ...
    }

    // A Comment
    $obj = new AClass($arg1, $arg2, ... );

    /* A note about another way of doing something
    $obj = AClass::newInstance($arg1, $arg2, ... );
    */

The presence of 3 dots ``...`` in a code sample indicates that code has been excluded,
for brevity. They are not actually part of the code.

Multi-line comments are displayed as ``/* ... */`` and show alternative ways
of achieving the same result.

You should read the code examples given and try to understand them. They are
kept concise so that you are not overwhelmed with information.

SearchCondition
---------------

Each search operation starts with a SearchCondition.
A SearchCondition defines a structure of requirements (conditions) for one or
more fields. Secondly a SearchCondition has a configuration set know as a **FieldSet**.

The final group/value structure of a condition can be as complex as needed, all values
in a condition are provided in the "Model" format (eg. a ``DateTime`` object) of the field.

A :class:`Rollerworks\\Component\\Search\\Value\\ValuesBag` can hold any value-type,
including simple values, ranges, comparisons and pattern matchers
(eg. starts with/contains, etc).

Transforming of an input structure and values is handled using an Input processor,
but semantically composing a new SearchCondition is also possible.

.. code-block:: php

    $condition = SearchConditionBuilder::create($fieldSet)
        ->field('customer')
            ->addSimpleValue(2)
            ->addSimpleValue(5)
        ->end()
    ->getSearchCondition();

See also: :doc:`composing_search_conditions`.

FieldSet
--------

A :class:`Rollerworks\\Component\\Search\\FieldSet` (or FieldSet configuration)
holds a set of **SearchField**'s, and optionally, a set-name for internal usage.

The set name can be anything but usually equals the class-name of the **FieldSetConfigurator**
that build the FieldSet configuration.

Normally you would create a FieldSet based on a subject-relationship.
For example invoice, order, news items, etc.

.. note::

    Each search field works independent from a FieldSet and may be reused in other FieldSet's.
    But the field's name must be unique within a FieldSet.

SearchField
-----------

A :class:`Rollerworks\\Component\\Search\\Field\\FieldConfig` consists of a name
a type (FieldType), resolved options and optionally a data-transformer for transforming
an input value to a model format (eg. string ``2017-03-6`` to a ``DateTime`` object).

A SearchField can be compared to a form-field or database column.

FieldTypes are really flexible and allow endless extensibility, they were heavily
inspired on the Symfony Form component.

.. tip::

    A ``FieldSet`` can also be created by using the ``FieldSetBuilder``,
    which provides a much simpler interface.

Field Type
~~~~~~~~~~

Field types are used for configuring SearchFields using reusable types
that make extensions as advanced as possible and reducing the amount of code
you have to duplicate. And making sure your fields are consistent.

You don't extend a Field type by extending the PHP class, but by using
an advanced field building system. Each type can have multiple extensions.

.. note::

    Build-in types are provided by the CoreExtension.

    You are free create your own field types for more advanced use-cases.
    See :doc:`cookbook/type/index` for more information.

FieldSetConfigurator
--------------------

A FieldSetConfigurator helps with making FieldSet's reusable and keeping your FieldSet
configurations in a logical place. Each configurator holds the configuration for single
FieldSet.

.. code-block:: php

    namespace Acme\Search\FieldSet;

    use Rollerworks\Component\Search\Extension\Core\Type\IntegerType;
    use Rollerworks\Component\Search\FieldSetBuilder;
    use Rollerworks\Component\Search\FieldSetConfigurator;

    final class UserFieldSet implements FieldSetConfigurator
    {
        public function buildFieldSet(FieldSetBuilder $builder)
        {
            $builder->add('id', Type\IntegerType::class);
            $builder->add('name', Type\TextType::class);
        }
    }

Loading a FieldSetConfigurator is done by referencing the fully qualified
class-name (FQCN) (eg. ``Acme\Search\FieldSet\UserFieldSet``).

.. tip::

    A Configurator is automatically initialized on first usage, if your
    configurator has external dependencies you can use a PSR-11 Container
    to lazily load configurators with dependencies.

    See :doc:`creating_reusable_fieldsets` for usage.

Input Processors
----------------

While creating a new **SearchCondition** is easy you would properly,
want to provide the condition is a more user-friendly format.

RollerworksSearch comes pre-bundled with various :doc:`input processors <input>`,
for JSON, XML, and StringQuery (a powerful and user-friendly string processor).

Each processor transforms and validates the user-input to a ready
to use SearchCondition.

Exporters
---------

While the input component processes user-input to a SearchCondition.
The exporters do the opposite, transforming a SearchCondition to an exported
format. Ready for input processing.

Exporting a SearchCondition is very useful if you want to store the condition
on the client-side in either a cookie, URI query-parameter or hidden form input field.
Or if you need to perform a search operation on an external system that uses
RollerworksSearch.

Condition Optimizers
--------------------

Condition optimizers help with Removing duplicated values, normalizing overlapping and
redundant values/conditions. To produce a minimal SearchCondition for faster processing
and storage.

RollerworksSearch comes pre-bundled with the a variety of optimizers, and creating
your own is also possible.

SearchFactory
-------------

The SearchFactory forms the heart of the search system, it provides
easy access to builders, the (default) condition optimizer, and serializer.

Unless you are using a specific Framework integration, you would rather want
to use the :class:`Rollerworks\\Component\\Search\\Searches`
class which takes care of all the boilerplate of setting up a SearchFactory.

SearchConditionSerializer
-------------------------

The :class:`Rollerworks\\Component\\Search\\SearchConditionSerializer`
class functions as a helper for serializing a ``SearchCondition``.

A SearchCondition holds a ValuesGroup (with nested ValuesBags, optionally
other nested ValuesGroup objects), and also a FieldSet.

The ValuesGroup and values can be easily serialized, but the FieldSet is
more difficult. A Field can have closures, service dependencies, and is
to complex to serialize.

Instead of serializing the FieldSet the serializer stores the FieldSet name,
and when unserializing it loads the FieldSet using a :class:`Rollerworks\\Component\\Search\\FieldSetRegistry`.

.. note::

    The Serializer doesn't check if the FieldSet is actually loadable
    by the FieldSetRegistry. You must ensure the FieldSet is loadable, else
    when unserializing you get an exception.

.. caution::

    Suffice to say, never store a serialized SearchCondition in the client-side!
    The Serializer still uses the PHP serialize/unserialize functions, and due to
    unpredictable values can't provide a list of trusted classes.

    Use an Exporter to store a SearchCondition in an untrusted storage.

FieldSetRegistry
----------------

A FieldSetRegistry (:class:`Rollerworks\\Component\\Search\\FieldSetRegistry`)
allows to load a FieldSet from a registry. This can be eg. the FQCN or ServiceLocator.

The FieldSetRegistry is used when unserializing a serialized SearchCondition,
so that don't have to inject the FieldSet explicitly. But you are free to use
it whenever you find it useful.

SearchProcessor
---------------

Properly handling a Search operation requires multiple steps, you need to
process the input, handle errors (exceptions) and somehow apply the search
condition while dealing with form posts and redirects.

Not to mention caching, you don't want to process the same condition once you
know it's valid. To help with this you can use a SearchProcessor, which takes
care of all these details.

A PSR-7 compatible processor is provided at https://github.com/rollerworks/search-processor
which can be easily installed.

.. tip::

    RollerworksSearch provides a number of Framework integration
    libraries which take care of adapting the different Request formats
    when PSR-11 is not supported.

Further reading
---------------

Now that you know the basic terms and conventions it's time to get started.
Note that some extensions are provided separate while there documentation is
kept within this manual.

Depending on your usage there are a number of dedicated chapters that help you
with integrating RollerworksSearch.

First make sure you :doc:`install <installing>` RollerworksSearch, and optionally
other extensions.

* :doc:`Using the SearchProcessor <processing_searches>`
* :doc:`composing_search_conditions`
* :doc:`Symfony Framework integration <integration/symfony_bundle>`
* :doc:`Using ElasticSearch with Elastica <integration/elastic_search>` (coming soon)
* :doc:`Doctrine DBAL/ORM integration <integration/doctrine/index>`
