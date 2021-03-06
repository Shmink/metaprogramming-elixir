## Chapter 1 - The Language of Macros

#### What are Macros?

-   Macros are code that write code.
-   Elixir itself is made with macros, as a result you can extend the language itself
    to include things you think you might need.
-   Metaprogramming in elixir serves the purpose of extensibility by design.
-   With this power one can even define languages within elixir. The following is a valid Elixir program.

```elixir
div do
    h1 class: "title" do
        text "Hello"
    end
    p do
        text "Metaprogramming Elixir"
    end
end
"<div><h1 class=\"title\">Hello</h1><p>Metaprogramming Elixir</p></div>"
```

#### The Abstract Syntax Tree

-   Most languages use AST but you never need to know about them. They are used typically during compilation
    or interpretation to transform source code into a tree structure before being turned into bytecode
    or machine code..
-   José Valim, the creator of Elixir, chose to expose this AST and the syntax to interact with it.
-   We can now operate at the same level as the compiler.
-   Metaprogramming in Elixir revolves around manipulating and accessing ASTs.
-   To access the AST representation we use the `quote` macro.

```elixir
iex> quote do: 1 + 2
{:+, [context: Elixir, import: Kernel], [1, 2]}
```

```elixir
iex> quote do: div(10, 2)
{:div, [context: Elixir, import: Kernel], [10, 2]}
```

-   This is the internals of the Elixir language itself.
-   This gives you easy options for infering meaning and optimising performance all while being within Elixirs high level syntax.
-   The purpose of macros is to interact with this AST with the syntax of Elixir.
-   Macros turn you from language consumer to language creator. You have the same level of power as José when he wrote the standard library.

#### Trying It All Together

"Let's write a macro that can print the spoken form of an Elixir mathematical expression, such as 5 + 2, when calculating a result.
In most languages, we would have to parse a string expression into something digestible by our program. With Elixir, we can access
the representation of expressions directly with macros."

[First macro - `math.exs`](../math/math.exs)

<details>
<summary>math.exs</summary>

```elixir
defmodule Math do
@moduledoc false

    defmacro say({:+, _, [lhs, rhs]}) do
        quote do
        lhs = unquote(lhs)
        rhs = unquote(rhs)
        result = lhs + rhs
        IO.puts("#{lhs} plus #{rhs} is #{result}")
        result
        end
    end

    defmacro say({:*, _, [lhs, rhs]}) do
        quote do
        lhs = unquote(lhs)
        rhs = unquote(rhs)
        result = lhs * rhs
        IO.puts("#{lhs} times #{rhs} is #{result}")
        result
        end
    end
end
```

Output:

```elixir
iex> Math.say 5 + 2
5 plus 2 is 7
7

iex> iex> Math.say 18 * 4
18 times 4 is 72
72
```

</details>

Note when you use this in iex you need to first `c "math.exs"` then `require Math` but i've included it in [.iex.exs](../math/.iex.exs) to save time.
Automagically adding these when you open iex with `iex math.exs`.

In this example. We take what we know from the AST representations so far from the `quote` we used. We then create `defmacro`-s. We can still
have many function clauses with macros. With that, we create two macros called `say` and we pattern match on the AST with the defining feature
being the operator at the start of the AST, `{:+, ...}`, and use a new keyword called `unquote`. From the docs:

```elixir
iex(1)> h unquote

                                defmacro unquote(expr)

Unquotes the given expression from inside a macro.

## Examples

Imagine the situation you have a variable value and you want to inject it
inside some quote. The first attempt would be:

    value = 13
    quote do
        sum(1, value, 3)
    end

Which would then return:

    {:sum, [], [1, {:value, [], quoted}, 3]}

Which is not the expected result. For this, we use unquote:

    iex> value = 13
    iex> quote do
    ...>   sum(1, unquote(value), 3)
    ...> end
    {:sum, [], [1, 13, 3]}
```

I assume from this then when you pass variables to a macro they need to be `unqoute`-d, incontrast to passing a value directly.
Which I'm not following as Elixir is pass-by-value so wouldn't the value just be known?

Turns out that yes that's correct because we are dealing with ASTs not the data it represents; therefore the pass-by-value argument doesn't hold.
Much like interpolation from Ecto and the difference between `"Hello world"` and `"Hello #{world}`.

