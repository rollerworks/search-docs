Value and Column Conversions
============================

Conversions for Doctrine DBAL are similar to the DataTransformers
used for transforming user-input to a normalized data format. Except that
the transformation happens in a single direction, and uses "model" values.

So why are are they useful? The power of relational databases is
storing complex data structures, and allow to retrieve these records
into a transformed and combined result.

For example the "birthday" field type accepts both an actual (birth)date
or an age value like "9". But you're not going to store the actual age,
as this would require constant updating. Instead you calculate the age by
the date that is stored in the database (you transform the data). This
transformation process is called a conversion.

.. note::

    Some values don't have to be converted if the Doctrine DBAL Type
    is already able to process the value as-is.

    For example a ``DateTime`` object can be safely used *if* the mapping-type
    is properly configured with "date" or "datetime" as column type.

There are two types of converters. Which can be combined together in one
single class. Conversions can happen at the column and/or value.
So you can really utilize the power of your SQL queries.

.. note::

    Unlike DataTransformers you`re limited to *one* converter per search
    field. So setting an new convert (of the same) will overwrite previous
    one.

Conversions are registered by setting the conversion object on the
``SearchField`` by using the ``configureOptions`` method of the field type.
Using:

   .. code-block:: php

       public function configureOptions(OptionsResolver $resolver)
       {
           $resolver->setDefaults(
               array('doctrine_dbal_conversion' => new MyConversionClass())
           );
       }

.. tip::

    The ``doctrine_dbal_conversion`` option also accepts a ``Closure`` object
    for lazy initializing, the closure is executed when creating the
    query-generator.

Before you get started, it's important to know the following about converters:

#. Converters should be stateless, meaning they don't remember anything
   about there operations. This is because the calling order of converter methods
   is not predictable and converters are only executed during the
   generation process, so using a cached result will not execute them.
#. Each converter method receives a `:class:`Rollerworks\\Component\\Search\\Doctrine\\Dbal\\ConversionHints`
   object which provides access to the used database connection, SearchField
   configuration, column and optionally the conversionStrategy.
#. The ``$options`` array provides the options of the SearchField.

.. tip::

    If you use SQLite and need to register an user-defined-function (UDF)
    you can register a ``postConnect`` event listener at the Doctrine EventsManager
    to register the function.

    See `SqliteConnectionSubscriber.php`_ for an example.

ColumnConversion
----------------

A ColumnConversion transforms the provided column to an SQL
part like a sub-query or user-defined functional call.

The :class:`Rollerworks\\Component\\Search\\Doctrine\\Dbal\\ColumnConversion`
requires the implementation of one method that must return the column
or anything that can be used as a replacement.

This example shows how to get the age of a person in years from their date
of birth. In short, the ``u.birthdate`` column is converted to an actual
age in years.

.. code-block:: php
    :linenos:

    namespace Acme\User\Search\Dbal\Conversion;

    use Rollerworks\Component\Search\Doctrine\Dbal\ConversionHints;
    use Rollerworks\Component\Search\Doctrine\Dbal\ColumnConversion;

    class AgeConversion implements ColumnConversion
    {
        public function convertColumn($column, array $options, ConversionHints $hints): string
        {
            if ('postgresql' === $hints->connection->getDatabasePlatform()->getName()) {
                return "TO_CHAR('YYYY', AGE($column))";
            } else {
                // Return unconverted
                return $fieldName;
            }
        }
    }

The ``u.birthdate`` column reference is wrapped inside two function calls,
the first function converts the date to an Interval and then the second function
extracts the years of the Interval and then casts the extracted years to a
integer. Now you easily search for users with a certain age.

.. _value_conversion:

ValueConversion
---------------

A ValueConversion converts the provided value to an SQL part like a sub-query
or user-defined functional call.

The :class:`Rollerworks\\Component\\Search\\Doctrine\\Dbal\\ValueConversion`
requires the implementation of one method that must return the value
as SQL query-fragment (with proper escaping and quoting applied).

