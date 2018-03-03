---
title: 'Design by Contract'
taxonomy:
    category:
        - docs
---

In addition to AOP, appserver also offers [Design-by-Contract](http://en.wikipedia.org/wiki/Design_by_contract) out of the box, which is another interesting architectural approach.

First introduced by Bertrand Meyer in combination with his design of the [Eiffel programming language](https://en.wikipedia.org/wiki/Eiffel_%28programming_language%29),
Design by Contract allows you to define formal, precise and verifiable interface specifications of
software components.

Design by Contract extends the ordinary definition of classes and interfaces by allowing the possibility of defining so called `contracts` (hence the name).
These contracts allow every public method to specify `preconditions`, which they need to function, and one or more `post-conditions` to ensure the method results are correct.
Additionally, one can state so called `invariants`, which define an atomic state of integrity for the structure itself.
Such contract elements are checked during program runtime and allow for true domain logic [fail-fast](https://en.wikipedia.org/wiki/Fail-fast) programming.

> Besides fail-fast behavior, which would be an exception, our implementation offers [PSR-3](https://github.com/php-fig/fig-standards/blob/master/accepted/PSR-3-logger-interface.md) compatible logging, as a reaction on contract breach.

## What can be contracted?

### Type hints
The most basic type of contract is one used all the time and is already integrated into most IDEs: annotation based type hinting.
If following [common syntax](http://phpdoc.org/docs/latest/guides/types.html) we can make use of these annotations to enforce strong typing at method call and return point.

> We currently support the `@param` and `@return` annotations as type hints (scalar and class/interface
> based), including special features like `typed arrays` using e.g. `\stdClass[]`.

Review the following example.

```php
<?php

/**
 * Add string to storage. Will return resulting storage size
 *
 * @param string $string String to add
 *
 * @return integer
 */
public function addString($string)
{
    // ...
}
```

As stated in the comments, this example method will add a string to a storage and return the resulting storage size. Using Design by Contract, both the parameter type and the return type will be enforced.

This is possible for scalar types as well as for complex ones and offers an addition for `typed arrays`.

Review the following example.

```php
<?php

/**
 * Will return an array of all strings currently stored
 *
 * @return string[]
 */
public function getStrings()
{
    // ...
}
```

### Execution constraints

A more complex use case is writing out execution constraints within contracts.
This can be done using the mentioned condition clauses.

The following example expands on the `addString` code snippet above.

```php
<?php

/**
 * Add string to storage. Will return resulting storage size
 *
 * @param string $string String to add
 *
 * @return integer
 *
 * @Ensures("$this->stringExists($string)")
 */
public function addString($string)
{
    // ...
}
```

The resulting behavior doesn't only check for basic parameter and return type, but also ensures that the passed string variable is added to the storage (assuming the existence of a `stringExists` method).

This constraint on the execution of the `addString` method is, by now, solely based on its implementation and can not be changed during runtime. But, we can also add a constraint through the method parameters as shown below.

```php
<?php

/**
 * Add a string shorter than six characters to storage. Will return resulting storage size
 *
 * @param string $string Short string to add
 *
 * @return integer
 *
 * @Requires("strlen($string) <= 6")
 *
 * @Ensures("$this->stringExists($string)")
 */
public function addShortString($string)
{
    // ...
}
```

Assuming that our storage can only handle strings equally long or shorter than six characters, we can constrain the length of our input parameter.
The precondition constraint guards our storage by enforcing proper string length and at the same time assures that the string will be stored, when it does have the proper length.

A constraint based contract for the execution of a method is born.

### State validity

As the Design by Contract principle is based on the concept of contracts within the business world we have to consider another element.
The already mentioned `invariant` as a global constraint to possibilities of interaction.

In the financial market this would be laws about taxes and contract clauses. In the programming world, this is a valid state an object has to maintain during its lifecycle.

> The invariant describes a valid state an object must have at defined moments in its lifetime.

For our implementation these defined moments are:

* After its construction (leaving the `__construct` method)
* Before any read/write access to a public property
* After any write access to a public property
* Before and after any call to a public method

These invariants are stated using the @Invariant annotation.
Below is a basic example continuing with our StringStorage scenario:

```php
<?php

/**
 * Class which is used to store string literals
 *
 * @Invariant("$this->onlyContainsStrings()")
 */
class StringStorage
{
    // ...
}
```

The annotation above would result in a constraint for the content of the storage (given `onlyContainsStrings` exists) that checks that there are only strings within the storage on every relevant public access to an object of the class `StringStorage`.

## Usage

The examples above already give a glimpse of how to use Design by Contract within your code.
Here, some are some more rules and guidelines.

### Annotation syntax

First of all, syntax.
The syntax of an annotation is very simple and follows `Doctrine` syntax and is composed as follows:

```
@<AnnotationName>("<Constraint>")
```

The constraint itself MUST be a valid conditional PHP expression which evaluates to the boolean `true` or `false` or any value castable to a boolean.

### Scope

The code within the constraints can make use of variables and properties of the containing structures based on their specific scope.
These constraint scopes are listed below:

| Annotation   | Scope       | Description                                                                         |
| -------------| ------------| ------------------------------------------------------------------------------------|
| `@Requires`  | method      | Shares the scope of the method the annotation belongs to                            |
| `@Ensures`   | method      | Shares the scope of the method the annotation belongs to                            |
| `@Invariant` | structure   | Consider as called from a parameter-less private method within the structure itself |

There are two more variables, which are available to add to constraints, above and beyond the variables and properties visible within the original structure definition.

| Variable name | Usable within | Description                                                                                                        |
| --------------| --------------| -------------------------------------------------------------------------------------------------------------------|
| `$dgResult`   | `@Ensures`    | Contains the result of the method the annotation belongs to. Can be used to constrain the actual method result     |
| `$dgOld`      | `@Ensures`    | Contains a copy of the object instance before method execution, can be used to test changes to the object's state  |

## Configuration

### Where to change the configuration

Design by Contract is fully integrated into the appserver.io infrastructure's autoloading capabilities. This is due to the
heavy modifications it requires for structure definitions and the applied caching in the file system.

> To change any Design by Contract behavior, the change must be made within the class loader configuration.

The class loader configuration can be found within any `context.xml` file.

A `context.xml` file can be found:

* Globally, in the `etc/appserver/conf.d/context.xml` file.
* Locally, in the webapp specific file located within the webapp's `META-INF` directory.

The local file can override or extend the global file's configuration, so be certain of where you make any changes or additions. Any changes made to a context.xml file will either have a global effect or will only affect the webapp it is located in.

### Configuration options

Below is an example of the configuration appserver.io ships with.
The attributes of the `classLoader` element are mandatory and need not be changed for any configuration changes.
One only might change them to use alternative implementations or bootstrapping processes.

```xml
<classLoader
    name="DgClassLoader"
    interface="ClassLoaderInterface"
    type="AppserverIo\Appserver\Core\DgClassLoader"
    factory="AppserverIo\Appserver\Core\DgClassLoaderFactory">
    <params>
        <param name="environment" type="string">production</param>
        <param name="enforcementLevel" type="integer">7</param>
        <param name="typeSafety" type="boolean">1</param>
        <param name="processing" type="string">logging</param>
    </params>
    <directories>
        <directory enforced="true">/common/classes</directory>
        <directory enforced="true">/WEB-INF/classes</directory>
        <directory enforced="true">/META-INF/classes</directory>
    </directories>
</classLoader>
```

Most important are the `param` elements.
See their meaning and options as listed below.

| Param name         | Options                     | Description                                                     |
| -------------------| ----------------------------| ----------------------------------------------------------------|
| `environment`      | production&#124;development | Whether or not contracted definitions get cached. Recommended is production, as an appserver restart will clear the cache by force, therefore the cache does not hinder development   |
| `enforcementLevel` | [ 1 - 7 ]                   | A bitmask similar to the Linux user right notation. This will be used to switch on (or off) enforcement features. The bitmask can be written as `invariants post-conditions preconditions`, so the default `7` results in all contract elements being enforced whereas `1` would only enforce preconditions |
| `typeSafety`       | 0&#124;1                    | Whether or not annotation based type hints get enforced as described [above](#what-can-be-contracted) |
| `processing`       | logging&#124;exception      | Which kind of reaction a contract or type safety breach triggers. `logging` will result in a message stored within the system logger. `exception` will result in an exception of the type `\AppserverIo\Psr\MetaobjectProtocol\Dbc\ContractExceptionInterface`. Throw point is the method in which the contract was breached |

Adding a `directory` element we can specify which directory should be loaded by our class loader.
This is common configuration for every class loader type.
A feature of the appserver class loader is ability to toggle the enforcement of Design by Contract for each one of the defined directories.


If the attribute `enforced` is set to FALSE,  no enforcement will take place for the mentioned directory.
