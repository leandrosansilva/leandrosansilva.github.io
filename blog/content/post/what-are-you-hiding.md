---
title: "What Are You Hiding?"
date: 2018-09-01T10:26:03+02:00
draft: false
tags:
  - cpp
  - oop
  - programming
---

The C++ visibility model is broken by design. By visibility model I mean using the keywords *private*, *public* and *protected* to manage how objects relate to each other.

I will explain exactly how such model is broken and propose a workaround to it. I admit I am not the first one to realize that, as I have been writting C++ for just a decade, whereas there are more skilled programmers than me or with way more experience.

***TL;DR*** my "proposal" consists in keeping everything public, by always using *structs* instead of *classes*, and hiding things using convention, not *private*, so keep in mind this goes against the [Core Guidelines](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#c2-use-class-if-the-class-has-an-invariant-use-struct-if-the-data-members-can-vary-independently).

Keep in mind as well that what I am writting here is based on my experience and might not apply to everyone. I aknowledge I might be wrong. No relativisms here.

## Why the C++ inheritance model is broken

Consider the following code:

{{< highlight cpp >}}

class A {};

class B: public A {};

{{< / highlight >}}

What does it mean to inherit publicly? The mental model for such construct is to implement the classic inheritance we see in UML digrams. It means that "*B* is an *A*", that implies that B can be used anywhere A can. B specializes A.

I'll not enter in the "Inheritance vs. Composition" discussion. That's not the point right now.

The keyword *public* in the code above is used because B, being a class, by default inheriths privately. Then we use *public* to change to a public inheritance.

The issue here is that it makes no sense, for the vast majority of use cases, to inherit privately. The default inheritance case makes no sense. 

Object Oriented Programming languages created after C++ realized this issue and inherit publicly by default, not requiring the *public* keyword. Examples are Java, C# and PHP.

Structs, on the other hand, implement public inheritance by default, leading to a more legible and sensible code:

{{< highlight cpp >}}

class A {};

// Read: B is an A
struct B: A {};

{{< / highlight >}}

There's a kind of third inheritance type not related to visibility though, called ***virtual inheritance***. It took many years to learn and use it. I'll not discuss it here.

## Why the header/implementation file duality is broken in regard method visibility

***TL;DR*** I know the modules proposal will probably fix it, but it still looks too far away for most users.

A personal confession: in the past two years I programmed profissionally mostly in Objective-C, although there was still some C++. Luckly I am starting in new position where I will move back to C++. 

By doing ObjC, I could learn a lot about other approaches of handling object oriented design.

**Note**: I know about the differences between C++ and ObjC regarding method calling and memory management. I am not comparing the two languages, just trying to bring to C++ the few good bits I found in ObjC regarding design.

Objective-C, as C and C++, separates the code between definiton and implementation. Contrary to C and C++ though, is the fact that in ObjC those are not conventions, but enforced by the language. Due the fact that classes are a ***runtime*** construct, they cannot be implemented in the definition file, otherwise there would be, in runtime, multiple classes with the same name, what is a runtime error. Normally the compiler or the linker can catch them during compilation, but anyways...

It follows an example of class in ObjC. First the declation:

{{< highlight objc >}}

// MyObj.h

#import <Foundation.h>

@interface MyObj: NSObject

- (instancetype)initWithName:(NSString*)name;

- (NSUinteger)computeSomething;

@end

{{< / highlight >}}

And finally the implementation:

{{< highlight objc >}}

// MyObj.m

#import "MyObj.h"

// declare "private" members in the class, not visible in the interface
@interface MyObj () {
  NSString* _name;
}

@end

@implementation MyObj

- (instancetype)initWithName:(NSString*)name {
  // this is a "constructor" and yes, it is ugly
  if (self = [super init]) {
    _name = name;
  }

  return self;
}

// this is a "private" method, as it is not declared in the interface
- (NSUinteger)somePrivateMethod:(NSUinteger)value {
  return _name.length * value;
}

// this is a "public" method, as it is declared in the interface
- (NSUinteger)computeSomething {
  return [self somePrivateMethod:2] + [self somePrivateMethod:5];
}

@end

{{< / highlight >}}

One obvious difference between the code above and a typical C++ one is **1)** the declaration of object members in the implementation file only.
This is not possible in C++ as the compiler always need to know the object size, in case one wants to put an object on the stack.

