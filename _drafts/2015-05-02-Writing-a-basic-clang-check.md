---
layout: post
title: Writing a basic clang check
---

[Clang](http://clang.llvm.org) provides a user-friendly framework for writing
basic static-analysis checks.
In this post we will see how analysis on the clang abstract syntax tree (AST)
can be used to implement pretty powerful checks for single translation units
pretty easily.


* TOC
{:toc}


## The problem

As an example we will write a static analysis check which can catch the
following problematic code:

{% highlight cpp %}
// problematic.cpp

struct A {
  void f() {}
};

struct B : public A {
  virtual void f() {}  // problematic
};
{% endhighlight %}

Here `B` defines a virtual method `B::f` with a name identical to a
member function in its base class `A` which was never intended to be customized
(i.e. it was not declared as `virtual` there).

This will create the confusing situation that depending on the type the member
function was called through different functions will be called, e.g.

{% highlight cpp %}
// create two pointers to a B, but through either A or B
std::unique_ptr<A> a(new B());
std::unique_ptr<B> b(new B());
a->f();  // calls A::f
b->f();  // calls B::f
{% endhighlight %}

We will write a check that catches the problematic method declaration in
class `B`.


## Setting up needed tools

We will write our check as an extension of [the `clang-tidy`
tool](http://clang.llvm.org/extra/clang-tidy.html) which is part of clang.

If we have a [compilation
database](http://clang.llvm.org/docs/JSONCompilationDatabase.html) for our
source we can perform a check with `clang-tidy` by running in the build
directory (assuming we have a compilation database)

{% highlight sh %}
% clang-tidy some/file.cpp
{% endhighlight %}

or on any source file, even without compilation databases, but with no
automatic support for custom build flags

{% highlight sh %}
% clang-tidy some/file.cpp --
{% endhighlight %}


Since we will be working directly inside the clang tool sources we need to
check out and build the upstream sources.

{% highlight sh %}
# download the sources
% git clone http://llvm.org/git/llvm.git
% cd llvm/tools/
% git clone http://llvm.org/git/clang.git
% cd clang/tools/
% git clone http://llvm.org/git/clang-tools-extra.git extra

# build everything
% cd ../../../
% mkdir build && cd build/
# see http://llvm.org/docs/CMake.html#options-and-variables
# for details on the cmake build process of LLVM
% cmake -DCMAKE_BUILD_TYPE=RelWithDebInfo ..
% make check-clang-tools
{% endhighlight %}

Make sure you add the `bin/` directory of that build to your path, e.g.

{% highlight sh %}
% export PATH=$PWD/bin:$PATH
{% endhighlight %}

With that we are ready to add our custom check.

The `clang-tidy` sources are in `tools/clang/tools/extra/clang-tidy` inside the
LLVM source tree.


## Adding scaffolding

Let's change to the `clang-tidy` source directory and add our check (which we
will aptly call `VirtualShadowingCheck`).

{% highlight sh %}
% cd ../tools/clang/tools/extra/clang-tidy.
{% endhighlight %}

We will add our tool to the `[misc]` category of `clang-tidy` checks and before
anything else will need to create the needed files and integrate them into the
build. Thankfully there is a tool doing all of that for us:

{% highlight sh %}
% ./add_new_check.py misc virtual-shadowing
{% endhighlight %}

This will create `misc/VirtualshadowingCheck.h` and
`misc/VirtualshadowingCheck.cpp`, and additionally include it in
`misc/MiscTidyModule.cpp` so it can be run as a normal part of `clang-tidy`.
As of `clang-tools-extra` svn revision `236309` (git commit `6a5bbb2`) we still
need to modify `misc/CMakeLists.txt` so that our newly added dependency
`VirtualShadowingCheck.cpp` comes before the `LINK_LIBS` line.

We can now build a version of `clang-tidy` including our checker by rebuild
llvm and tools. To run it on some code we would run

{% highlight sh %}
% clang-tidy -checks='-*,misc-virtual-shadowing' some/file.cpp
{% endhighlight %}

Here we have first disabled all default-enabled checks with `-*` and then
exclusively enabled our check.  Right now running this does not output too much
interesting information.


## Anatomy of a checker

Looking at `misc/VirtualShadowingCheck.h` we find

{% highlight cpp %}
namespace clang {
namespace tidy {

class VirtualShadowingCheck : public ClangTidyCheck {
public:
  VirtualShadowingCheck(StringRef Name, ClangTidyContext *Context)
      : ClangTidyCheck(Name, Context) {}
  void registerMatchers(ast_matchers::MatchFinder *Finder) override;
  void check(const ast_matchers::MatchFinder::MatchResult &Result) override;
};

} // namespace tidy
} // namespace clang
{% endhighlight %}

Here `VirtualShadowingCheck` is our custom check defined inside the `clang::tidy` namespace. It derives from
`ClangTidyCheck`. We will need to provide implementations for two functions,
`registerMatchers` and `check`:

* in `registerMatchers` we register [clang AST
  matchers](http://clang.llvm.org/docs/LibASTMatchers.html) to filter out
  intesting source locations, and
* with `check` we provide a function which is called by the clang machinery
  whenever a match was found and we can perform further actions (e.g. emit a
  warning).

In our case we want for any `virtual` method of some class check
whether any of the class' bases defined a method with the same name, so our
implementation strategy will be

* filter out declarations of virtual methods with a matcher registered in
  `registerMatchers`, and
* walk the bases of the matched class in `check` and compare to the matched
  `virtual` method.


## A matcher for virtual methods

Clang comes with [a large set of basic
matchers](http://clang.llvm.org/docs/LibASTMatchersReference.html) for many use
cases. Chaining them allows creating powerful matchers.

To get a feeling for the kind of AST node representing `B::f` looking at the
clang AST is helpful,

{% highlight cpp linenos %}
% clang-check -ast-dump problematic.cpp --
TranslationUnitDecl 0x2b3cd20 <<invalid sloc>> <invalid sloc>
|-TypedefDecl 0x2b3d258 <<invalid sloc>> <invalid sloc> implicit __int128_t '__int128'
|-TypedefDecl 0x2b3d2b8 <<invalid sloc>> <invalid sloc> implicit __uint128_t 'unsigned __int128'
|-TypedefDecl 0x2b3d698 <<invalid sloc>> <invalid sloc> implicit __builtin_va_list '__va_list_tag [1]'
|-CXXRecordDecl 0x2b3d6e8 </media/sf_home_host/test.cpp:1:1, line:3:1> line:1:8 referenced struct A definition
| |-CXXRecordDecl 0x2b3d800 <col:1, col:8> col:8 implicit struct A
| `-CXXMethodDecl 0x2b3d8e0 <line:2:9, col:19> col:14 f 'void (void)'
|   `-CompoundStmt 0x2b3d9b8 <col:18, col:19>
`-CXXRecordDecl 0x2b3d9d0 <line:5:1, line:7:1> line:5:8 struct B definition
  |-public 'struct A'
  |-CXXRecordDecl 0x2b85050 <col:1, col:8> col:8 implicit struct B
  |-CXXMethodDecl 0x2b85100 <line:6:3, col:21> col:16 f 'void (void)' virtual
  | `-CompoundStmt 0x2b854f8 <col:20, col:21>
  |-CXXMethodDecl 0x2b85208 <line:5:8, <invalid sloc>> col:8 implicit operator= 'struct B &(const struct B &)' inline noexcept-unevaluated 0x2b85208
  | `-ParmVarDecl 0x2b85330 <col:8> col:8 'const struct B &'
  `-CXXDestructorDecl 0x2b853b8 <col:8> col:8 implicit ~B 'void (void)' inline noexcept-unevaluated 0x2b853b8
{% endhighlight %}

We see the two classes defined in lines 6 and 10 as `CXXRecordDecl` which
encode both classes and structs; the two definitions of `f` in lines 8 and 13
encoded as `CXXMethodDecl`s which encapsulate (not much suprisingly)
declarations of methods in C++.

The filter we need would first need method declarations and then then refine
that to only methods declared `virtual`. As first filter we use the
`methodDecl` matcher,

{% highlight sh %}
% clang-query problematic.cpp --
clang-query> match methodDecl()

/tmp/problematic.cpp:8:3: note: "root" binds here
  virtual void f() {}  // problematic
  ^~~~~~~~~~~~~~~~~~~
1 match.
clang-query> match methodDecl()

Match #1:

/tmp/problematic.cpp:4:3: note: "root" binds here
  void f() {}
  ^~~~~~~~~~~

Match #2:

/tmp/problematic.cpp:8:3: note: "root" binds here
  virtual void f() {}  // problematic
  ^~~~~~~~~~~~~~~~~~~

Match #3:


Match #4:

/tmp/problematic.cpp:7:8: note: "root" binds here
struct B : public A {
       ^
4 matches.
{% endhighlight %}

which 4 finds all the {CXXMethodDecl} (two for our explicitly declared methods
`f` and two for implicitly declared methods).

We chain that with the `isVirtual` matcher to only match virtual methods,

{% highlight sh %}
% clang-query problematic.cpp --
clang-query> match methodDecl(isVirtual())

/tmp/problematic.cpp:8:3: note: "root" binds here
  virtual void f() {}  // problematic
  ^~~~~~~~~~~~~~~~~~~
1 match.
{% endhighlight %}

which finds just the method we are interested in.


All left for us to do is to update the definition of
`VirtualShadowingCheck::registerMatchers` in `misc/VirtualShadowingCheck.cpp`
so it matches `virtual` method declarations,

{% highlight cpp %}
void VirtualShadowingCheck::registerMatchers(MatchFinder *Finder) {
  Finder->addMatcher(methodDecl(isVirtual()).bind("method"), this);
}
{% endhighlight %}

This binds the identifier `method` to the found method.


## Testing the check

If you have built `clang-tidy` with

{% highlight sh %}
% make check-clang-tools
{% endhighlight %}

you will have seen that the dummy test case added by `add_new_check.py` now
fails so we should update it to correctly reflect our intended use case.



## Processing of base classes

The matcher we registered will call `VirtualShadowingCheck::check` whenever a
matching AST node was found. There we can retrieve the matches by name with

{% highlight cpp %}
const auto method = Result.Nodes.getNodeAs<CXXMethodDecl>("method");
{% endhighlight %}

Here `Result.Nodes.getNodeAs<CXXMethodDecl>` returns a `CXXMethodDecl*` which
is valid as long as the translation unit is loaded (i.e. much longer than
`check` is running).

To check the base classes for non-virtual methods with identical names we need
to walk the tree of bases; clang provides infrastructure to perform that walk
with
[`CXXRecordDecl::forallBases`](http://clang.llvm.org/doxygen/classclang_1_1CXXRecordDecl.html#acd5fb7de4853357c1fd9a9450f44ce60),

{% highlight cpp %}
bool CXXRecordDecl::forallBases(ForallBasesCallback *BaseMatches,
                                void *UserData,
                                bool AllowShortCircuit = true 
                               ) const
{% endhighlight %}

`BaseMatches` is a callback with the signature

{% highlight cpp %}
typedef bool ForallBasesCallback(const CXXRecordDecl *BaseDefinition,
                                 void *UserData)
{% endhighlight %}

and `BaseDefinition` a `CXXRecordDecl` pointing to the declaration of a base
class. `UserData` can point to something we could use to pass additional
information along.  With `AllowShortCircuit = true` the callback will be called
for all bases as long the callback returns `true`; for `AllowShortCircuit =
false` all bases would be walked. If the class has no bases
`CXXRecordDecl::forallBases` returns `false`.

We can use this to recursively walk the tree of bases. We will pass a pointer
to `method` as `Userdata` to perform checks on the name. Inside
`VirtualShadowingCheck::check` we would call

{% highlight cpp %}
if (not cl_decl->forallBases(CandidatePred,
                             const_cast<CXXMethodDecl *>(method)))
  return;
{% endhighlight %}

i.e. call some predicate `CandidatePred` for all bases of the class containing
`method` and give up if none of them returns true.

If this would not exit `check` we emit a warning for `method` since it collides
with some base's name. With this `VirtualShadowingCheck::check`'s declaration would look like

{% highlight cpp %}
void VirtualShadowingCheck::check(
    const ast_matchers::MatchFinder::MatchResult &Result) {
  const auto method = Result.Nodes.getNodeAs<CXXMethodDecl>("method");
  const auto cl_decl = method->getParent();

  if (not cl_decl->forallBases(CandidatePred,
                               const_cast<CXXMethodDecl *>(method)))
    return;

  diag(method->getLocStart(),
       "method hides non-virtual function from a base class");
}
{% endhighlight %}

* * *

Now all the work left is to define the predicate.

{% highlight cpp %}
namespace
{
bool CandidatePred(const CXXRecordDecl *BaseDefinition, void *UserData) {
  // get the original method from UserData
  const auto method = reinterpret_cast<const CXXMethodDecl *>(UserData);

  // get a string ref to the original method's name
  const auto name = method->getName();

  // check all the methods in the base class
  for (const auto &base_method : BaseDefinition->methods()) {
    if (name == base_method->getName()) {
      // if we found a method with the same name check if it's virtual:
      //   * if it is not virtual the original method was problematic
      //   * if the base's method was virtual the original method was OK
      //     (but the base's method could be problematic and would be checked
      //     in a different analyzer run)
      return not base_method->isVirtual();
    }
  }

  // we found no collisions with this base's methods, now check the base's
  // bases for collisions with the original method
  return BaseDefinition->forallBases(CandidatePred, UserData);
}
} // namespace
{% endhighlight %}


## Testing the check

Even though I have started this post with a test case you still have not seen
how to integrate your test cases with `clang-tidy`'s own test suite. To make
sure our checker stays valid while clang's implementation moves on we should
add it there (in reality this would have been our zeroth step before moving to
the implementation, but I wanted to go directly into that first).

The test suite for extra clang tools lives in
`llvm/tools/clang/tools/extra/test/`; the test cases for `clang-tidy` are in
the (much suprise) `clang-tidy/` subdirectory. In fact when we created the
scaffolding code for our check with `add_new_check.py` already added a
well-documented skeleton test case `misc-virtual-shadowing.cpp` for us. We will
just need to add some test cases which would then be run as part of the clang
tools test suite,

{% highlight sh %}
% make check-clang-tools
{% endhighlight %}


## Conclusion

While we have just scratched the surface of what is possible, it has never been
easier to write custom static analysis checks of C++ code. Clang provides both
infrastructure and tools which allow the user to focus on the real problem --
formulating the problem.
