Testing framework for Emacs
===========================

Tests
-----

A simple test may look like:

```Lisp
(deftest test-foo
  (assert (= (+ 2 2) 4))
  (assert (= (* 3 3) 9))
  (assert (= (% 4 2) 0)))
```

This checks that 2 + 2 = 4, that 3 * 3 = 9, and that 4 % 2 = 0.
(If it isn't, something is seriously wrong!) To run the test:

```Lisp
(test-foo)    ; `M-x test-foo' interactively
```

To run the test when it's defined, specify `:run t`:

```Lisp
(deftest test-foo
  :run t
  (assert (= (+ 2 2) 4))
  (assert (= (* 3 3) 9))
  (assert (= (% 4 2) 0)))
```

Note that it's a good idea to name tests with a prefix like "`test-`"
to avoid overwriting other functions.

Let's simplify it a bit. Most `assert-` forms defined here accept
multiple (sets of) arguments, similarly to, e.g., `setq`:

```Lisp
(deftest test-foo
  :run t
  (assert
    (= (+ 2 2) 4)
    (= (* 3 3) 9)
    (= (% 4 2) 0)))
```

This is Lisp, after all! To remove the `=` noise, use `assert-=`:

```Lisp
(deftest test-foo
  :run t
  (assert-=
    (+ 2 2) 4
    (* 3 3) 9
    (% 4 2) 0))
```

Note that xUnit frameworks sometimes use the reverse order, e.g.,
"`assertEquals(4, 2 + 2);`", where the expected value comes first.
Here, however, the expectation always comes last, mirroring the
original code.

At this point it's advisable to add some commentary. Tests as well
as assertions can have textual annotations:

```Lisp
(deftest test-foo
  "Example test."
  :run t
  (assert-=
    "Elementary truths."
    (+ 2 2) 4
    (* 3 3) 9
    (% 4 2) 0))
```

If the test fails, the annotations show up in the failure report.
(Note, again, that the forms must be ordered as shown above for
the report to make sense.)

For a more BDD-like style, one can write `should` in place
of `assert`:

```Lisp
(deftest test-baz
  "A list."
  (let ((list '(a b c)))
    (push 'd list)
    (should (eq (first list) 'd))))
```

Related actions may be grouped together with `expect`, which only
checks the last form:

```Lisp
(deftest test-baz
  "A list."
  (let ((list '(a b c)))
    (expect
      "Push elements to the front."
      (push 'd list)
      (eq (first list) 'd))))
```

This is useful for providing context in the failure report, as well
as in the test itself. `should` statements can also be grouped:

```Lisp
(deftest test-baz
  "A list."
  (let ((list '(a b c)))
    (expect
      "Push elements to the front."
      (push 'd list)
      (should (eq (first list) 'd))
      (push 'e list)
      (should (eq (first list) 'e)))))
```

BDD aliases are defined for all assertions: `should-eq` instead of
`assert-eq`, `should-=` instead of `assert-=`, and so on.
`should` is sufficient in most cases, though: it displays a
recursive inspection of any failing form, and it checks each and
every form it contains.

**Note:** `assert` only accepts multiple arguments inside `deftest`.
Outside `deftest` it's a different macro (defined by `cl.el`).

Test suites
-----------

Tests can be grouped into suites with `defsuite`. The most
straightforward way is simply to wrap it around them:

```Lisp
(defsuite test-foo-suite
  (deftest test-foo
    (assert-=
      (+ 2 2) 4))
  (deftest test-bar
    (assert-=
      (* 3 3) 9)))
```

Like tests, the suite is executed with (test-foo-suite),
`M-x test-foo-suite` or `:run t` in the definition. Suites
can also have annotations:

```Lisp
(defsuite test-foo-suite
  "Example suite."
  :run t
  (deftest test-foo
   (assert-=
     (+ 2 2) 4)))
```

One can also define the test suite first and then add tests
and suites to it, using the `:suite` keyword or `add-to-suite`:

```Lisp
(defsuite test-foo-suite
  "Example suite.")

(deftest test-foo
  :suite test-foo-suite
  (assert-=
    (+ 2 2) 4))

(deftest test-bar
  (assert-=
    (* 3 3) 9))

(add-to-suite 'test-foo-suite 'test-bar)
```

Furthermore, `defsuite` forms may nested. (Self-referencing suite
definitions should be avoided, although some safeguards exist to
prevent infinite loops.)

Fixtures
--------

Sometimes it's useful to set up and tear down an environment for
each test in a suite. This can be done with the `:setup` and
`:teardown` keyword arguments, which accept a form to evaluate before
and after each test. (You can use `progn` to group expressions.)

```Lisp
(defsuite test-foo-suite
  "Example suite."
  :setup (wibble)
  :teardown (progn (wobble) (flob))
  (deftest test-foo
    ...)
  (deftest test-bar
    ...))
```

However, this might not be sufficient: what if the setup and
teardown need to share variables, or the test should be wrapped in
a macro like `save-restriction`? To that end, the more powerful
`:fixture` keyword argument may be used. It accepts a one-argument
function which is used to call the test:

```Lisp
(defsuite test-foo-suite
  "Example suite."
  :fixture (lambda (body)
             (unwind-protect
                 ;; set up environment
                 (save-restriction
                   (wibble)
                   (wobble)
                   ;; run test
                   (funcall body))
               ;; tear down environment
               (wubble)
               (flob)))
  (deftest test-foo
    ...)
  (deftest test-bar
    ...))
```

As shown above, the function must contain `(funcall body)` somewhere
in its definition for the test to be run at all. It is good style
to use `unwind-protect` to ensure that the fixture always completes
properly, regardless of the test's outcome.

Finally, there's the `:wrap` keyword argument, which specifies an
around-advice for the whole test, e.g., `:wrap ((wobble) ad-do-it)`.
See the docstring of `defadvice` for more details on advice. While
the other fixtures are repeated for each test in the suite, `:wrap`
is executed once for the whole suite. The order is:

    +-------------------+
    |:wrap              |
    |  +==============+ |
    |  |:setup        | |
    |  |  +---------+ | |
    |  |  |:fixture | | |
    |  |  |  +----+ | | |
    |  |  |  |TEST| | | |
    |  |  |  +----+ | | |
    |  |  +---------+ | |
    |  |:teardown     | |
    |  +==============+ |
    +-------------------+

A test defined as part of a suite carries with it the suite's
fixtures even when called outside the suite. However, when the test
is called by a different suite, that suite's fixtures temporarily
override the fixtures inherited from the original suite.

Any single test may also specify its own fixtures. In that case,
the suite fixtures are wrapped around the test fixtures. However,
no fixtures are executed at all if the test is called from within
another test: the calling test is assumed to provide the necessary
environment.

When defining a function to use as a fixture, make sure to define
it before the tests are run (before the test if using `:run t`).

Mocks and stubs
---------------

Mocks and stubs are temporary stand-ins for other pieces of code.
They are useful for disabling (or "stubbing out") external behavior
while testing a unit.

To stub a function, use `stub`:

```Lisp
(deftest test-foo
  "Example test."
  (stub foo)
  (assert-not (foo)))  ; foo returns nil
```

In the rest of the test, any calls to the stubbed function will
return `nil`. To return a different value, specify the stub's body,
e.g., `(stub foo t)`.

A stub only changes a function's output, not its input: the
argument list remains the same. The stub's body may refer to the
original arguments. To change a function's input too, use `mock`:

```Lisp
(deftest test-foo
  "Example test."
  (mock foo (arg)
    (1+ arg))
  (assert-= (foo 1) 2))  ; foo returns 2
```

`mock` specifies a temporary function to assign to `foo` for the
duration of the test. Here it increments its argument by one. When
the test completes (or fails), all stubs and mocks are released.

If the same mock is frequently reused, put it in a fixture or
define a function for it and call that function in the test. Just
ensure that it is never called outside a test, otherwise it will
not be released (unless wrapped in `with-mocks-and-stubs`).
