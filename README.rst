===========================
Divergence Debugging Module
===========================

    The official dogfood project for Rebase.

Getting Started
===============

The debug module depends on `Divergence <http://github.com/spencertipping/divergence>`_ and `Rebase <http://github.com/spencertipping/divergence.rebase>`_. Make sure you've got both of these
loaded prior to loading the debug module::

  <script src='divergence.js'></script>
  <script src='divergence.rebase.js'></script>
  <script src='divergence.debug.js'></script>

Once you've got the debug module loaded, you can start debugging things. You'll also want to set up a trace function::

  // for HTML:
  d.tracer = function (x) {document.getElementById ('log').appendChild (document.createTextNode (x))};

  // for node.js:
  d.tracer = require('sys').print;

  // for Firefox:
  d.tracer = console.log.bind(console);

Tracing Functionals
===================

The debug module hooks into Divergence functionals (see the `Divergence guide <http://github.com/spencertipping/divergence-guide>`_ if the term "functional" is unfamilar) to provide a
``traced()`` method. The output is a functional that traces its arguments, ``this`` binding, return value, and any exceptions thrown::

  var f = '$0 + $1'.traced ();
  f (4, 5)
    // returns 9 and traces:
    //   anonymous (1) called on [object global] with [4, 5]
    //   anonymous (1) returned 9

You can also name your functions and/or specify a custom trace function::

  var f = '$0 + $1'.traced ({name: 'foo', tracer: alert});
  f (4, 5)      // alerts:
                //   foo (1) called ...
                //   foo (1) returned ...

Notice that numbers are printed out along with the function name. These are important for debugging recursive functions, as the call and return of the same invocation will have matching
numbers. So, for example::

  d.rebase (function () {
    var buggy_factorial = (n >$> (n > 0 ? n * buggy_factorial (n - 1) : n)).traced();
    buggy_factorial (3)
      // returns 6 and traces:
      //   anonymous (1) called on [object global] with [3]
      //   anonymous (2) called on [object global] with [2]
      //   anonymous (3) called on [object global] with [1]
      //   anonymous (4) called on [object global] with [0]
      //   anonymous (4) returned 0
      //   anonymous (3) returned 0
      //   anonymous (2) returned 0
      //   anonymous (1) returned 0
  }) ();

Expression-Level Debugging
==========================

Sometimes you need to know not only which functions get called, but also what is going on inside your functions. This requires some hardcore debugging action, but that's what this module is
all about (note that this API is subject to change)::

  d.rebase (function () {
    var watcher = new d.debug.watcher ();
    var factorial = eval (watcher.annotate_local (n >$> literal(n > 1 ? n * factorial (n - 1) : n)));
    factorial (2);
    watcher.log() * d.trace;
      // traces:
      //   (n) = (2) at 1275195177662
      //   (n > 1) = (true) at 1275195177662
      //   (n) = (2) at 1275195177662
      //   (factorial) = (function (n) {return gensym_hook1(9 , gensym_hook1(1 , gensym_hook1(0 , n) > 1) ? gensym_hook1(7 , gensym_hook1(2 , n) * gensym_hook1(6 , gensym_hook1(3 ,
      //     factorial)(gensym_hook1(5 , gensym_hook1(4 , n) - 1)))) : gensym_hook1(8 , n))}) at 1275195177662
      //   (n) = (2) at 1275195177662
      //   (n - 1) = (1) at 1275195177662
      //   (n) = (1) at 1275195177662
      //   (n > 1) = (false) at 1275195177662
      //   (n) = (1) at 1275195177662
      //   (n > 1 ? n * factorial(n - 1) : n) = (1) at 1275195177662
      //   (factorial(n - 1)) = (1) at 1275195177662
      //   (n * factorial(n - 1)) = (2) at 1275195177662
      //   (n > 1 ? n * factorial(n - 1) : n) = (2) at 1275195177662
  }) ();

There's a lot of information here, but the idea is that every expression's value was timestamped and recorded. This lets you observe the exact trace of evaluation as it occurred in the
program. Now obviously this isn't very useful as-is, but you can easily grab subsets of that information::

  watcher.log().grep (e >$> (e.node.op == '>')) * d.trace;
    // traces:
    //   (n > 1) = (true) at 1275196068275
    //   (n > 1) = (false) at 1275196068276

A reference to the value of each expression is also stored. You can see when something evaluates to 1, for instance, by running this::

  watcher.log().grep (e >$> (e.value === 1)) * d.trace;
    // traces:
    //   (n - 1) = (1) at 1275196262921
    //   (n) = (1) at 1275196262921
    //   (n) = (1) at 1275196262922
    //   (n > 1 ? n * factorial(n - 1) : n) = (1) at 1275196262922
    //   (factorial(n - 1)) = (1) at 1275196262922

OK, so I didn't really address this earlier, but you're probably wondering what's with the ``eval`` above. You actually don't need it in all situations, but it is required when you're making
references to locally-scoped variables (hence the ``annotate_local`` method, as opposed to plain ``annotate``). Here's the deal. Just like Rebase loses a reference to closure variables, the
debug module will lose that reference as well. It doesn't just lose the variables, it in fact loses the entire scope chain. In situations when you need to preserve that scope chain, you use
``eval``, which is dynamically scoped, in the location where the function's scope chain should be. Thus ``annotate_local``, rather than returning the annotated function, returns a regular
string to be evaled.

Tracing vs. Logging
-------------------

By default a watcher logs its functions' actions. This is more helpful for digging through the history in a paused script, but it won't tell you the exact interplay of events within a traced
function or print things in real-time. If you need that, then you should switch the watcher to tracing mode::

  watcher.use_tracing();
  factorial (2);
    // traces the big listing of expression values

You can also specify a predicate to determine whether a log event should be printed::

  watcher.use_tracing(e >$> (e.value === 1));
  factorial (2);
    // traces the short list
