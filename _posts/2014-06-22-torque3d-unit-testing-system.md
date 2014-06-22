---
layout: post
title:  "The Torque 3D unit testing system - a safari"
date:   2014-06-22 20:10
tags:   programming c++ torque safaris
---

This is the first in a series of experimental posts that explore some of [Torque
3D][]'s subsystems. It's going to be written in a nearly stream-of-consciousness
style as I explore the system myself, and there'll be relatively little editing.
This is an experiment to provide more organic engine documentation. I've picked
an easy first subject, the unit testing framework, because I'm currently [replacing
it][] with [Google Test][], and figured this would be a good way for me to get to
know the extent of the code I'll be touching.

[Torque 3D]: http://torque3d.org
[Google Test]: https://code.google.com/p/googletest/
[replacing it]: https://github.com/GarageGames/Torque3D/issues/626

We'll mostly be looking at [unit/test.h][] and [unit/test.cpp][] so go ahead and
have them open and ready for your perusal.

[unit/test.h]: https://github.com/GarageGames/Torque3D/blob/feec36731ef870c36084fd08c1cd53865aa01ad4/Engine/source/unit/test.h
[unit/test.cpp]: https://github.com/GarageGames/Torque3D/blob/feec36731ef870c36084fd08c1cd53865aa01ad4/Engine/source/unit/test.cpp

## Entry point

I search for `unitTest_runTests` because that's what we call from the console to
run tests. Search for text in 'current solution'. Choose the line with a
`ConsoleFunction` definition because that defines the console function, yo. It
lives in `unit/consoleTest.cpp`. It creates a `TestRun` and calls `test` with
the two arguments, `searchString` and `skipInteractive`.

    TestRun tr;
    return tr.test(searchString, skip);

## TestRun class

`TestRun` is defined in `unit/test.h`. Everything in here is in the `UnitTesting`
namespace. This class includes members `_testCount`, `_failureCount`, etc. The
`test` method has two overloads, a public version which is called with the two
parameters above, and a private version that takes a `TestRegistry`.

### Public `test` method

The public one iterates over everything in `TestRegistry::getFirst()`, which is
evidently some static list of all unit tests the engine knows about, and calls
the private `test` on each one.

Note that the `cwd` is set to the executable or `main.cs` directory before each
call to the private `test`. The `cwd` is restored after the test run to whatever
it was before the testing started.

We then `printStats` and call `Process::provessEvents()` which has the comment
`sanity check for avoid Process::requestShutdown() called on some tests`. I think
Luis added this recently to avoid a crash. Investigte later.

It returns `!_failureCount`.

### Private `test` method

The private `test` method first does `UnitMargin::Push` which seems mysterious.
It then creates a `new UnitTest*` by calling `newTest` on the `TestRegistration`
is is passed, then calls `run` on that `UnitTest`. And then `UnitMargin::Pop`.
`_failureCount` and friends are updated from the properties of the `UnitTest`
object, which is then deleted.

    UnitMargin::Push(_Margin[0]);
    
    // Run the test.
    UnitTest* test = reg->newTest();
    test->run();
    
    UnitMargin::Pop();

Okay, what's this `UnitMargin`?

### UnitMargin struct

Oh gosh global variables everywhere. Why is there
a struct for the methods that operate on global variables? Okay it looks like
what this is actually doing is filling the `_MarginString` with spaces up to
the size of `_MarginPtr`'s first element. But `_MarginPtr` is an array so it can
remember the size of each margin added when `pop` is called. It can be nested 32
times. `_printMargin` is ised to `frwite` the `_MarginString` to `stdout`. So
basicaly it just lets us print spaces before a line. I guess we'll see where it's
used later.

### TestRegistration class

