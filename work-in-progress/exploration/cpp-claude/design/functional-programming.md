# Analysis: cpp-design-principles/functional-programming.md

Disposition: **PORTS_AS_IS** for the principles; **PORTS_WITH_ADAPTATION**
for the storage-and-callable subsection (no `inplace_function`, but the
"capture lifetimes" guidance has Python-specific traps).

## Section-by-section

### Pure functions and value semantics

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Inputs in, value out, no globals, no mutation, no
  logging.
- **Python rendering:** Identical. The "discount" example:

  ```python
  def apply_discount(total: Money, discount_pct: int) -> Money:
      return total - total * discount_pct // 100
  ```

  Add a note on `[[nodiscard]]` -> the Python equivalent is "test
  asserts on the return value" + a ruff rule (`B015` flags
  expression-statement results). There is no exact `[[nodiscard]]`
  analogue in Python.

### Pattern matching on sum types

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Use `lib::match` over `std::variant`, with one arm
  per alternative, exhaustiveness checked at compile time.
- **Python rendering:** Use `match` / `case` over a discriminated union
  (`type OrderEvent = Placed | Canceled | Filled`) with a final
  `case _ as never: assert_never(never)` arm. pyright/pyrefly verify
  exhaustiveness. See `research/exhaustive-matching.md` for the full
  recipe.

### Higher-order functions

- **Classification:** PORTS_AS_IS.
- **C++ principle:** Pass behavior as a value; named predicates;
  predicate factories.
- **Python rendering:** Same. Python idioms: `functools.partial`,
  `operator.attrgetter` / `operator.itemgetter`, a closure over a
  captured value, a `Protocol` for the parameter shape.

### Storing callables

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Type-erased wrappers (`std::function`,
  `function_ref`, `inplace_function`) with different ownership and
  allocation properties.
- **Python rendering:** No allocation concern -- Python callables are
  first-class objects. The relevant table for Python:

  | Callable shape                     | When                                                                |
  |------------------------------------|---------------------------------------------------------------------|
  | `Callable[[Args], Ret]` type hint  | Default for function parameters.                                    |
  | `functools.partial`                | Pre-bind arguments; same shape as the C++ "predicate factory".      |
  | `Protocol` with `__call__`         | When the callable carries extra structure (named methods, state).   |
  | A `weakref.proxy` to a bound method | When the callback outlives the owning instance and we want it cleared on owner GC. |

  Drop the inplace-function discussion; it has no Python analogue.

### Compile-time predicates and folds

- **Classification:** PYTHON_SPECIFIC_VARIANT.
- **C++ principle:** Concepts and `if constexpr (requires { ... })`
  let the compiler pick branches.
- **Python rendering:** Python's compile-time story is "the type checker
  picks branches". The equivalent shapes:

  - `Protocol` checks via `isinstance(x, MyProtocol)` with
    `@runtime_checkable`, narrowed inside the `if` branch.
  - `TypeIs` / `TypeGuard` user-defined type guards (PEP 742) for
    narrowing.
  - Parameter packs become `*args: T` / `**kwargs: T` with `TypeVarTuple`
    where the typing pays off.

  Skip the "fold over a parameter pack" subsection -- the Python
  equivalent is `sum(args)` / `functools.reduce`, which everyone
  already knows.

### When functional style costs you

- **Classification:** PORTS_WITH_ADAPTATION.
- **C++ principle:** Type erasure has a cost; materializing views
  allocates; captures have lifetimes.
- **Python rendering:**
  - **Type erasure** -- no equivalent concern. Drop.
  - **Materializing generators** -- valid; same as the C++ point.
    `list(gen)` allocates; iterating a `filter(...)` twice raises
    `StopIteration` the second time.
  - **Captures have lifetimes** -- a different shape: Python's late
    binding in closures is the classic trap. The example:

    ```python
    # BAD - late binding; all funcs see the final i
    funcs = [lambda: i for i in range(3)]
    # [2, 2, 2], not [0, 1, 2]

    # GOOD - bind via default argument
    funcs = [lambda i=i: i for i in range(3)]
    ```

    Also note that closures hold strong references to captured names,
    which can prolong object lifetimes; use `weakref` if a closure is
    stored long-term and the captured object should be allowed to die.

## Suggested Python file: `python-design-principles/functional-programming.md`

Verbatim port of the principle sections; drop the C++-specific
type-erasure subsection; replace the "compile-time predicates" subsection
with a "type-checker-driven dispatch" subsection covering `Protocol`,
`TypeIs`, and structural narrowing.
