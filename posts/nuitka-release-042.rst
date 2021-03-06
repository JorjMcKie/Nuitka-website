This is to inform you about the new stable release of `Nuitka <http://nuitka.net>`_. It is the extremely compatible Python compiler. Please see the page `"What is Nuitka?" </pages/overview.html>`_ for an overview.

This release comes with many bug fixes, some of which are severe. It also
contains new features, like basic Python 3.3 support. And the `performance
diagrams <http://nuitka.net/pages/performance.html>`__ got expanded.

New Features
------------

- Support for FreeBSD.

  Nuitka works for at least FreeBSD 9.1, older versions may or may not
  work. This required only fixing some "Linuxisms" in the build process.

- New option for warning about compile time detected exception raises.

  Nuitka can now warn about exceptions that will be raised at run time.

- Basic Python3.3 support.

  The test suite of CPython3.2 passes and fails in a compatible way. New feature
  ``yield from`` is not yet supported, and the improved argument parsing error
  messages are not implemented yet.

Bug Fixes
---------

- Nuitka already supported compilation of "main directories", i.e. directories
  with a "__main__.py" file inside. The resulting binary name was "__main__.exe"
  though, but now it is "directory.exe"

  .. code-block:: sh

     # ls directory
     __main__.py

     # nuitka --exe directory
     # ls
     directory directory.exe

  This makes this usage more obvious, and fixes the older issue `Issue#49
  <http://bugs.nuitka.net/issue49>`__ for this feature.

- Evaluation order of binary operators was not enforced.

  Nuitka already enforces evaluation order for just about everything. But not
  for binary operators it seems. Corrects `Issue#61
  <http://bugs.nuitka.net/issue61>`__.

- Providing an ``# coding: no-exist`` was crashing under Python2, and ignored
  under Python3, now it does the compatible thing for both.

- Global statements on the compiler level are legal in Python, and were not
  handled by Nuitka, they now are.

  .. code-block:: python

     global a # Not in a function, but on module level. Pointless but legal!
     a = 1

  Effectively these statements can be ignored. Corrects part of `Issue#65
  <http://bugs.nuitka.net/issue65>`__.

- Future imports are only legal when they are at the start of the file. This was
  not enforced by Nuitka, making it accept code, which CPython would reject. It
  now properly raises a syntax error. Corrects part of `Issue#65
  <http://bugs.nuitka.net/issue65>`__.

- Raising exceptions from context was leaking references.

  .. code-block:: python

     raise ValueError() from None

  Under CPython3.2 the above is not allowed (it is acceptable starting
  CPython3.3), and was also leaking references to its arguments. Corrects
  `Issue#76 <http://bugs.nuitka.net/issue76>`__.

- Importing the module that became ``__main__`` through the module name, didn't
  recurse to it.

  This also gives a warning. PyBench does it, and then stumbles over the
  non-found "pybench" module. Of course, programmers should use ``sys.modules[
  "__main__" ]`` to access main module code. Not only because the duplicated
  modules don't share data. Corrects `Issue#68
  <http://bugs.nuitka.net/issue68>`__.

- Compiled method ``repr`` leaked references when printed.

  When printing them, they would not be freed, and subsequently hold references
  to the object (and class) they belong to. This could trigger bugs for code
  that expects ``__del__`` to run at some point. Corrects `Issue#81
  <http://bugs.nuitka.net/issue81>`__.

- The ``super`` built-in leaked references to given object.

  This was added, because Python3 needs it. It supplies the arguments to
  ``super`` automatically, whereas for Python2 the programmer had to do it. And
  now it turns out that the object lost a reference, causing similar issues as
  above, preventing ``__del__`` to run. Corrects `Issue#81
  <http://bugs.nuitka.net/issue81>`__.

- The ``raise`` statement didn't enforce type of third argument.

  This Python2-only form of exception raising now checks the type of the third
  argument before using it. Plus, when it's None (which is also legal), no
  reference to None is leaked.

- Python3 built-in exceptions were strings instead of exceptions.

  A gross mistake that went uncaught by test suites. I wonder how. Them being
  strings doesn't help their usage of course, fixed. Corrects `Issue#82
  <http://bugs.nuitka.net/issue82>`__.