Let's see what `TestRegistration::newTest` does. 

    template<class T>
    class TestRegistration: public TestRegistry
    {
       ...
    
       virtual UnitTest* newTest() 
       { 
          return new T; 
       }

Right, so it just constructs some instance of the template type that this test
registration is of. Which must be a subclass of `UnitTest`. Let's detour for a
second to a macro back in `test.h`:

    #define CreateUnitTest(Class,Name) \
       class Class; \
       static UnitTesting::TestRegistration<Class> _UnitTester##Class (Name, false, #Class); \
       class Class : public UnitTesting::UnitTest

So the `CreateUnitTest` macro starts a clas definition for a subclass of `UnitTest`
and also creates a static `TestRegistration` object. Its constructor is actually
empty:


    TestRegistration(const char* name, bool interactive, const char *className)
       : TestRegistry(name, interactive, className) 
    {
    }

so let's go have a look at `TestRegistry`. It does a bunch of stuff including
making sure there is no conflicting test name, and then adds the new object to
`TestRegistry::_list`, which I infer is a linked list of tests that we should
be able to access using...

       static TestRegistry *_list;
    public:
       static TestRegistry* getFirst() { return _list; }

Okay, so that makes sense. Let's check out an example of a unit test, then. Oh,
but before we do, I want to point out this gem:

    friend class DynamicTestRegistration; // Bless me, Father, for I have sinned, but this is damn cool

## A unit test

To find unit tests I 'find all references' on `namespace UnitTesting`. Likely
candidates are probably anywhere that's `using namespace UnitTesting`.
`testbasictypes.cpp` sounds like an easy place to start.

### An example

    CreateUnitTest(CheckTypeSizes, "Platform/Types/Sizes")
    {
       void run()
       {
          // Run through all the types and ensure they're the right size.
    
    #define CheckType(typeName, expectedSize) \
                   test( sizeof(typeName) == expectedSize, "Wrong size for a " #typeName ", expected " #expectedSize);
    
          // One byte types.
          CheckType(bool, 1);
          CheckType(U8,   1);
          CheckType(S8,   1);
          CheckType(UTF8, 1);

Okay, fairly simple. A `UnitTest` calls its own `test` method to assert that
something should be true. In this case we're doing a bunch of size tests on the
platform type wrappers.

### An assertion

Let's have a look at `test`. This is in `test.h`, in the definition of `UnitTest`:

    bool test(bool a,const char* msg) {
       dFetchAndAdd( _testCount, 1 );
       if (!a)
          fail(msg);
       _lastTestResult = a;
       return a;
    }

Ooh. That's interesting. What's `dFetchAndAdd`? Apparently it lives in
`platform/platformIntrinsics.visualc.h` which sounds platform-specific, but isn't
in `platformWin32/` for some reason.

    inline void dFetchAndAdd( volatile S32& ref, S32 val )
    {
       _InterlockedExchangeAdd( ( volatile long* ) &ref, val );
    }

Okay, and `_InterlockedExchangeAdd` is some Windows API function I think. I'm
going to intuit that it's for atomically incrementing a memory location, so we
avoid race conditions. Also, this comment is pertinent:

    // NOTE: These do not return the pre-add value because
    // not all platforms (damn you OSX) can do that.

Interesting. Wait, can we not do that ourselves? Since we know the pre-add value?
Oh no, of course we don't, because this is a volatile situation where the _actual_
value before we add to it may be different to the one we read on the line before
we write. For example, if we were to do this:

       long old = ref;
       _InterlockedExchangeAdd( ( volatile long* ) &ref, val );
       return old;

The value of memory in `ref` may change between the first and second line. Duh.
So we need to rely on platform intrinsic methods to give us that information, but
apparently we can't rely on _all_ platform's intrinsics to do that. Shame.

Okay, wait wait. So what this means is that unit tests are potentially
multithreaded? Within a single test, I mean, because each `UnitTest` object is
self-contained. But one `UnitTest` could spawn multiple threads that each call
`test` and hopefully that will result in a consistent result. Cool.

### Another example

I'm going to try to find a test that does some setup and teardown, or uses a
fixture in some way. That seems helpful. Oh, but I've found this instead in
`platformWin32/winWindow.cpp`:

    S32 PASCAL WinMain( HINSTANCE hInstance, HINSTANCE, LPSTR lpszCmdLine, S32)
    {
    #if 0
       // Run a unit test.
       StandardMainLoop::initCore();
       UnitTesting::TestRun tr;
       tr.test("Platform", true);
    #else

Ha ha. Okay, a unit test example, that's what I was looking for. Yes. Hmm. Ooh.
Doing a 'find all references' on `UnitTest` turns up a bunch of results in the
Google Test library files. Whoops. I suspect nobody is really using fixtures.
Here's a test that stores some instance data:

    CreateUnitTest(TestingProcess, "Journal/Process")
    {
       // How many ticks remaining?
       U32 _remainingTicks;

       void process()
       {
          ...
          _remainingTicks--;
       }
    
       void run()
       {
          // We'll run 30 ticks, then quit.
          _remainingTicks = 30;
    
          // Register with the process list.
          Process::notify(this, &TestingProcess::process);

Okay, fair enough. I assume that if you defined a constructor (in this case,
`TestingProcess()`) then it would be called in the appropriate place? I.e. when
the test instance is created way back up in `TestRun::test` (the private one).

## Existing tests

Okay, that's cool. So what sort of coverage do we have with tests? I do a 'find
all references' on `CreateUnitTest`. This should be fun. 153 results found. Hm.

First up are some tests for the unused component system. Then some miscellaneous
ones, the basic type tests I listed some of above, and then a very interesting
test: `TestDefaultConstruction` in `unit/tests/testDefaultConstruction.cpp`.

        for( AbstractClassRep* classRep = AbstractClassRep::getClassList();
             classRep != NULL;
             classRep = classRep->getNextClass() )
        {
           // Create object.
           ConsoleObject* object = classRep->create();
           test( object, avar( "AbstractClassRep::create failed for class '%s'", classRep->getClassName() ) );
           if( !object )
              continue;

This iterates over every class exposed to the console (i.e. every class you can
use from scripts) and tries to create one. This is interesting because it indicates
that every object exposed to scripts should be valid with no members. I think.
Let's see what `create` does, and hopefully verify this.

There are two subclasses of `AbstractClassRep` - the `Concrete` variety and the
`Dynamic` variety. They both do the same thing:

    /// Wrap constructor.
    ConsoleObject* create() const { return new T; }

Okay, easy enough. And though it wont' matter, let's see which one is usual. My
bet is on `Concrete`. If we text search for `#define DECLARE_CONOBJECT` we get
this:

    #define DECLARE_CONOBJECT( className )                   \
       DECLARE_CLASS( className, Parent );                   \
       static S32 _smTypeId;                                 \
       static ConcreteClassRep< className > dynClassRep;     \
       static AbstractClassRep* getStaticClassRep();         \
       ...

Okay, so things are mostly of the `Concrete` variety of class representation. If
you're confused about what all this machinery is for, I direct you to the comment
in `console/consoleObject.h`:

> Many of Torque's subsystems, especially network, console, and sim,
> require the ability to programatically instantiate classes. For instance,
> when objects are ghosted, the networking layer needs to be able to create
> an instance of the object on the client. When the console scripting
> language runtime encounters the "new" keyword, it has to be able to fill
> that request.
> 
> Since standard C++ doesn't provide a function to create a new instance of
> an arbitrary class at runtime, one must be created. This is what
> AbstractClassRep and ConcreteClassRep are all about. They allow the registration
> and instantiation of arbitrary classes at runtime.

Anyway, let's give this a shot. I'm running Torque, opening the console and
entering:

    unitTest_runTests("Console/DefaultConstruction", true);

Ah. Aha. I see.

    SFXSource::onAdd() - no description set on source 4148 ((null))
    ** Failed: registerObject failed for object of class 'SFXSource'
    SFXSource::onAdd() - no description set on source 4149 ((null))
    ** Failed: registerObject failed for object of class 'SFXSound'
    SFXParameter::onAdd - 4152 ((null)): parameter object does not have a name
    ** Failed: registerObject failed for object of class 'SFXParameter'
    ShapeBase::onAdd - no datablock on shape 4495:Item ((null))
    ** Failed: registerObject failed for object of class 'Item'
    Debris::onAdd - Fail - No datablock
    ** Failed: registerObject failed for object of class 'Debris'
    ShapeBase::onAdd - no datablock on shape 4505:Camera ((null))
    ** Failed: registerObject failed for object of class 'Camera'
    ShapeBase::onAdd - no datablock on shape 4507:AIPlayer ((null))
    ** Failed: registerObject failed for object of class 'AIPlayer'

And so on. Well that's amusing. I guess this test... isn't supposed to pass? I
guess it verifies that the engine doesn't, you know, crash or anything. But I'm
fairly certain that unit tests should be designed so that _passing them_ means
success, not just _not crashing_ while running them.

Okay. That's a slight diversion. Where were we?

There are tests for files (one of which includes a 5 second sleep...), a smattering
of maths tests, packet and networking tests (that call out to `garagegames.com`),
some tests of utilities like `String` and the defunct component interface, lots
of thread tests including stress tests, even more thread tests, tests for `Vector`,
and interestingly some tests of the window manager. Oh, and of the zip filesystem.
Hopefully they can help disambiguate whether T3D has zip filesystem support...
Anyway, that's not an exhaustive list, but those are the major ones.

## The end

I hope that was at least slightly illuminating, rather than just confusing. I also
apologise in retrospect for any attitude that crept into my analysis. I tried to
keep it factual!
