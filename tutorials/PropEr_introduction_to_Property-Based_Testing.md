---
layout: tutorial
title: A really basic tutorial for PropEr beginners
author: Kostis Sagonas
sitemap:
    priority: 1.0
---
This tutorial is for absolute beginners. We will introduce the idea of
property-based testing and describe how one can employ it using PropEr.
In doing this, we will also present one particular aspect of input
generators that is probably unique to PropEr, namely generation and
shrinking of general Erlang terms.

The programming task that this tutorial is based on is really simple.
Let us assume that we want to write and test a program that sorts lists.
Our first attempt, inspired by quicksort, is the following Erlang code:

{% highlight erlang %}
-module(my_sort).
-export([sort/1]).

-spec sort([T]) -> [T].
sort([]) -> [];
sort([P|Xs]) ->
    sort([X || X <- Xs, X < P]) ++ [P] ++ sort([X || X <- Xs, P < X]).
{% endhighlight %}

To get some confidence that this module is correct, we can of course
compile and run some unit tests for this module in the Erlang shell:

{% highlight erl %}
1> c(my_sort).
{ok,my_sort}
2> my_sort:sort([17,42]).
[17,42]
3> my_sort:sort([42,17]).
[17,42]
4> my_sort:sort([3,1,4,2]).
[1,2,3,4]
{% endhighlight %}

Seems to be working alright.  At this point we probably want to add these
as <a href="http://erlang.org/doc/man/eunit.html">EUnit</a> tests in the
module.  This involves adding an appropriate `include_lib` directive below
the export declaration:

{% highlight erlang %}
-include_lib("eunit/include/eunit.hrl").
{% endhighlight %}

and also the following EUnit tests:

{% highlight erlang %}
sort_test_() ->
    [test_zero(), test_two(), test_four()].

test_zero() ->
    ?_assertEqual([], sort([])). % notice underscore
test_two() ->
    [?_assertEqual([17,42], sort([X,Y])) || {X,Y} <- [{17,42}, {42,17}]].
test_four() ->
    [?_assertEqual([1,2,3,4], sort([3,1,4,2]))].
{% endhighlight %}

At this point, we can recompile the module and run all the unit tests with
a single command:

{% highlight erl %}
5> c(my_sort).
{ok,my_sort}
6> eunit:test(my_sort).
  All 4 tests passed.
ok
{% endhighlight %}

Unit testing is an important part of program development. It provides us
with some confidence that our programs do what they are supposed to do in
some hard-coded, manually selected test cases that we want to ensure are
working properly when developing our program and will continue to work
properly after program modifications. Note, however, that the process of
writing unit tests becomes boring and tedious after a point: it's a manual
process after all.  More importantly, it's difficult to become _really_
confident that the tests we have written exercise all aspects of program
correctness that we care about.

This is where _property-based testing_ comes to play. The idea is to
specify only the _properties_ that we want our programs to satisfy, and
use a property-based testing tool to automatically generate inputs that
test whether our code satisfies these properties or not.

But what exactly are these properties? They are logical statements that
capture aspects of partial correctness of our functions/programs.
In the case of our example, one property we can reasonably expect our
sorting function to satisfy is that for all valid inputs, lists of some
elements in this case, the result of applying the `sort/1` function to
some list gives us back a list whose elements are ordered.  Let us write
this property using the language of PropEr:

{% highlight erlang %}
prop_ordered() ->
    ?FORALL(L, list(integer()), ordered(sort(L))).
{% endhighlight %}

PropEr uses the convention that all properties are functions whose name
begins with `prop_`. The body of this function here begins with a **`?FORALL`**,
which uses a macro defined by PropEr. For the Erlang compiler to find this
macro, we need to add an appropriate `include_lib` directive _above_ the
corresponding `include_lib` directive for EUnit. (We need to put the PropEr
include before the EUnit one, because both tools define some macro with the
same name, as we will soon see.)  The two directives should look as follows:

{% highlight erlang %}
-include_lib("proper/include/proper.hrl").
-include_lib("eunit/include/eunit.hrl").
{% endhighlight %}

The **`FORALL`** macro has three arguments. The first two go hand-in-hand:
the first is an Erlang term containing _variables_ that will get values which
are produced by the _generators_ in the second argument. In this particular
case, there is only one variable whose values will be (semi-random) lists
of integers. Both `list/1` and `integer/0` are built-in generators defined
in the [proper_types](/apidocs/proper_types.html) module. However,
one does not need to specify the module name, as PropEr knows where to find
them when used in PropEr macros. The third argument is a Boolean expression
that specifies the property we are interested in. In this case the property
is self-explanatory: calling the `sort/1` function on some list `L` should
return a list which is ordered.