-   We know that macros receive the AST representation of the arguments.
-   "To complete the macro, we used `quote` to return an AST for the caller to replace out `Math.say` invocations." I'm assuming by this that every macro needs some form of `quote`.

#### Macro Rules

-   **Rule 1: Don't write macros.**  We have to remember that writing code to produce code require special care. It's easy to get caught in a web of your own code generation. Too many macros can make debugging more difficult.

-   **Rule: 2 Use macros gratuitously.** Metaprogramming is sometimes framed as complex and fragile as well as offering productive advantages in a fraction of the required code. It's important to keep this duality in mind when writing macros.

#### The Abstract Syntax Tree - Demystified

-   Every expression you write in Elixir breaks down to a three-element tuple in the AST.
-   This uniform format makes pattern matching arguments a lot easier.
-   Quoting a couple more complex expressions to see how entire Elixir prograns are structure in the AST.

<details>
<summary>(5 * 2 - 1 + 7)</summary>

```elixir
iex(1)> quote do: (5 * 2) - 1 + 7
{:+, [context: Elixir, import: Kernel],
 [
   {:-, [context: Elixir, import: Kernel],
    [{:*, [context: Elixir, import: Kernel], [5, 2]}, 1]},
   7
 ]}
```

</details>

<details>
<summary>MyModule</summary>

```elixir
iex(1)> quote do
...(1)>   defmodule MyModule do
...(1)>     def hello, do: "World"
...(1)>   end
...(1)> end
{:defmodule, [context: Elixir, import: Kernel],
    [
    {:__aliases__, [alias: false], [:MyModule]},
    [
        do: {:def, [context: Elixir, import: Kernel],
        [{:hello, [context: Elixir], Elixir}, [do: "World"]]}
    ]
    ]}
```

</details>

-   A stacking tuple was produced from each quoted exoression. The first example shows the familiar structures used by our `Math.say` macro, but multiple tuples are stacked into an embedded tree to represent the entire expression. The result of the second example shows how an entire Eliir module represented by a simple AST.

-   **All elixir code is represented as a series of three-element tuples with the following format:**

    -   **The first element** is an atom denoting the funcation call, or another tuple, representing a nested node in the AST.
    -   **The second element** represents metadata about the expression.
    -   **The third element** is a list of arguments for the function call.

-   Applying this to the AST of `(5 * 2) - 1 + 7` from before

```elixir
iex(1)> quote do: (5 * 2) - 1 + 7
{:+, [context: Elixir, import: Kernel],
    [
    {:-, [context: Elixir, import: Kernel],
    [{:*, [context: Elixir, import: Kernel], [5, 2]}, 1]},
    7
    ]}
```

-   An AST tree strunction of functions and arguments has been made. If we were to format this output into a more readable tree:

```bash
+
├── -
│   ├── *
│   │   ├── 5
│   │   └── 7
│   └── 1
└── 7
```

-   It seems easiest to start from the end of the AST and work up. The root AST node is the `+` operator, and its arguments are
    the number 7 combined with another nested node in the tree. We can see that the nested nodes contain out `(5 * 2)` expression,
    whose results are applied to the `- 1` branch.
-   `5 * 2` is syntactic sugar for `Kernel.*(5, 2)`. This means the `:*` atom is just a function call from the import of `Kernel` as
    you can see from the AST output from before.

#### High-Level Syntax vs. Low-level AST

Comparing the AST from elixir to the source of Lisp. If you look closely, you can see how elixir operates at a layer just above this format.

For example, Lisp source code:

```lisp
(+ (* 2 3) 1)
```

And Elixir from source to generated AST:

```elixir
quote do: 2 * 3 + 1
{:+, _, [{:*, _, [2, 3]}, 1]}
```

If we compare the both you can see that the structure itself is nearly identical with only the syntax being different. The beauty here is that
the transformation from high-level source to low-level AST requires only a `quote` invocation. On the contrary, with Lisp you have all the power
of a programmable AST at the cost of a less natural and flexible syntax. José seperated the AST from the syntax, meaning we get the best of both worlds.

#### AST Literals