.. warning::

    The ``convertValue`` method is required to return an SQL query-fragment
    that will be applied as-is!

    Be extremely cautious to properly escape and quote values, failing to do
    this can easily lead to a category of security holes called SQL injection,
    where a third party can modify the executed SQL and even execute their own
    queries through clever exploiting of the security hole!

    The only only save way to escape and quote a value is with:

    .. code-block:: php

        $hints->connection->quote($value);

    Don't try to replace the escaping with your own implementation
    as this may not provide a full protection against SQL injections.

    One minor exception is using integer values with SQLite, because
    quoting these values don't work as expected. Make sure the value is integer
    and nothing else!

One of these values is Spatial data which requires a special type of input.
The input must be provided using an SQL function, and therefor this can not be done
with only PHP.

This example describes how to implement a MySQL specific column type called Point.

The point class:

.. code-block:: php
    :linenos:

    namespace Acme\Geo;

    class Point
    {
        private $latitude;
        private $longitude;

        /**
         * @param float $latitude
         * @param float $longitude
         */
        public function __construct($latitude, $longitude)
        {
            $this->latitude  = $latitude;
            $this->longitude = $longitude;
        }

        /**
         * @return float
         */
        public function getLatitude()
        {
            return $this->latitude;
        }

        /**
         * @return float
         */
        public function getLongitude()
        {
            return $this->longitude;
        }
    }

And the GeoConversion class:

.. code-block:: php
    :linenos:

    namespace Acme\Geo\Search\Dbal\Conversion;

    use Acme\Geo\Point;
    use Rollerworks\Component\Search\Doctrine\Dbal\ConversionHints;
    use Rollerworks\Component\Search\Doctrine\Dbal\SqlValueConversionInterface;

    class GeoConversion implements ValueConversion
    {
        public function convertValue($input, array $options, ConversionHints $hints)
        {
            if ($value instanceof Point) {
                $value = sprintf('POINT(%F %F)', $input->getLongitude(), $input->getLatitude());
            }

            return $value;
        }
    }

.. note::

    Alternatively you can choose to create a custom Type for Doctrine DBAL.
    See `Custom Mapping Types`_ in the Doctrine DBAL manual for more information.

    But doing this may cause issues with certain database vendors as the generator
    doesn't now the value is wrapped inside a function and therefor is unable
    to adjust the generation process for better interoperability.

Using Strategies
----------------

You already know it's possible to convert columns and values
to a different format and that you can wrap them with SQL statements.
But there is more.

Converting columns and/or values will work in most situations, but what if
you need to work with differing values like the birthday type, which accepts
both dates and integer (age) values? To make this possible you need to add
conversion-strategies. Conversion-strategies are based on the `Strategy pattern`_
and work very simple and straightforward.

A conversion-strategy is determined by the given value, each mapping
and value gets a determined strategy assigned. If there is no strategy
(which is the default) ``null`` is used instead. Then each strategy is
applied per field and it's values, meaning that a field and the related
values are grouped together.

Say you have the following values-list for the birthday type: ``2010-01-05, 2010-05-05, 5``.
The first two values are dates, but third is an age. With the conversion
strategy enabled the system will process the values as follow;

    Dates are assigned strategy-number 1, integers (ages) are assigned with
    strategy-number 2.

    So ``2010-01-05`` and ``2010-05-05`` get strategy-number 1.
    And the ``5`` value gets strategy-number 2.

    Now when the query is generated the converter's methods receive the strategy
    using the ``conversionStrategy`` property of the ``ConversionHints``, which
    helps to determine how the conversion should happen.

    But there is more to this idea, as the values don't need any SQL logic
    for the value conversion the generator can use the ``IN`` statement to
    group values of the same strategy together.

    So in the end you will have something like this:

    .. code-block:: sql

        (((u.birthday IN('2010-01-05', '2010-05-05') OR search_conversion_age(u.birthday) IN(5))))

