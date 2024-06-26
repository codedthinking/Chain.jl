# v0.6

**Breaking**: The rules for transforming withs were simplified.
Before, there was the two-arg block syntax (this was the only syntax originally):

```julia
@with x begin
    y
    z
end
```

the inline syntax:

```julia
@with x y z
```

and the one-arg block syntax:

```julia
@with begin
    x
    y
    z
end
```
All of these are now a single syntax, derived from the rule that any `begin ... end` block in the inline syntax is flattened into its lines.
This means that you can also use multiple `begin ... end` blocks, and they can be in any position, which can be nice for interactive development of a with in the REPL.

```julia
@with x y begin
    x
    y
    z
end u v w begin
    g
    h
    i
end
```

This is only breaking if you were using a `begin ... end` block in the inline syntax at argument 3 or higher, but you also had to be using an underscore without withing in that begin block, which is deemed quite unlikely given the intended use of the package.
All "normal" usage of the `@with` macro should work as it did before.

As another consequence of the refactor, withs now do not error anymore for a single argument form `@with x` but simply return `x`.

# v0.5

**Breaking**: The `@with` macro now creates a `begin` block, not a `let` block.
This means that variables that are assigned within the macro are available outside.
Technically, situations are imaginable where this could lead to overwritten variables if someone used large expressions with intermediate variable names in begin blocks spliced into the with.
It is however quite unlikely for the normal way that `@with` is intended to be used.

Additionally, it is now possible to use the syntax `variable = some_expression` to make use of the feature that variables can be exported.
The `some_expression` part is handled exactly like before.
This enables you to carry parts of a computation forward to a later step in the with or outside of it:

```julia
@with df begin
    transform(...)
    select(...)
    intermediate = subset(...)
    groupby(...)
    combine(...)
    join(intermediate)
end

@show intermediate
```