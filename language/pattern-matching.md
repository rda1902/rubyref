---
title: Pattern matching
prev: "/language/control-expressions.html"
next: "/language/methods-def.html"
---

## Pattern matching[](#pattern-matching)

<div class="since-version">Since Ruby 2.7</div>

Pattern matching is an experimental feature allowing deep matching of structured values: checking the structure, and binding the matched parts to local variables.

Pattern matching in Ruby is implemented with the `in` operator, which can be used in a standalone expression:


```
<variable> in <pattern>
```

or within the +case+ statement:


```
case <variable>
in <pattern1>
  ...
in <pattern2>
  ...
in <pattern3>
  ...
else
  ...
end
```

(Note that `in` and `when` branches can *not* be mixed in one `case` statement.)

Pattern matching is *exhaustive*\: if variable doesn't match pattern (in a separate `in` statement), or doesn't matches any branch of `case` statement (and `else` branch is absent), `NoMatchingPatternError` is raised.

Therefore, standalone `in` statement is most useful when expected data structure is known beforehand, to unpack parts of it:


```ruby
def connect_to_db(config) # imagine config is a huge configuration hash from YAML
  # this statement will either unpack parts of the config into local variables,
  # or raise if config's structure is unexpected
  config in {connections: {db: {user:, password:}}, logging: {level: log_level}}
  p [user, passsword, log_level] # local variables now contain relevant parts of the config
  # ...
end
```

whilst `case` form can be used for matching and unpacking simultaneously:


```ruby
case config
in String
  JSON.parse(config)                    # ...and then probably try to match it again

in version: '1', db:
  # hash with {version: '1'} is expected to have db: key
  puts "database configuration: #{db}"

in version: '2', connections: {database:}
  # hash with {version: '2'} is expected to have nested connection: database: structure
  puts "database configuration: #{database}"

in String => user, String => password
  # sometimes connection is passed as just a pair of (user, password)
  puts "database configuration: #{user}:#{password}"

in Hash | Array
  raise "Malformed config structure: #{config}"
else
  raise "Unrecognized config type: #{config.class}"
end
```

See below for more examples and explanations of the syntax.

### Patterns[](#patterns)

Patterns can be:

* any Ruby object (matched by `===` operator, like in +when+);
* array pattern: `[<subpattern>, <subpattern>, <subpattern>, ...]`
* hash pattern: `{key: <subpattern>, key: <supattern>, ...}`
* combination of patterns with `|`

Any pattern can be nested inside array/hash patterns where `<subpattern>` is specified.

Array patterns match arrays, or objects that respond to `deconstruct` (see below about the latter). Hash patterns match hashes, or objects that respond to `deconstruct_keys` (see below about the latter). Note that only symbol keys are supported for hash patterns, at least for now.

Important difference between array and hash patterns behavior is arrays match only a *whole* array


```ruby
case [1, 2, 3]
in [Integer, Integer]
  "matched"
else
  "not matched"
end
#=> "not matched"
```

while the hash matches even if there are other keys besides specified part:


```ruby
case {a: 1, b: 2, c: 3}
in {a: Integer}
  "matched"
else
  "not matched"
end
#=> "matched"
```

There is also a way to specify there should be no other keys in the matched hash except those explicitly specified by pattern, with `**nil`: 

```ruby
case {a: 1, b: 2}
in {a: Integer, **nil} # this will not match the pattern having keys other than a:
  "matched a part"
in {a: Integer, b: Integer, **nil}
  "matched a whole"
else
  "not matched"
end
#=> "matched a whole"
```

Both array and hash patterns support "rest" specification:


```ruby
case [1, 2, 3]
in [Integer, *]
  "matched"
else
  "not matched"
end
#=> "matched"

case {a: 1, b: 2, c: 3}
in {a: Integer, **}
  "matched"
else
  "not matched"
end
#=> "matched"
```

In `case` (but not in standalone `in`) statement, parentheses around both kinds of patterns could be omitted


```ruby
case [1, 2]
in Integer, Integer
  "matched"
else
  "not matched"
end
#=> "matched"

case {a: 1, b: 2, c: 3}
in a: Integer
  "matched"
else
  "not matched"
end
#=> "matched"
```

### Variable binding[](#variable-binding)

Besides deep structural checks, one of the very important features of the pattern matching is the binding of the matched parts to local variables. The basic form of binding is just specifying `=> variable_name` after the matched (sub)pattern (one might find this similar to storing exceptions in local variables in `rescue ExceptionClass => var` clause):


```ruby
case [1, 2]
in Integer => a, Integer
  "matched: #{a}"
else
  "not matched"
end
#=> "matched: 1"

case {a: 1, b: 2, c: 3}
in a: Integer => m
  "matched: #{m}"
else
  "not matched"
end
#=> "matched: 1"
```

If no additional check is required, only binding some part of the data to a variable, a simpler form could be used:


```ruby
case [1, 2]
in a, Integer
  "matched: #{a}"
else
  "not matched"
end
#=> "matched: 1"

case {a: 1, b: 2, c: 3}
in a: m
  "matched: #{m}"
else
  "not matched"
end
#=> "matched: 1"
```

For hash patterns, even a simpler form exists: key-only specification (without any value) binds the local variable with the key's name, too:


```ruby
case {a: 1, b: 2, c: 3}
in a:
  "matched: #{a}"
else
  "not matched"
end
#=> "matched: 1"
```

Binding works for nested patterns as well:


```ruby
case {name: 'John', friends: [{name: 'Jane'}, {name: 'Rajesh'}]}
in name:, friends: [{name: first_friend}, *]
  "matched: #{first_friend}"
else
  "not matched"
end
#=> "matched: Jane"
```