-   Playing with elixir source, sometimes the results of quoted expressions can appear confusing and irregular. This is because of literals. Literals
    have the same representation within the AST and at high-lvel source. This includes atoms, ints, floats, lists, strings and any two-element tuples
    containing the former types. Such as the following:

```elixir
iex> quote do: :atom
:atom

iex> quote do: 123
123

iex> quote do: [1, 2, 3]

iex> quote do: {:ok, [1, 2, 3]}
{:ok, [1, 2, 3]}
```

If you pass any of the example to a macro, the macro receives the literal arguments instead of an abstract representation.

```elixir
quoute do: %{a: 1, b: 2}
{:%{}, [], [a: 1, b: 2]}

iex> quoute do: Enum
{:__aliases__, [alias: false], [:Enum]}
```

In these two examples we see that there are two different ways in which elixir types are represented in the AST. Some values are passed through as
is, while more complex tuples are returned as a quoted expression. It's useful to keep literals in mind when writing macros to avoid confusion.

#### Macros: The Building Blocks of Elixir

-   Let's imagine that `unless` does not exist in the elixir language. `unless` is essentially the same as a negated if statement, e.g.
    unless(1 + 1 = 5) would return true.

[Second macro](../unless/unless.exs) - `unless.exs` also [.iex.exs](../unless/.iex.exs)

<details>
<summary>unless.exs</summary>

```elixir
defmodule ControlFlow do
    defmacro unless(expression, do: block) do
    quote do
        if !unquote(expression), do: unquote(block)
    end
    end
end
```

Output:

```elixir
iex> ControlFlow.unless 2 == 5, do: "block entered"
"block entered"

iex> ControlFlow.unless 5 == 5, do: "block entered"
nil
```

</details>

-   Since the macros receive the AST representation of arguments, we can accept any valid elixir expression as the first argument to `unless`.
    In the second argument we pattern match of the `do` blockand bind uts AST value to a variable.

-   We then go straigh into our `quote` block where we build upon the `if` macro and simply negate it with `!`.

-   We also of course `unquote` the parameters passed to the macro.

#### Macro Expansion

-   When the compiler encounters a macro, it recursivley expands it until the code no loinger contains any macro calls.

-   Take the code `ControlFlow.unless 2 == 5`. The compiler seeing this will ask if `unless` is a macro. If it is, expand it and see what's in that macro. In this case it goes into `unless` and finds `if !`. Is `if` a macro? Yes it is, so expand again to find `case !` is that a macro? No, expansion is now complete.
-   `case` case macro is a member of a small set of special macros, located in `Kernel.SpecialForms`. These macros are funfamental building blocks in Elixir that cannot be overridden. They also represent the end of the road for macro expansion.

#### Code Injection and the Caller's Context

-   Elixir has the concept of macro _hygiene_. Hygiene means that variables, imports, and aliases that you define in a macro do not leak into the caller's own definitions.We must take special consideration with macro hygiene when expanding code, because sometimes it is necessary evil to implicitly access the caller's scope in an unhygenic way.
-   This safeguard also happens to prevent accidental namespace clashes.
-   This hygiene seems like a fancy version of just being in or out of scope, perhaps that's the point.
-   You can override this hygiene by pre-pending `var!`. For example, `if var!(meaning_to_life) == 42 do ....`  
-   When working with macros, it's important to be aware of what context a macro is executing in and to respect hygiene.

[callers_context.exs](callers-context/callers_context.exs)

<details>
<summary>callers_context.exs</summary>

```elixir
defmodule Mod do
  defmacro definfo do
    IO.puts("In macro's context (#{__MODULE__}).")

    quote do
      IO.puts("In caller's context (#{__MODULE__}).")

      def friendly_info do
        IO.puts("""
        My name is #{__MODULE__}
        My functions are #{inspect(__info__(:functions))}
        """)
      end
    end
  end
end

defmodule MyModule do
  require Mod
  Mod.definfo()
end
```

Output:

```elixir
iex(1)> c "callers_context.exs
In macro's context (Elixir.Mod).
In caller's context (Elixir.MyModule).
[MyModule, Mod]

iex(2)> MyModule.friendly_info
My name is Elixir.MyModule
My functions are [friendly_info: 0]

:ok
```