Implementing conversion-strategies
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

To make your own conversions support strategies you need to
implement the :class:`Rollerworks\\Component\\Search\\Doctrine\\Dbal\\StrategySupportedConversion`
interface and the ``getConversionStrategy`` method.

.. note::

    If your conversion supports both the column and value conversions
    then both conversion methods will receive the determined strategy.

The following example uses a simplified version of AgeConversion class already
provided by RollerworksSearch.

.. code-block:: php
    :linenos:

    use Doctrine\DBAL\Types\Type as DBALType;
    use Rollerworks\Component\Search\Doctrine\Dbal\ConversionHints;
    use Rollerworks\Component\Search\Doctrine\Dbal\StrategySupportedConversion;
    use Rollerworks\Component\Search\Doctrine\Dbal\ColumnConversion;
    use Rollerworks\Component\Search\Doctrine\Dbal\ValueConversion;
    use Rollerworks\Component\Search\Exception\UnexpectedTypeException;

    /**
     * AgeDateConversion.
     *
     * The chosen conversion strategy is done as follow.
     *
     * * 1: When the provided value is an integer, the DB-value is converted to an age.
     * * 2: When the provided value is an DateTime the input-value is converted to an date string.
     * * 3: When the provided value is an DateTime and the mapping-type is not a date
     *      the input-value is converted to an date string and the DB-value is converted to a date.
     */
    class AgeDateConversion implements StrategySupportedConversion, ColumnConversionInterface, ValueConversion
    {
        public function getConversionStrategy($value, array $options, ConversionHints $hints)
        {
            if (!$value instanceof \DateTime && !ctype_digit((string) $value)) {
                throw new UnexpectedTypeException($value, '\DateTime object or integer');
            }

            if ($value instanceof \DateTime) {
                return $hints->field->getDbType()->getName() !== 'date' ? 2 : 3;
            }

            return 1;
        }

        public function convertColumn($column, array $options, ConversionHints $hints): string
        {
            if (3 === $hints->conversionStrategy) {
                return $column;
            }

            if (2 === $hints->conversionStrategy) {
                return "CAST($column AS DATE)";
            }

            $platform = $hints->connection->getDatabasePlatform()->getName();

            switch ($platform) {
                case 'postgresql':
                    return "to_char(age($column), 'YYYY'::text)::integer";

                default:
                    throw new \RuntimeException(
                        sprintf('Unsupported platform "%s" for AgeDateConversion.', $platform)
                    );
            }
        }

        public function convertValue($value, array $options, ConversionHints $hints)
        {
            if (2 === $hints->conversionStrategy || 3 === $hints->conversionStrategy) {
                return DBALType::getType('date')->convertToDatabaseValue(
                    $value,
                    $hints->connection->getDatabasePlatform()
                );
            }

            return (int) $value;
        }
    }

That's it, your conversion is now ready for usage.

.. caution::

    A strategy is expected to an integer, the are however no technical
    limitations to enforce this. The strategy must be at least a scalar value.

    When the returned strategy is an integer as string ``'5'``,
    the final strategy will be ``5`` as an actual integer.

Testing Conversions
-------------------

To test if the conversions work as expected your can compare the generated,
SQL with what your expecting, however there's no promise that the SQL
structure will remain the same for the future releases.

The only way to ensure your conversions work is actually run it against an
actual database with existing records.

.. _`SqliteConnectionSubscriber.php`: https://github.com/rollerworks/rollerworks-search-doctrine-dbal/blob/master/src/EventSubscriber/SqliteConnectionSubscriber.php
.. _`Custom Mapping Types`: http://docs.doctrine-project.org/projects/doctrine-dbal/en/latest/reference/types.html#custom-mapping-types
.. _Strategy pattern: http://en.wikipedia.org/wiki/Strategy_pattern