The "rest" part of a pattern also can be bound to a variable:


```ruby
case [1, 2, 3]
in a, *rest
  "matched: #{a}, #{rest}"
else
  "not matched"
end
#=> "matched: 1, [2, 3]"

case {a: 1, b: 2, c: 3}
in a:, **rest
  "matched: #{a}, #{rest}"
else
  "not matched"
end
#=> "matched: 1, {:b=>2, :c=>3}"
```

Binding to variables currently does NOT work for alternative patterns joined with `|`: 

```ruby
case {a: 1, b: 2}
in {a: } | Array
  "matched: #{a}"
else
  "not matched"
end
# SyntaxError (illegal variable in alternative pattern (a))
```

### Variable pinning[](#variable-pinning)

Due to variable binding feature, existing local variable can't be straightforwardly used as a sub-pattern:


```ruby
expectation = 18

case [1, 2]
in expectation, *rest
  "matched. expectation was: #{expectation}"
else
  "not matched. expectation was: #{expectation}"
end
# expected: "not matched. expectation was: 18"
# real: "matched. expectation was: 1" -- local variable just rewritten
```

For this case, "variable pinning" operator `^` can be used, to tell Ruby "just use this value as a part of pattern"


```ruby
expectation = 18
case [1, 2]
in ^expectation, *rest
  "matched. expectation was: #{expectation}"
else
  "not matched. expectation was: #{expectation}"
end
#=> "not matched. expectation was: 18"
```

One important usage of variable pinning is specifying the same value should happen in the pattern several times:


```ruby
jane = {school: 'high', schools: [{id: 1, level: 'middle'}, {id: 2, level: 'high'}]}
john = {school: 'high', schools: [{id: 1, level: 'middle'}]}

case jane
in school:, schools: [*, {id:, level: ^school}] # select the last school, level should match
  "matched. school: #{id}"
else
  "not matched"
end
#=> "matched. school: 2"

case john # the specified school level is "high", but last school does not match
in school:, schools: [*, {id:, level: ^school}]
  "matched. school: #{id}"
else
  "not matched"
end
#=> "not matched"
```

### Matching non-primitive objects: `deconstruct_keys` and `deconstruct`[](#matching-non-primitive-objects-deconstructkeys-and-deconstruct)

As already mentioned above, hash and array patterns besides literal arrays and hashes will try to match any object implementing `deconstruct` (for array patterns) or `deconstruct_keys` (for hash patterns).


```ruby
class Point
  def initialize(x, y)
    @x, @y = x, y
  end

  def deconstruct
    puts "deconstruct called"
    [@x, @y]
  end

  def deconstruct_keys(keys)
    puts "deconstruct_keys called with #{keys}"
    {x: @x, y: @y}
  end
end

case Point.new(1, -2)
in px, Integer  # subpatterns and variable binding works
  "matched: #{px}"
else
  "not matched"
end
# prints "deconstruct called"
"matched: 1"

case Point.new(1, -2)
in x: 0.. => px
  "matched: #{px}"
else
  "not matched"
end
# prints: deconstruct_keys called with [:x]
#=> "matched: 1"
```

`keys` are passed to `deconstruct_keys` to provide a room for optimization in the matched class: if calculating a full hash representation is expensive, one may calculate only the necessary subhash.

Additionally, when matching custom classes, expected class could be specified as a part of the pattern and is checked with `===`


```ruby
class SuperPoint < Point
end

case Point.new(1, -2)
in SuperPoint(x: 0.. => px)
  "matched: #{px}"
else
  "not matched"
end
#=> "not matched"

case SuperPoint.new(1, -2)
in SuperPoint[x: 0.. => px] # [] or () parentheses are allowed
  "matched: #{px}"
else
  "not matched"
end
#=> "matched: 1"
```

### Guard clauses[](#guard-clauses)

`if` can be used to attach an additional condition (guard clause) when the pattern matches


```ruby
case [1, 2]
in a, b if b == a*2
  "matched"
else
  "not matched"
end
#=> "matched"

case [1, 1]
in a, b if b == a*2
  "matched"
else
  "not matched"
end
#=> "not matched"
```

`unless` works, too:


```ruby
case [1, 1]
in a, b unless b == a*2
  "matched"
else
  "not matched"
end
#=> "matched"
```

### Current feature status[](#current-feature-status)

As of Ruby 2.7, feature is considered *experimental*\: its syntax can change in the future, and the performance is not optimized yet. Every time you use pattern matching in code, the warning will be printed:


```ruby
{a: 1, b: 2} in {a:}
# warning: Pattern matching is experimental, and the behavior may change in future versions of Ruby!
```

To suppress this warning, one may use newly introduced Warning::\[\]= method:


```ruby
Warning[:experimental] = false
{a: 1, b: 2} in {a:}
# ...no warning printed...
```

Alternatively, command-line key `-W:no-experimental` can be used to turn off "experimental" feature warnings.

One of the things developer should be aware of, which probably to be fixed in the upcoming versions, is that pattern matching statement rewrites mentioned local variables on partial match, *even if the whole pattern is not matched*.


```ruby
a = 5
case [1, 2]
in String => a, String
  "matched"
else
  "not matched"
end
#=> "not matched"
a
#=> 5  -- even partial match not happened, a is not rewritten

case [1, 2]
in a, String
  "matched"
else
  "not matched"
end
#=> "not matched"
a
#=> 1  -- the whole pattern not matched, but partial match happened, a is rewritten
```

Currently, the only core class implementing `deconstruct` and `deconstruct_keys` is Struct.


```ruby
 Point = Struct.new(:x, :y)
 Point[1, 2] in [a, b]
 # successful match
 Point[1, 2] in {x:, y:}
 # successful match
```