This is possible in ObjC because every ObjC object ***must*** be heap allocated as the compiler usually is unable to know the object size.
Stack allocation is allowed only for types compatible with C.

Another difference, and the most important for my explanation, is that **2)** the class declaration contains only the public interface of the object.
That's why the keyword **@interface** is used. Not present in the interface are the private members (in this case, _name) and the private methods.

If one needs to know how to use a class, just looking at it's interface is enough [^1]. The interface is not cluttered with stuff that is not useful for the
user of that class.

Moreover, the class implementation can be changed at will, and it will not affect it's interface, not even requiring the user to recompile their code in case of those changes don't affect the public methods' signature.

Let's not see the equivalent code, but in C++:

{{< highlight cpp >}}

// MyObj.h

#include <string>

class MyObj {
public:
  MyObj(std::string);
  size_t computeSomething() const;
private:
  std::string _name;
  size_t somePrivateMethod(size_t) const;
};

{{< / highlight >}}

And finally the implementation:

{{< highlight cpp >}}

// MyObj.cpp

#include "MyObj.h"

MyObj::MyObj(std::string name): _name(name) {}

size_t MyObj::somePrivateMethod(size_t value) const {
  return _name.length() * value;
}

size_t MyObj::computeSomething() const {
  return somePrivateMethod(2) + somePrivateMethod(5);
}

{{< / highlight >}}

Comparing to the ObjC code, we see that **1)** the private member _name is declared in the header/declaration file.
There's really no way to avoid it if one wants to allocate *MyObj* on the stack [^2].

Regarding **2)**, we can see that the C++ example declares private methods in the header file. This is clutter to the user and a pain in the ass to the 
class maintainer, as changing implementation details (private methods) imply on affecting the user (requiring them to recompile their code), when no public
change has been made, as well as requiring changing between declaraion implementation files all the time.

Clearly, there's no correspondence between a class declaration and it's interface.

Moreover, one is lying to oneself when declares a method as private.

## Why One Is Lying to Oneself When Declaring A Method As Private

***TL;DR*** every private method is actually visible to anyone and prevents important optimizations.

Consider again the "MyObj" C++ class example. 

Everytime we declare, but don't define a method in a class, the compiler implicitly adds an ***extern*** modifier to it.
That means that our class declaration actually looks like this to the compiler[^3]:

{{< highlight cpp >}}

// MyObj.h

#include <string>

class MyObj {
public:
  extern MyObj(std::string);
  extern size_t computeSomething() const;
private:
  std::string _name;
  extern size_t somePrivateMethod(size_t) const;
};

{{< / highlight >}}

***NOTE:*** the behaviour bellow has been observed in both Linux and MacOS with GCC and Clang. The behaviour on Windows is/should_be different as I'll further explain.

In C++, a non virtual method is basically an ordinary function with an extra argument, ***this***, plus modifiers (const, moveable, etc).
When we (or the compiler) declare a function as **extern**, we are telling the compiler that the function is defined somewhere else,
so the compiler will be unable to inline the function, in case it could be inlined as an optimization.

Moreover, in case the class is implemented in a library (either shared on static) [^4], every method, regardless of being public or private, will have an entry in the
library symbols table, so it can be called directly, if you know its signature. Worse than that, in case it is on a shared library, it can be "overriden" at link timeby any other function with same signature!

That means that for the compiler and linker, **private** has no meaning.

On GCC and Clang, at least on Unix-like environments, every function is visible unless stated otherwise.

The only way to trully hide a function is declare it as **static** in the implementation file, or more idiomatic, put it in an annonymous namespace.
But it implies that methods need to be converted into free functions. This is most of the time a non-trivial task.

Suppose that the C++ MyObj class above is defined in a shared library. As I am using Linux, I check the symbols in the library:

``` bash

$ nm -aC libmyobj.so

[..lots of output...]
0000000000001150 T MyObj::MyObj(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >)
0000000000001150 T MyObj::MyObj(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >)
0000000000001230 T MyObj::computeSomething() const
0000000000001220 T MyObj::somePrivateMethod(unsigned long) const
[...lots of output...]

```

As you can see, there's an entry for **MyObj::computeSomething() const**. Keep that in mind.

I have then a simple program that makes use of the MyObj class:

{{< highlight cpp >}}

// myobj_user.cpp

#include "myobj.h"
#include <assert.h>

int main(int, char**) {
  const auto obj = MyObj{"house"};
  assert(obj.computeSomething() == 35);
  return 0;
}

{{< / highlight >}}

If I run this program, I expect the assertion to hold true.

Now, let's override the *MyObj::computeSomething()* my simply changing the private method *somePrivateMethod()*. To to that, I create another library called "myfakeobj", consisting on a single C++ file:

{{< highlight cpp >}}

// myfakeobj.cpp

#include <iostream>

struct MyObj {
  /*implicit extern here*/ size_t somePrivateMethod(size_t) const;
};

size_t MyObj::somePrivateMethod(size_t) const {
  std::cout << "I am doing something nasty!" << std::endl;
  return 42;
}

{{< / highlight >}}

Let's check the visible symbols in the library:

``` bash
$ nm -C libmyfakeobj.so
[...lots of output...]
00000000000011a0 T MyObj::somePrivateMethod(unsigned long) const
[...lots of output...]
```

That's what we expect. Now, let's load the symbols in *libmyfakeobj.so* before any other library and see that happens:

``` bash
$ LD_PRELOAD=./libmyfakeobj.so ./myobj_user 
I am doing something nasty!
I am doing something nasty!
myobj_user: /tmp/oi/myobj_user.cpp:8: int main(int, char**): Assertion `obj.computeSomething() == 35' failed.
[1]    3813 abort (core dumped)  LD_PRELOAD=./libmyfakeobj.so ./myobj_user