To use this, we also need to define the `ordered/1` function.  But this is
simple enough:

{% highlight erlang %}
ordered([]) -> true;
ordered([_]) -> true;
ordered([A,B|T]) -> A =< B andalso ordered([B|T]).
{% endhighlight %}

We can now test whether this property holds for our program:

{% highlight erl %}
7> c(my_sort).
{ok,my_sort}
8> proper:quickcheck(my_sort:prop_ordered()).
....................................................................................................
OK: Passed 100 test(s).
true
{% endhighlight %}

Indeed, it holds. PropEr ran 100 tests, which is the default number of tests
to run, generating one list of integers at a time, and checked whether the
property was true for them.

We can actually test a particular property any number of times we wish:

{% highlight erl %}
9> proper:quickcheck(my_sort:prop_ordered(), 4711).
...................... <4711 dots> ..................................................................
OK: Passed 4711 test(s).
true
{% endhighlight %}

This is not the only property that we want a list sorting function to satisfy.
Another one is that the input and output lists have the same length.
Let's write it:

{% highlight erlang %}
prop_same_length() ->
    ?FORALL(L, list(integer()), length(L) =:= length(sort(L))).
{% endhighlight %}

and test it:

{% highlight erl %}
10> c(my_sort).
{ok,my_sort}
11> proper:quickcheck(my_sort:prop_same_length()).
..........!
Failed: After 11 test(s).
[0,0,1,0]

Shrinking .(1 time(s))
[0,0]
false
{% endhighlight %}

Oops!  This property does not hold for our program.  PropEr ran some tests,
eleven in total in this case, and found the input list `[0,0,1,0]` for which
the property is false.  Consequently, PropEr automatically _shrank_ this
input list to a list of smaller length, actually of minimal length, that
also falsifies the property.

Let's test the property again:

{% highlight erl %}
12> proper:quickcheck(my_sort:prop_same_length()).
..................!
Failed: After 19 test(s).
[-8,14,-1,1,-1]

Shrinking ......(6 time(s))
[0,0]
false
{% endhighlight %}

Once again, PropEr discovered that the property does not hold.  It took a
total of nineteen tests this time for the property to be falsified.  The
input list in this case was `[-8,14,-1,1,-1]`, which was subsequently shrunk
down to the `[0,0]` list once again.  In contrast to the previous shrinking
process, which was done in one step, shrinking now involved six steps to find
a minimal input that can be derived from the original one and still falsify
the property.

What's the problem?  We can find out by manually testing the `sort/1` function
in the Erlang shell using these inputs that PropEr discovered for us:

{% highlight erl %}
13> my_sort:sort([-8,14,-1,1,-1]).
[-8,-1,1,14]
14> my_sort:sort([0,0]).
[0]
{% endhighlight %}

It appears that the `sort/1` function we wrote does not work when the list
contains duplicate elements.  A moment's thought will reveal that we have
forgotten to handle the case that one of the elements in `Xs` is equal to
the pivot element `P`.  Consequently, our `sort/1` function throws away
duplicate elements in the input list.  Oh dear!

Of course, we can easily correct the problem by changing one of the `'<'` tests
in the code of the `sort/1` function to `'=<'` and the function will now satisfy
the `prop_same_length` property.  However, this is not a tutorial on how to
write a correct sorting function in Erlang.  Instead, it's a tutorial on how
to do property-based testing using PropEr.

Let us suppose that we actually _wanted_ to write a program that only sorts
lists without duplicates.  How would we write the same-length property then?

One way of writing this property is to impose a precondition for it.  That is,
only test its validity if this precondition is satisfied.  The **`IMPLIES`**
macro of PropEr allows us to write the property in a way that the property is
checked only in this case:

{% highlight erlang %}
prop_same_length_conditional_check() ->
    ?FORALL(L, list(integer()),
            ?IMPLIES(no_duplicates(L), length(L) =:= length(sort(L)))).
{% endhighlight %}

that is, only check the property if the list `L` contains no duplicates.
A possible implementation of function `no_duplicates/1` is the following:

{% highlight erlang %}
%% better implementations of no_duplicates/1 exist ...
no_duplicates([]) -> true;
no_duplicates([A|T]) ->
     not lists:member(A, T) andalso no_duplicates(T).
{% endhighlight %}

