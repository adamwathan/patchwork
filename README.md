# Patchwork

### Version 1.0.0

An implementation of [monkey patching](http://en.wikipedia.org/wiki/Monkey_patching) for PHP, written in pure userland PHP 5.3 code and available under the terms of [the MIT license](http://www.opensource.org/licenses/mit-license.php).

## Introduction

Being able to **redefine a function or a method at runtime** is something that would make many testing tasks noticeably easier, especially those involving hardly testable code. Function redefinition, which is a particular kind of [monkey patching](http://en.wikipedia.org/wiki/Monkey_patching), is a very simple answer to such testability obstacles as singletons or dependencies on static methods. However, in the PHP world, it is a not an easy thing to do without the help of non-standard core extensions (like [runkit]() or [php-test-helpers]()).

Nevertheless, an userland implementation is still possible, albeit requiring us to resort to such dirty tricks as code preprocessing. These "tricks" are lacking in performance and general reliability, so they have no place in production code, but they should generally be plausible in testing environments.

This brings us to the aim of Patchwork: providing a pure userland implementation of runtime function redefinition for use in various kinds of software testing.

Unfortunately, this implementation has a very grave limitation: it can **only** be used to redefine **user-defined** functions and methods.

## Terminology

From this point on, this readme does not refer to the functionality of Patchwork as "function redefinition", because "patching" would be a more technically correct term. This is mainly because of the possibility to apply multiple patches to a single function and remove them when needed, which hardly qualifies as "redefinition". However, these two terms are still very closely related and often interchangeable.

## Getting Started

To start using Patchwork, we have to include `Patchwork.php`:

    require __DIR__ . "/patchwork/Patchwork.php";
    
All functions and methods that may need to be patched should be defined **after** this step, and **not in the same file** from which `Patchwork.php` has been included.

## Usage

### Applying Patches

To apply a patch to a function or a method, we use `Patchwork\patch`:

    function patchingWorks()
    {
        echo "It does not.";
        return false;
    }

    # Returns FALSE and prints "It does not."
    patchingWorks();
    
    Patchwork\patch("patchingWorks", function()
    {
        return true;
    });
    
    # Returns TRUE without printing anything
    patchingWorks();

Like in the example above, returning a value from a patch makes the patched function return it as well, before any of the original statements inside it are executed, so it can be said that in such cases the patched function does not run at all. This occurs even when the return is implicit:

    Patchwork\patch("patchingWorks", function()
    {
        echo "It does.";
        # Returns NULL implicitly
    });
    
    # Returns NULL and prints "It does.";
    patchingWorks();

### Handling Methods

In both of the above examples, we are patching a global function. However, we may also do that to a method, be it instance or static:

    Patchwork\patch("Database::getInstance", function() use ($fakeDb)
    {
        return $fakeDb;
    });
    
    Patchwork\patch(array($registry, "getDatabase"), function() use ($fakeDb)
    {
        return $fakeDb;
    });
    
If `Database::getInstance` were actually an instance method (even though that would make no sense in this example), the patch would still work. It should also be noted that this behavior allows us to apply patches to all instances of a class.

Another thing to note is that `Patchwork\patch` does not obey polymorphism. If `Database` had a sub-class, then our patch would have no effect on its `getInstance` method.

### Removing Patches

A function can also be "unpatched" when the patch is no longer needed:

    $handle = Patchwork\patch("Database::getInstance", "getFakeDatabaseInstance");
    Patchwork\unpatch($handle);

### Applying Multiple Patches

Although that is rarely necessary, any number of patches can be applied to the same function or method:

    function aFunctionCanHaveMultiplePatches()
    {
        echo "It cannot.";
        return false;
    }
    
    Patchwork\patch("aFunctionCanHaveMultiplePatches", function()
    {
        echo "Perhaps it can... ";
        return null;
    });
    
    Patchwork\patch("aFunctionCanHaveMultiplePatches", function()
    {
        echo "It really can!";
        return true;
    });
    
    # Prints "Perhaps it can... It really can!" and returns TRUE
    aFunctionCanHaveMultiplePatches();
    
In this example, the final return value is `true`, because the patch that returns it is the last one to run.

### Accessing the Arguments

If a function receives arguments, its patches receive them too:

    function add($a, $b, &$result)
    {
        throw new Exception("Not implemented");
    }
    
    Patchwork\patch("add", function($a, $b, &$result)
    {
        $result = $a + $b;
    });

### Skipping Filters

A patch is not required to yield a result. Instead, it may ask to be skipped.

When a patch is skipped, Patchwork acts as if it did not exist at all. This means that if a function has no other patches applied to it, or if all of them are skipped, then the patched function runs as usual:

    Patchwork\patch("HashTable::offsetGet", function($offset) 
    {
        if ($offset == "foo") {
            return "bar";
        }
        Patchwork\skip();
        echo "This will never be printed";
    });
    
    $ht = new HashTable;
    
    # Returns "bar"
    $ht["foo"];
    
    # Runs the actual offsetGet method
    $ht["???"];
    

Note that **every** call to `Patchwork\skip` throws an exception, which immediately terminates the patch which it was called from.

### Inspecting the Call Stack

A function call can be said to have many "properties", including the arguments, the function name, the class name, the object which received the call, or the file from which the call was made.

As already demonstrated, the first of these properties, that is, the arguments, can be accessed from a patch in the "traditional" way. However, it would not be the same if we were to retrieve the object on which the patched method was called. We would have to inspect the call stack manually, for example, by using `debug_backtrace`. Alternatively, we can use these shortcuts provided by Patchwork:

    Patchwork\patch("HashTable::offsetGet", function($offset)
    {
        # Returns a full stack trace, as returned by debug_backtrace
        # (beginning with the call to offsetGet)
        Patchwork\traceCall();
        
        # Returns the top stack frame (representing the call to offsetGet)
        Patchwork\getCallProperties();
        
        # Extracts a single property from the top stack frame
        $ht = Patchwork\getCallProperty("object");
        
        # INFINITE RECURSION!
        return $ht->offsetGet($offset);
    });
    
For the complete list of properties that can be retrieved using `getCallProperty`, refer to the documentation for [debug_backtrace]().

### PHPUnit Integration

Internally, Patchwork uses global variables to store its state (parts of which are not serializable), so it interferes with the global state backup feature of PHPUnit. To solve this problem, we can blacklist these variables manually, or we can also use a ready-made specialization of the `PHPUnit_Framework_TestCase` class, named `Patchwork\TestCase`.

Additionally, this specialization provides a `patch` method, which can be used to apply patches that only last for the lifetime of a single test method:

    function getInteger()
    {
        return 0;
    }

    class Test extends Patchwork\TestCase
    {
        private $integersRequested = 0;
        
        function testSomething()
        {
            $this->patch("getInteger", array($this, "mimicIntegerGetter"));
            $this->assertEquals(1, getInteger());
            $this->assertEquals(1, $this->integersRequested);
        }
        
        function testThePatchIsNoLongerInEffect()
        {
            $this->assertEquals(0, getInteger());
        }
        
        function mimicIntegerGetter()
        {
            $this->integersRequested++;
            return 1;
        }
    }

## Final Notes

If you happen to discover any bugs in Patchwork, please do not hesitate to [report them](http://github.com/antecedent/patchwork/issues). This also applies to any grammatical errors, factual discrepancies or invalid code samples that may have appeared in this document. And, of course, suggestions are welcome as well.

Thank you for your interest!