- The ``-nan`` and ``nan`` both exist and make a difference.

  A older story continued. There is a sign to ``nan``, which can be copied away
  and should be present. This is now also supported by Nuitka. Corrects
  `Issue#75 <http://bugs.nuitka.net/issue75>`__.

- Wrong optimization of ``a == a``, ``a != a``, ``a <= a`` on C++ level.

  While it's not done during Nuitka optimization, the rich comparison helpers
  still contained short cuts for ``==``, ``!=``, and ``<=``.

- The ``sys.executable`` for ``nuitka-python --python-version 3.2`` was still
  ``python``.

  When determining the value for ``sys.executable`` the CPython library code
  looks at the name ``exec`` had received. It was ``python`` in all cases, but
  now it depends on the running version, so it propagates.

- Keyword only functions with default values were losing references to
  defaults.

  .. code-block:: python

     def f(*, a = X())
        pass

     f()
     f() # Can crash, X() should already be released.

  This is now corrected. Of course, a Python3 only issue.

- Pressing CTRL-C didn't generate ``KeyboardInterrupt`` in compiled code.

  Nuitka never executes "pending calls". It now does, with the upside, that the
  solution used, appears to be suitable for threading in Nuitka too. Expect more
  to come out of this.

- For ``with`` statements with ``return``, ``break``, or ``continue`` to leave
  their body, the ``__exit__`` was not called.

  .. code-block:: python

     with a:      # This called a.__enter__().
         return 2 # This didn't call a.__exit__(None, None, None).

  This is of course quite huge, and unfortunately wasn't covered by any test
  suite so far. Turns out, the re-formulation of ``with`` statements, was
  wrongly using ``try/except/else``, but these ignore the problematic
  statements. Only ``try/finally`` does. The enhanced re-formulation now does
  the correct thing. Corrects `Issue#59 <http://bugs.nuitka.net/issue59>`__.

- Starting with Python3, absolute imports are now the default.

  This was already present for Python3.3, and it turns out that all of Python3
  does it.

New Optimization
----------------

- Constants are now much less often created with ``pickle`` module, but created
  directly.

  This esp. applies for nested constants, now more values become ``is``
  identical instead of only ``==`` identical, which indicates a reduced memory
  usage.

  .. code-block:: python

     a = ("something_special",)
     b = "something_special"

     assert a[0] is b # Now true

  This is not only about memory efficiency, but also about performance. Less
  memory usage is more cache friendly, and the "==" operator will be able to
  shortcut dramatically in cases of identical objects.

  Constants now created without ``pickle`` usage, cover ``float``, ``list``, and
  ``dict``, which is enough for PyStone to not use it at all, which has been
  added support for as well.

- Continue statements might be optimized away.

  A terminal ``continue`` in a loop, was not optimized away:

  .. code-block:: python

     while 1:
         something
         continue   # Now optimized away

  The trailing ``continue`` has no effect and can therefore be removed.

  .. code-block:: python

     while 1:
         something

- Loops with only break statements are optimized away.

  .. code-block:: python

     while 1:
         break

  A loop immediately broken has of course no effect. Loop conditions are
  re-formulated to immediate "if ... : break" checks. Effectively this means
  that loops with conditions detected to be always false to see the loop
  entirely removed.

New Tests
---------

- Added tests for the found issues.

- Running the programs test suite (i.e. recursion) for Python3.2 and Python3.2
  as well, after making adaptation so that the absolute import changes are now
  covered.

- Running the "CPython3.2" test suite with Python3.3 based Nuitka works and
  found a few minor issues.

Organizational
--------------

- The `Downloads <http://nuitka.net/pages/download.html>`__ page now offers RPMs
  for RHEL6, CentOS6, F17, F18, and openSUSE 12.1, 12.2, 12.3. This large
  coverage is thanks to openSUSE build service and "ownssh" for contributing an
  RPM spec file.

  The page got improved with logos for the distributions.

- Added "ownssh" as contributor.

- Revamped the "`User Manual <http://nuitka.net/doc/user-manual.html>`__" in
  terms of layout, structure, and content.

Summary
-------

This release is the result of much validation work. The amount of fixes the
largest of any release so far. New platforms, basic Python3.3 support,
consolidation all around.
