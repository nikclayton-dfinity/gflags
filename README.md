# gflags

[![Build Status](https://api.travis-ci.org/dtolnay/gflags.svg?branch=master)](https://travis-ci.org/dtolnay/gflags)
[![Latest Version](https://img.shields.io/crates/v/gflags.svg)](https://crates.io/crates/gflags)
[![Rust Documentation](https://img.shields.io/badge/api-rustdoc-blue.svg)](https://docs.rs/gflags)

Command line flag library in the style of [gflags (formerly Google Commandline
Flags)][gflags].

[gflags]: https://gflags.github.io/gflags/

```toml
[dependencies]
gflags = "0.1"
```

*Supports rustc 1.31+*

<br>

Quoting directly from the C++ gflags documentation, because the concept is
the same here:

> **Commandline flags** are flags that users specify on the command line when
> they run an executable. In the command
>
> ```text
> fgrep -l -f /var/tmp/foo johannes brahms
> ```
>
> `-l` and `-f /var/tmp/foo` are the two commandline flags. (`johannes` and
> `brahms`, which don't start with a dash, are **commandline arguments**.)
>
> Typically, an application lists what flags the user is allowed to pass in, and
> what arguments they take -- in this example, `-l` takes no argument, and `-f`
> takes a string (in particular, a filename) as an argument. Users can use a
> library to help parse the commandline and store the flags in some data
> structure.
>
> Gflags, the commandline flags library used within Google, differs from other
> libraries, such as `getopt()`, in that flag definitions can be scattered
> around the source code, and not just listed in one place such as `main()`. In
> practice, this means that a single source-code file will define and use flags
> that are meaningful to that file. Any application that links in that file will
> get the flags, and the gflags library will automatically handle that flag
> appropriately.
>
> There's significant gain in flexibility, and ease of code reuse, due to this
> technique.

This style of flag registration is better suited for large scale development
than maintaining a single central list of flags, as the central list would
become an endless source of merge conflicts in an application developed
simultaneously by hundreds of developers.

## Defining flags

Flags may be defined from any source file through the [`gflags::define!`] macro.
There is no central list of all the flags of the application. (That's the point
and advantage of gflags for large-scale development compared to other flags
libraries.)

[`gflags::define!`]: https://docs.rs/gflags/0.1/gflags/macro.define.html

```rust
gflags::define! {
    /// Include 'advanced' options in the menu listing.
    --big_menu = true
}

gflags::define! {
    /// Comma-separated list of languages to offer in the 'lang' menu.
    -l, --language <LANG> = "english,french,german"
}
```

Flags are required to have a long name (like `--verbose`) and may optionally
have a short name (like `-v`). Flags must have exactly one long name and at most
one short name; multiple different aliases for the same flag is not supported.

Flags of a type other than bool may have an optional value-placeholder like
`<LANG>`. This is optional and purely cosmetic. It appears in help text.

## Accessing flags

Somewhere early in your application, call [`gflags::parse()`] to parse the
command line. This call returns a `Vec<OsString>` containing everything on the
command line which is not a flag (these are sometimes known as positional
arguments) in a vector.

[`gflags::parse()`]: https://docs.rs/gflags/0.1/gflags/fn.parse.html

After `gflags::parse()` has been called, the value of each flag is available in
the `.FLAG` field of the flag's long name.

```rust
gflags::define! {
    --print-args = false
}

fn main() {
    let args = gflags::parse();

    if print_args.FLAG {
        println!("args = {:?}", args);
    }
}
```

As shown in this snippet, flag names may contain hyphens, in which case the
variable through which the flag's value can be accessed has underscores in place
of the hyphens.

Additionally every flag provides a method `.is_present()` to query whether that
flag was provided on the command line. When using flags for which a default
value is not provided, be sure to check `.is_present()` because accessing
`.FLAG` when not present will cause a panic. Note also that flags without a
default value must specify their data type, as below.

```rust
use std::path::Path;

gflags::define! {
    /// Search for patterns from the given file, with one pattern per line.
    -f, --file: &Path
}

fn main() {
    let patterns = gflags::parse();

    if file.is_present() {
        let path = file.FLAG;
        println!("searching for patterns from file: {}", path.display());
    } else {
        println!("searching for patterns given on command line: {:?}", patterns);
    }
}
```

## Printing help

There is no built-in `-h` flag for help, but you can define your own and call
[`gflags::print_help_and_exit()`] to render the documentation of all flags.

[`gflags::print_help_and_exit()`]: https://docs.rs/gflags/0.1/gflags/fn.print_help_and_exit.html

```rust
gflags::define! {
    -h, --help = false
}

fn main() {
    gflags::parse();
    if help.FLAG {
        gflags::print_help_and_exit(0);
    }

    /* ... */
}
```

For some of the flag definitions shown in this documentation, the help text
would be rendered as follows.

```text
        --big_menu
            Include 'advanced' options in the menu listing.

    -f, --file
            Search for patterns from the given file, with one pattern per line.

    -l, --language <LANG>
            Comma-separated list of languages to offer in the 'lang' menu.
```

The flags are listed in alphabetical order by long name.

You will likely want to print your own content above this including the
application name, version, author, introductory explanation, and usage strings.

## Custom data types

The `gflags::define!` macro is extensible to custom data types by providing an
impl of [`gflags::custom::Value`] for your type.

[`gflags::custom::Value`]: https://docs.rs/gflags/0.1/gflags/custom/trait.Value.html

```rust
use gflags::custom::{Arg, Error, Result, Value};

gflags::define! {
    --color <WHEN>: Color = Color::Auto
}

enum Color {
    Never,
    Always,
    Auto,
}

impl Value for Color {
    fn parse(arg: Arg) -> Result<Self> {
        match arg.get_string().as_str() {
            "never" => Ok(Color::Never),
            "always" => Ok(Color::Always),
            "auto" => Ok(Color::Auto),
            _ => Err(Error::new("invalid color")),
        }
    }
}
```

<br>

#### License

<sup>
Licensed under either of <a href="LICENSE-APACHE">Apache License, Version
2.0</a> or <a href="LICENSE-MIT">MIT license</a> at your option.
</sup>

<br>

<sub>
Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in this crate by you, as defined in the Apache-2.0 license, shall
be dual licensed as above, without any additional terms or conditions.
</sub>