Let's test this property:

{% highlight erl %}
15> c(my_sort).
{ok,my_sort}
16> proper:quickcheck(my_sort:prop_same_length_conditional_check()).
........x.x.......x......xxx..xxxxxx....xxx.x.x..x.xxxxxxx.xxx.x.....x.xxxxx
xx.xxx.x.x.xxxx..xxxxx..x.xx.x..x.xx.xxxxx.xx.....xxx.xxx...xxx..x..xx.xxxx.
xxxx.xx.xx.xxxxx.xxxx.x..x...x.x.xxx..xx..xxxx.x....xx.x.x.xxxxx.x.xx..xx.
OK: Passed 100 test(s).
true
{% endhighlight %}

What's happening here?  Well, for starters the property is now found to hold
when running 100 tests, which enhances our confidence that our `sort/1`
function now satisfies this property for its intended uses.

But what are the `x` symbols that are intermixed with the `.` symbols above?
This is PropEr's way of indicating that it generated some list of integers
for which the `no_duplicates/1` precondition is false.
Consequently, the property was not checked for these inputs, and PropEr had
to generate some other list(s) in order to reach one hundred randomly
generated test cases for which the property passes.  In fact, in this case,
PropEr had to generate a total of 226 input lists of integers in order to
find 100 of them that do not contain duplicates.

This is clearly wasteful.  In situations where the precondition is more
involved, it may take quite a while before random input generation comes
up with a particular number of instances that satisfy the precondition.
In such cases, we may be better off generating inputs for which the precondition
holds by construction.  Let's do this for lists that do not contain duplicate
elements.  We can do this in a really simple way: we will use PropEr's built-in
machinery to generate random lists of integers and we will filter out any
duplicate elements ourselves.  The **`LET`** macro can be used for this:

{% highlight erlang%}
list_no_dupls(T) ->
    ?LET(L, list(T), remove_duplicates(L)).

%% better versions of remove_duplicates/1 exist ...
remove_duplicates([]) -> [];
remove_duplicates([A|T]) ->
    case lists:member(A, T) of
        true -> remove_duplicates(T);
        false -> [A|remove_duplicates(T)]
    end.
{% endhighlight %}

This provides a _custom generator_ (for lists of some type `T` that contain no
duplicates).  We can now use this generator for our same-length property:

{% highlight erlang %}
prop_same_length_no_dupls() ->
    ?FORALL(L, list_no_dupls(integer()), length(L) =:= length(sort(L))).
{% endhighlight %}

which we can include in the module and test:

{% highlight erl %}
17> c(my_sort).
{ok,my_sort}
18> proper:quickcheck(my_sort:prop_same_length_no_dupls()).
....................................................................................................
OK: Passed 100 test(s).
true
{% endhighlight %}

Note that now PropEr does not disgard any input which is provided by the
`list_no_dupls/1` custom generator.

At this point we have a module that contains four different properties.
In EUnit it was possible to run all tests of a module by calling
`eunit:test(ModuleName).`  Is there something analogous for PropEr?
Indeed there is:

{% highlight erl %}
19> proper:module(my_sort).
Testing my_sort:prop_same_length_no_dupls/0
....................................................................................................
OK: Passed 100 test(s).

Testing my_sort:prop_same_length_conditional_check/0
.......x....x...xx.....x...x...x..x...x.xxx.....x..x....x.x.xx.xxxxx...x.xx.x.xxxxxxxx....xxxxxx.x.x
..xx..xx.xxxxx.x..x.xxxxx.x.xxxx..x.xx.x....x.x.x.x.x...x.xx.xx....xxx.x.xxxxxxx.x.xx.x.xxxxxx.xx..x
x..
OK: Passed 100 test(s).

Testing my_sort:prop_ordered/0
....................................................................................................
OK: Passed 100 test(s).

Testing my_sort:prop_same_length/0
.........................!
Failed: After 26 test(s).
[4,7,-5,-9,7,0,23]

Shrinking .......(7 time(s))
[0,0]

[{my_sort,prop_same_length,0}]
{% endhighlight %}

PropEr examines all properties, one at a time, and either successfully
completes some specific number of tests for which the property is true,
or discovers some input that falsifies a property (and shrinks this input
to a mimimal one that also falsifies the property).  Note that the return
value of `proper:module/1` is a list of MFAs of properties for which a
counterexample is found.

