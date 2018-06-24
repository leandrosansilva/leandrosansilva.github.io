---
title: "Testable Singletons in C++"
date: 2018-06-23T20:50:19+02:00
draft: false

tags:
  - cpp
  - programming
  - testing
  - refactoring
  - singleton
  - design patterns
---

_TL-DR: make the **getInstance()** method of a Singleton return an abstract type and allow it to be overriden at link-time._

*Disclaimer: the technique here explained is maybe as old as C++. I don't claim to be the author of the idea. I just wish I had learned it 10 years ago.*

So you hate, or at least dislike, the Singleton pattern, right?

I am not its fan and can't recall the last time I created one, but we must accept the fact that they exist in the wild, specially in legacy codebases.
And replacing Singleton usages can be very hard in such cases. There must be a better way to turn code that depends on them testable!

The most popular argument against Singletons is that they are global variables, but I don't believe this is their greatest sin.
The major issue about Singletons is that they are concrete. Or this is how most developers implement them.

By being concrete, testing code that depend on them is hard. This is not the only fate of a Singleton, though.

In the book "Working Effectively with Legacy Code", Michael C. Feathers describes what he called "seams":

*A seam is a place where you can alter the behaviour in your program without editing that place.*

In the case of code that use a Singleton, we can create a seam in the *getInstance()* method. We will use a "Link Seam", that means we will alter some behaviour during linking time.

Consider the sample code below, that consist on a class called **Config** that implements the Singleton pattern, and some code that depends on it. The code is simplified and some aspects required in real world code are intentionally missing.

First, the interface of the **Config** class:

{{< highlight cpp >}}
// config.h

#pragma once

#include <string>

struct Config
{
  Config(Config&&) = delete;
  Config(const Config&) = delete;
  Config& operator=(const Config&) = delete;
  static Config& instance();
  int getAge() const;
  std::string getName() const;
private:
  Config();
};
{{< / highlight >}}

And its implementation:

{{< highlight cpp >}}
// config.cpp

#include "config.h"
#include <mutex>

namespace {
  // I know, in the real world this should be a unique_ptr
  Config* instance = nullptr;
}

Config::Config()
{
}

Config& Config::instance()
{
  static std::once_flag flag;

  std::call_once(flag, [] {
    ::instance = new Config();
  });

  return *::instance;
}

int Config::getAge() const
{
  // read something from the filesystem
  return 10;
}

std::string Config::getName() const
{
  // get something from a database or webservice
  return "Some Name";
}
{{< / highlight >}}

We have then some complex and hard to change system that use that Singleton. Although we don't want or can't change it we need to write tests to it.
It declares a function that returns a string that depends on **Config**.

{{< highlight cpp >}}
// printer.h

#pragma once

#include <string>

std::string printNameAndAge();
{{< / highlight >}}

{{< highlight cpp >}}
#include "printer.h"
#include "config.h"

#include <sstream>

std::string printNameAndAge()
{
  std::ostringstream ss;

  ss << "age: " << Config::instance().getAge() << ", name: " << Config::instance().getName();

  return ss.str();
}
{{< / highlight >}}

Let's then introduce the seam. We can do it by making the class **Config** abstract, by extracting its interface, and moving the method **Config::instance()** to a location that can be replaced during link time. In the example, I moved the entire implementation to a shared library that I call *libconfigsingleton*. The code then looks like this:

{{< highlight cpp >}}
// config.h

#pragma once

#include <string>

struct Config
{
  virtual ~Config() = default;

  static Config& instance();

  virtual int getAge() const = 0;
  virtual std::string getName() const = 0;
};
{{< / highlight >}}

I also create a class **ProductionConfig** that implements the **Config** interface to make explicit this is ***THE*** production Singleton.

{{< highlight cpp >}}
// config.cpp
#include "config.h"
#include <mutex>

namespace {
  struct ProductionConfig: Config
  {
    int getAge() const final
    {
      return 10;
    }

    std::string getName() const final
    {
      return "Some Name";
    }
  };

  // I know, in the real world this should be a unique_ptr
  Config* instance = nullptr;
}

// instance() is our seam, as it is now in a shared library
Config& Config::instance()
{
  static std::once_flag flag;

  std::call_once(flag, [] {
    ::instance = new ProductionConfig();
  });

  return *::instance;
}
{{< / highlight >}}

We then link the production code to *libconfigsingleton*.

That said, we can create our test code, that overrides, during link time, **Config::instance()**, and link to the system rest of the system, but not to *libconfigsingleton*:

{{< highlight cpp >}}
// test.cpp
#include <assert.h>

#include "printer.h"
#include "config.h"

// Mocking Config
struct TestConfig: Config
{
  int age = int{};
  std::string name;

  int getAge() const final
  {
    return age;
  }

  std::string getName() const final
  {
    return name;
  }
};

TestConfig testConfig;

// Our seam
Config& Config::instance()
{
  return testConfig;
}

int main(int, char**)
{
  // setup some test values
  testConfig.age = 23;
  testConfig.name = "Another Name";

  // and run the system
  assert(printNameAndAge() == "age: 23, name: Another Name");
}
{{< / highlight >}}

One downside of this technique are that we pay the runtime price of indirect call to all methods in the singleton, as they are now virtual. Additionally sometimes breaking up the codebase smaller libraries can difficult in some build systems and will require a big parts of the system to be recompiled.

On the other hand it makes the definition of the Singleton much clearer and removes workarounds like the private constructor to ensure that there is only one instance of the singleton class. By definition in C++ a abstract class can never be directed allocated.

Regarding this last point, one could complain that the Singleton pattern is used when there can be only one instance of a class and the abstract one I use here does not restrict it, as there can be as many instances of its subclasses. I personally see no problems with that, as it is still impossible to create another instance that depends on the behaviour of the single production one, only accessible via the **instance()** method.

I confess though that I haven't yet had the opportunity to use this technique in production code (I have worked mostly with Objective-C in the past couple of years), so am open to suggestions on how to improve the solution and to hear from those who faced similar challenges in real world codebases.
