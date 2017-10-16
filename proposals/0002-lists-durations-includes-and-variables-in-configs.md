# Lists, Durations, Includes, and Variables in Configs

* Proposal: [SE-0002](0002-config-structuring-improvements.md)
* Author: [Dominik Charousset](https://github.com/neverlord)
* Status: Awaiting 

## Introduction

The `actor_system_config` with its CLI and INI parsers exist because CAF needs
powerful and flexible ways to tweak CAF applications without recompilations.
Configuring the applications themselves is also easy to achieve since the
configuration is easily extensible. However, the current implementation lacks a
few basic types and additional features would greatly improve its usefulness.

## Motivation

This proposal discusses improvements to CAF's configuration mechanism in
various categories.

### Missing Duration Type

Configurations commonly include time intervals. CAF exposes the polling
intervals of the scheduler via its config, for example. However, durations are
not treated as first-class entity. As a result, users have to pick a resolution
(milliseconds, for example) and read integers from the config.

### Data Structures

Recursive config types are unsupported in CAF. Any key/value pair in the INI
file creates a pair of string (key) and `config_value` (value). The latter is
defined as `variant<string, double, int64_t, bool, atom_value>`. This building
block lacks any support for data structures like lists and maps. An
undocumented feature of CAF's parsers already enables lists, albeit very
inelegantly. This example illustrates a custom configuration with a list of
values:

```cpp
struct config : actor_system_config {
  std::vector<string> hosts;
  config() {
    opt_group{custom_options_, "global"}
    .add(hosts, "hosts,h", "set hosts");
  }
};
```

Now, each time the value is set using an CLI parameter CAF actually appends a
value to the list (an undocumented feature). For example,
`./app -h host1 -h host2 -h host3` will fill the list with the strings "host1",
"host2", and "host3". This behavior is counterintuitive. Even more so when
doing the same per config file:

```ini
[global]
hosts="host1"
hosts="host2"
hosts="host3"
```

CAF happily adds each entry to the list, but that's a "feature" (hack, really)
of the config reader. The CLI and INI parsers are completely unaware of lists,
let alone maps. Maps could be easily supported by adding pairs and lists.

### Error Handling and Dynamic Configuration Parameters

The actor system config drops all unknown parameters, printing error messages
for unrecognized options. This has several downsides:

1. There is no generic way to access the whole configuration. For example,
   simply iterating the configuration as-is with proper type information is not
   possible. One can iterate the set of all options, but that
   [isn't straightforward](https://github.com/actor-framework/actor-framework/blob/80b973/libcaf_core/src/actor_system_config.cpp#L438).
2. Actors can only access the configuration as `actor_system_config` via
   `self->system().config()`. This means users have no easy access to any
   option specified in a subtype (unless using `dynamic_cast` and friends).
3. Fully dynamic parsing of the received config is simply impossible because
   all "unknown" options are dropped. The only supported way to read a config
   is by subtyping `actor_system_config` and filling `custom_options_`.

Having a `map<string, map<string, config_value>>` (i.e. mapping groups to the
key/value pairs for options) in the `actor_system_config` for storing *all*
parameters given by the user (or present by default) would pave the way to give
users access to a "raw" representation of the whole system configuration.

The one case where the configuration should drop configuration values with an
error message is when trying to override a value with a different type. The
following example should raise an error:

```ini
[scheduler]
max-threads="foo" ; type mismatch: trying to override an integer with a string
```

### Accessing Environment Variables

Using environment variables for defining config parameters requires using the
CLI parameters. Allowing the INI parser to expand environment variables closes
the gap in functionality and makes distributing CAF applications easier.

### Convenience Features

Splitting complex configurations into multiple files is a common use case but
unsupported. 

Finally, each group in the INI file is essentially a list of key/value pairs,
i.e., a map. Allowing users to reference other groups as key/value lists could
make composing complex configurations much easier.

## Proposed Solution

The parsers should receive first-class support for lists, durations,
environment variables, and include directives.

### Data Structures and Missing Duration Type

This is the current definition of the `config_value` building block:

```cpp
using config_value = variant<string, double, int64_t, bool, atom_value>;
```

The variant should include `caf::duration` and support lists:

```cpp
using config_value = variant<string, double, int64_t, bool, atom_value,
                             duration, std::vector<config_value>>;
```

A list can easily represent a map as a list of pairs (i.e. lists with two
elements each).

The parses should accept the following syntax:
- `key=infinite` should be parsed as `caf::infinite`
- `key=(x0, x1, ..., xn)` should be parsed as a list
- `key=(("a", 1), ("b", 2))` is a list of lists, allowing users to model maps
- `key=@other` assigns a config group converted into a list of key/value pairs
  to a single key; here's a motivating example with two identically defined
  parameters `bar1.foo` and `bar2.foo`:
```ini
[foo]
min=5
max=10
[bar1]
foo=@foo
[bar2]
foo=(("min", 5), ("max", 10))
```

The current hack for appending with multiple assignments should be removed in
favor of a syntactically clean solution. A natural candidate is using the `+=`
operator:

```ini
list=(1, 2, 3)
list+=(4, 5, 6)
; list is now (1, 2, 3, 4, 5, 6)
```

Of course this operator could operate on any type, not just lists.

### Accessing Variables

Adopting the `$x` notation for accessing environment variables is most
intuitive. We could also extend the `@`-notation to mean "local variable from
this config" and also allow `@group.key` to pick individual values from a
group. In this way, each group and each key automatically becomes a local
variable.

### Includes

Currently, each line is either empty or contains a `[group]` or `key=value`
definition. Includes would add direct commands to the parser. Using an operator
for this seems natural and `!` a good prefix. Introducing the prefix allows us
to extend this functionality in the future, for example "!print @my-group"
could make it easier to debug complex configurations.

The INI parser could restrict include files to be declared at the top of the
file or simply read any file content in-place. Consider this example:

```ini
!read /path/to/file1

[somegroup]
!read /path/to/file2
```

If `!read file` is allowed everywhere than it would operate in the same way a
C++ `include` does, i.e., simple text replacement. This is the most flexible
solution, since it allows including key/value definitions from a file
multiple times.

### Additional State in `actor_system_config`

The system config should keep a `map<string, map<string, config_value>>` for
storing all parameters. This map is pre-filled with the hard-coded defaults of
CAF on startup and updated from the INI and CLI parsers. This map should be
readable by users.

Further, the config class should provide functions for conveniently looking up
a value. For example: `cfg.value("global", "foobar")` could give quick access
to individual parameters.

## Impact on Existing Code

1. The `config_value` type becomes more complex.
2. INI and CLI parsers need to be updated and improved.
3. The class `actor_system_config` gets more state and convenience functions

## Alternatives

Lists already have quirky support. Building on this would avoid complexity in
the parsers. However, the long term inconvenience for users outweighs any
short-term benefits of not updating the parsers.

Alternative to the proposed syntax changes could be considered. For example,
PHP's INI parser accepts the following notation for lists and maps:

```ini
[somegroup]
somelist[] = "a"
somelist[] = "b"
somelist[] = "c"
somelist[] = "d"
urls[svn] = "http://svn.php.net"
urls[git] = "http://git.php.net"
```

This style of list support is easy to add on top of the current undocumented
behavior. The corresponding syntax using our proposed solution reads:

```ini
[somegroup]
somelist = ("a", "b", "c", "d")
urls = (("svn", "http://svn.php.net"), ("git","http://git.php.net"))
```

The proposed solution is less redundant.