```

Well, we print "I am doing something nasty!" twice, as *MyObj::somePrivateMethod()* is being called twice by the *MyObj::computeSomething()* method. And we return something that breaks the call to the public method by changing the behaviour of a function that should be private.

Note that I did not re-define the entire MyObj class, but just the method *somePrivateMethod()*. Note as well that I changed the method visibility from private to public, by defining MyObj as a struct instead of a class.

My conclusion so far is that hiding behaviour using the *private* keyword is lying to oneself.

I hope I'll be able to present you a way to fix (or better, or workaround) this issue.

### Short note on Windows/MSVC

Whereas on Unix a function visibility is public by default, on Windows the opposite happens. Everything is hidden by default, unless symbols are defined with ***__declspec(dllimport)*** and ***__declspec(dllexport)***. I am not sure though if the compiler is allowed to inline private methods, which are not declared as visible, as I have not tested my code there.

## The Right Way® To Hide Implementation Details

Let's refactor MyObj as:

{{< highlight cpp >}}

// MyObj.h

#include <string>

class MyObj {
public:
  MyObj(std::string);
  size_t computeSomething() const;
private:
  std::string _name;
};

{{< / highlight >}}

The only difference here is the absence of the method **somePrivateMethod()**. Everything else looks the same.

Now, the implementation file:

{{< highlight cpp >}}

// MyObj.cpp

#include "MyObj.h"

namespace {
size_t somePrivateMethod(const std::string& name, size_t value) {
  return name.length() * value;
}
}

MyObj::MyObj(std::string name): _name(name) {}

size_t MyObj::computeSomething() const {
  return somePrivateMethod(_name, 2) + somePrivateMethod(_name, 5);
}

{{< / highlight >}}

By having a look at the public symbols:

```
$ nm -C libmyobj.so
[...lots of output...]
0000000000001140 T MyObj::MyObj(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >)
0000000000001140 T MyObj::MyObj(std::__cxx11::basic_string<char, std::char_traits<char>, std::allocator<char> >)
0000000000001210 T MyObj::computeSomething() const
[...lots of output...]
```

Voilà, *somePrivateMethod()* has been properly hidden and probably inlined by the compiler. Its behaviour cannot anymore be overriden in linking time.

## When Things Get More Complex

In the previous example it was quite trivial to hide *somePrivateMethod()* because the only private member is hanndles is the `std::string` `_name`.

What if it uses other private members? Well, just add them as function parameters. So far so good.

But what if *somePrivateMethod()* needs access to a public member of MyObj? Well, then we pass MyObj as a function agument:

{{< highlight cpp >}}
namespace {
size_t somePrivateMethod(const MyObj& self, const std::string& name, size_t value) {
  /*...use const-public stuff from MyObj...*/
}
}

size_t MyObj::computeSomething() const {
  return somePrivateMethod(*this, _name, 2) + somePrivateMethod(_name, 5);
}

{{< / highlight >}}

It starts to look ugly, but that's fine for now.

## When Things Get Too Ugly

But what if *somePrivateMethod()* needs to access a protected method of a parent class or a private method of a class whose MyObj is friend of?

Consider the code, defined in a header file:

{{< highlight cpp >}}

class A {
protected:
  void aProtectedMethod() const;
};

class B: public A {
private:
  void aMethodThatUsesAProtectedMethodFromA() const
  {
    aProtectedMethod();
  }
};

{{< / highlight >}}

In this case, there's no way we can move *aMethodThatUsesAProtectedMethodFromA()* out of B, as the compiler will not allow it, as free function, to access a protected member of A.

Consider also the case:

{{< highlight cpp >}}

class B;

class A {
private:
  friend class B;
  void aPrivateMethod() const;
};

class B {
private:
  void aMethodThatUsesAPrivateMethodFromA() const
  {
    const auto a = A{};
    a.aPrivateMethod();
  }
};

{{< / highlight >}}

Again, the compiler will not let you extract *aMethodThatUsesAPrivateMethodFromA()*, as it expects only B to be able to access private members of A, not any arbitrary free function.

"Oh, we could make the free function *aMethodThatUsesAPrivateMethodFromA()* a friend of A!". Yes, but then it would make *aMethodThatUsesAPrivateMethodFromA()* visible again, even being a free function.

### Solution

I have no simple solution for those issues. For the friendship conflict, the solution is never using this feature. The need of friendship between classes and functions is almost always signal of bad design, with classes having too much responsability and uneeded dependencies to/from other classes or components.

Who has ever seen code where a system class has test classes or test functions as friends, just to test private behaviour? I guess everyone :-)

To the access to protected behaviour of parent classes, the solution I personally found is extracting protected methods to their own classes, where such protected methods are defined as public in such extracted classes, for example, rewritting the code we have seen before.

{{< highlight cpp >}}

class A_util {
public:
  // this name is misleading, I know, but I just extracted it with no further changes
  void aProtectedMethod() const;
};

class A { };

class B: public A {
private:
  A_util _a_util;

  void aMethodThatUsesAProtectedMethodFromA() const
  {
    _a_util.aProtectedMethod();
  }
};

{{< / highlight >}}

Such extraction is more complex in practise than in theory, but I have learned that with discipline it gets easier over time, until a point where such design becomes natural.

Such design is also considered a good practise in the literature, called "Composition over Inheritance".

## Hiding Object State Without Making It Private

I mentioned before that I appreciate the way Objective-C exposes only its interface, not declaring even the private members of a class in the header file. It implies in dynamic memory allocation (heap allocation) as that's the only way to allocate an object with unknown size to the compiler.

But back to C++, who are we trying to hide the private members from? The answer is only one: from the user. Maybe it's a good idea to rely on the user the task of hiding the private bits of a class. The way to accomplish that is through polymorphism. Luckly in C++ we have at least[^5] two kinds off polymorphism: at compile time and at runime.

At compile time, we can use templates or, on more recent versions (C++14 and newer) through automatic type deduction.

For instance, in the code below, the only known methods of the object passed as parameter is *action()*, all the rest is implicitely hidden from the user and from the compiler. 
Any class that has an action method can be used in *function()*, even though such class can have other methods and members, irrelevant to the current code.

{{< highlight cpp >}}

template<typename T>
void function(T t)
{
  t.action(); // any other method is "hidden"
}

{{< / highlight >}}

***NOTE:*** we could have defined function(auto), taking advantage of C++14 features. But You got the idea.

When finally concepts become part of the language, writting this kind of code will become way easier and less error-prone.

Another way is using runtime polymorphism, where again any other method of the passed object is irrelevant for the call of *function()*:

{{< highlight cpp >}}

struct Base
{
  virtual ~Base() = default;
  virtual void action();
};

void function(Action& t)
{
  t.action(); // any other method is "hidden"
}

{{< / highlight >}}

## A strange proposal ##

After all that said, I would like to propose something I have not yet been able to test in production, but that I think can be an alternative for the visibility model in C++. The idea is that, instead of having private and public members and functions in the same "place", but separated by visibility, to put them in segregated entities. Well, let me show you an example:

{{< highlight cpp >}}

// person.h

struct Person
{
  // move all "private" members to an inner (likely POD) struct
  struct details
  {
    int age;
    std::string name;
  };

  // public methods remain in the same place
  Person(int, std::string);
  int some_complex_computation() const;
};

{{< / highlight >}}

{{< highlight cpp >}}

// person.cpp

Person::Person(int age, std::string name):
  details({age, name})
{
}

namespace {
  int simpler_computation(const Person::details& details)
  {
    return details.name.lenght() * 42;
  }

  int other_simpler_computation(const Person::details& details)
  {
    return details.age + 3;
  }
}

int Person::some_complex_computation() const
{
  return simpler_computation(details) + other_simpler_computation(details);
}

{{< / highlight >}}

In this code, all implementation detail is completely hidden and able to be optimized out by compiler, and the object layout is the same as if age and name were declared directly in the outer class, at least in my tests.

One will complain: "But you are making private members accessible to anyone! What about the object invariants? Any user can break them by accessing manually the details inner struct!". Yes, this violates the [Core Guidelines](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#c2-use-class-if-the-class-has-an-invariant-use-struct-if-the-data-members-can-vary-independently) that states:

```C.2: Use class if the class has an invariant; use struct if the data members can vary independently```

If you document and make explicit to the user that they should not touch the values inside the *details* struct, you would expect them not to touch it. In my experience, treating the user as stupid is always a bad move. And, if the user ignore your warnings and change such values, breaking the class invariants, it is *THEIR* problem, not yours. You can never 100% prevent your code to be misused (as seen by the LD_PRELOAD trick mentioned earlier in this article).

In my experience, this strategy prevents a lot of common design issues in C++, like usage of friends and weird dependencies among classes. It also prevents the abomination which are protected members (often result of bad design), when used along object composition.

It also reduces recompilation of the client code when implementation details change, as only the bare minimum public interface and code that affects the object allocation is available in the header file.

But I repeat here that I have not had much time to discuss much or test such approach in production. I have seen code that I wrote using it become, from simple and readable, to monsters when someone devides to grow the class with private methods and public/private/protected. It almost made me cry.

## Conclusion ##

To summarize, C++ visibility model shows its age and has become a source of issues than a good tool in the programmer's toolbox. I demonstrated how hiding methods using private prevents important optimizations and allows the behaviour of such methods to be redefined at will in some environments.

I then suggest "keeping everything public" aproach, that relies on code convention rather than on compiler enforcement to handle what should or not be visible to the user of a class, and demonstrated how it "fixes" the forementioned issues.

On my approach, the object declaration IS the object interface, using Objective-C as source of inspiration.

I would like to stress (again!) that I am totally open to suggestions of improvements and even to accept being wrong on what I exposed here. Please feel free to comment.

Cheers!

[^1]: Err... not really. There's a lot of "magic" that can change add/remove behaviour of a class during runtime.
[^2]: A common workaround is using the *pimpl* idiom, but it usually implies in dynamic memory allocation.
[^3]: Actually extern cannot be explicitely used like that, so the code is not C++ valid.
[^4]: I have tested in both Linux and MacOS, using Clang and GCC. The behaviour on Windows/MSVC is different.
[^5]: I personally believe that the LD_PRELOAD trick is a kind of polymorphism, but done at linking time. Someone has called it Linking Time Polymorphism
