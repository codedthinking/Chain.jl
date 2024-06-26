# With.jl

A [Julia package](https://julialang.org/packages/) for piping a value through a series of transformation expressions using a more convenient syntax than Julia's native [piping functionality](https://docs.julialang.org/en/v1/manual/functions/#Function-composition-and-piping).

<table>
<tr><th>With.jl</th><th>Base Julia</th></tr>
<tr>
<td>
      
```julia
@with df begin
  dropmissing
  filter(:id => >(6), _)
  groupby(:group)
  combine(:age => sum)
end
```

</td>
<td>

```julia
df |>
  dropmissing |>
  x -> filter(:id => >(6), x) |>
  x -> groupby(x, :group) |>
  x -> combine(x, :age => sum)
```

</td>
</tr>
<tr>
<th><a href="https://github.com/oxinabox/Pipe.jl">Pipe.jl</a></th>
<th><a href="https://github.com/MikeInnes/Lazy.jl">Lazy.jl</a></th>
</tr>
<tr>
<td>
  
```julia
@pipe df |>
  dropmissing |>
  filter(:id => >(6), _)|>
  groupby(_, :group) |>
  combine(_, :age => sum)
```

</td>
<td>
		
```julia
@> df begin
  dropmissing
  x -> filter(:id => >(6), x)
  groupby(:group)
  combine(:age => sum)
end
```

</td>
</tr>
</tr>
</table>

## Summary

With.jl exports the `@with` macro.

This macro rewrites a series of expressions into a with, where the result of one expression
is inserted into the next expression following certain rules.

**Rule 1**

Any `expr` that is a `begin ... end` block is flattened.
For example, these two pseudocodes are equivalent:

```julia
@with a b c d e f

@with a begin
    b
    c
    d
end e f
```

**Rule 2**

Any expression but the first (in the flattened representation) will have the preceding result
inserted as its first argument.

If the expression is a symbol, the symbol is treated equivalently to a function call.

For example, the following code block

```julia
@with begin
    x
    f()
    @g()
    h
    @i
end
```

is equivalent to

```julia
begin
    local temp1 = f(x)
    local temp2 = @g(temp1)
    local temp3 = h(temp2)
    local temp4 = @i(temp3)
end
```

**Rule 3**

An expression that begins with `@aside` does not pass its result on to the following expression.
Instead, the result of the previous expression will be passed on.
This is meant for inspecting the state of the with.
The expression within `@aside` will not get the previous result auto-inserted, you can use
underscores to reference it.

> FIXME: change example

```julia
@with begin
    [1, 2, 3]
    filter(isodd, _)
    @aside @info "There are \$(length(_)) elements after filtering"
    sum
end
```

**Rule 4**

It is allowed to start an expression with a variable assignment.
In this case, the usual insertion rules apply to the right-hand side of that assignment.
This can be used to store intermediate results.

```julia
@with begin
    [1, 2, 3]
    filtered = filter(isodd, _)
    sum
end

filtered == [1, 3]
```

**Rule 5**

The `@.` macro may be used with a symbol to broadcast that function over the preceding result.

```julia
@with begin
    [1, 2, 3]
    @. sqrt
end
```

is equivalent to

```julia
@with begin
    [1, 2, 3]
    sqrt.(_)
end
```


## Motivation

- The implicit first argument insertion is useful for many data pipeline scenarios, like `groupby`, `transform` and `combine` in DataFrames.jl
- There is no need to type `|>` over and over
- Any line can be commented out or in without breaking syntax, there is no problem with dangling `|>` symbols
- The state of the pipeline can easily be checked with the `@aside` macro
- Flattening of `begin ... end` blocks allows you to split your with over multiple lines
- Because everything is just lines with separate expressions and not one huge function call, IDEs can show exactly in which line errors happened
- Pipe is a name defined by Base Julia which can lead to conflicts

## Example

An example with a DataFrame:

```julia
using DataFrames, With

df = DataFrame(group = [1, 2, 1, 2, missing], weight = [1, 3, 5, 7, missing])

result = @with df begin
    dropmissing
    filter(r -> r.weight < 6, _)
    groupby(:group)
    combine(:weight => sum => :total_weight)
end
```

The with block is equivalent to this:

```julia
result = begin
    local var"##1" = dropmissing(df)
    local var"##2" = filter(r -> r.weight < 6, var"##1")
    local var"##3" = groupby(var"##2", :group)
    local var"##4" = combine(var"##3", :weight => sum => :total_weight)
end
```

## Nested Withs

The `@with` macro replaces all underscores in the following block, unless it encounters another `@with` macrocall.
In that case, the only underscore that is still replaced by the outer macro is the first argument of the inner `@with`.
You can use this, for example, in combination with the `@aside` macro if you need to process a side result further.

```julia
@with df begin
    dropmissing
    filter(r -> r.weight < 6, _)
    @aside @with _ begin
            select(:group)
            CSV.write("filtered_groups.csv", _)
        end
    groupby(:group)
    combine(:weight => sum => :total_weight)
end
```

## License

This software is licensed under an [MIT License](LICENSE). Large parts of the software builds on [Chain.jl](https://github.com/jkrumbiegel/Chain.jl) by Julius Krumbiegel (2020), available under an MIT License.