We've come quite far in using property-based testing.  Still, if we think
about it, the two properties (ordered and same-length) that we specified for
our sort function only _partially_ capture its intended behaviour.  We
probably would like to also check that the output list contains not only the
_same number_ of elements, but _exactly the same_ elements as the input list.
There are various ways that this property can be specified, but this is the
point where we can take a shortcut.  Rather than specifying a function that
ensures this, we can employ some already defined - and presumably well-tested -
function that is known to have the property we are interested in and check
that the two functions are equivalent.  For our `sort/1` function that does
not need to handle duplicate elements in the input list, an appropriate such
function is the `usort/1` function from the `lists` module of the standard
library.  Specifying this property is now very easy:

{% highlight erlang %}
prop_equiv_usort() ->
    ?FORALL(L, list(integer()), sort(L) =:= lists:usort(L)).
{% endhighlight %}

So is testing it:

{% highlight erl %}
20> c(my_sort).
{ok,my_sort}
21> proper:quickcheck(my_sort:prop_equiv_usort()).
....................................................................................................
OK: Passed 100 test(s).
true
{% endhighlight %}

Although this particular property may feel a bit trivial, testing two functions
for equivalence using a simple **`FORALL`** property like the above is something
that comes handy very often.  For example, during program development, it
allows us to start from a quite inefficient but 'obviously correct' prototype
implementation, and gradually refine and optimize it using the prototype
implementation as a reference to check whether the code refactorings and
optimizations we performed have not somehow jeopardized its correctness.

Let us bring us one last point before we finish this tutorial.  As can be seen
in its `spec`, our `sort/1` function is polymorphic.  In fact, in Erlang it is
supposed to be applicable to lists containing all sorts of Erlang terms, not
some particular sort only, and certainly not only integers.  How can we be sure
that indeed it works for all Erlang terms?  Shouldn't we have been testing that
our properties hold for lists of any term instead of just for lists of integers?
Well, let's do that.  Let's modify this last property to work for general lists;
for this we just need to use the built-in `list/0` generator:

{% highlight erlang %}
prop_equiv_usort() ->
    ?FORALL(L, list(), sort(L) =:= lists:usort(L)).
{% endhighlight %}

and test the property again:

{% highlight erl %}
22> c(my_sort).
{ok,my_sort}
23> proper:quickcheck(my_sort:prop_equiv_usort()).
....................................................................................................
OK: Passed 100 test(s).
true
{% endhighlight %}

Unsurprisingly, the property still passes 100 tests.  But how can be sure that
PropEr indeed generated lists containing Erlang terms of different sorts instead
of some particular one?  More importantly, how does shrinking work for such
lists?  We can see this by using the `list/0` generator in the
`prop_same_length` property which is false for our `sort/1` function.  Let's do
that:

{% highlight erlang %}
prop_same_length() ->
    ?FORALL(L, list(), length(L) =:= length(sort(L))).
{% endhighlight %}

and see it fail:

{% highlight erl %}
24> c(my_sort).
{ok,my_sort}
25> proper:quickcheck(my_sort:prop_same_length()).
.......................!
Failed: After 24 test(s).
[[],{[<<1:2>>,-3]},[<<6:3>>,-4,'^\227m-\220­',[{},{[]}],-3,1,'\e\214\024'],
[[],23.308667937715537,-7.223776166565898,{},11,5.782189592459757,-8],
2.06372249239632,[-1.2567064751604635,-8.020835435581166,{[]},[],-3,'',{}],
[],<<228,37>>]

Shrinking ....(4 time(s))
[[],[]]
false
{% endhighlight %}

So we see that PropEr has no problem in generating lists containing any Erlang
term and then shrinking them down to a minimal list that still falsifies the
property.  This is something that other property-based testing tools for
Erlang can _not_ do (or consciously decide not to support).


# Summary
In this introductory tutorial you have learned:

*  What property-based testing is and how it differs from unit testing.

*  How to use the **`FORALL`** and **`IMPLIES`** macros for writing properties.

*  About some built-in generators (`integer/1`, `list/1` and `list/0`).
   The [proper_types](/apidocs/proper_types.html) module contains
   many more and we invite you to learn about their existence.

*  How to create simple _custom generators_ using **`LET`**.

*  That PropEr tries to minimize the counterexamples it finds through a
   process called _shrinking_.

*  That shrinking can be applied to any Erlang term.

**Note:**
You can get the complete final code of this tutorial, without this last
modification to the `prop_same_length` property, by clicking on the following
link:
  [my_sort.erl](/code/my_sort.erl)
