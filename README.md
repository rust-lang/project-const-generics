# 👋🏽 Const Generics Project Group

<!--
 Status badge advertising the project as being actively worked on. When the
 project has finished be sure to replace the active badge with a badge
 like: https://img.shields.io/badge/status-archived-grey.svg
-->
![project group status: active](https://img.shields.io/badge/status-active-brightgreen.svg)
<!--
FIXME(website)
[![project group documentation](https://img.shields.io/badge/MDBook-View%20Documentation-blue)][gh-pages]
-->
The const generics project group implements and designs the `const_generics` feature. Please refer to our [charter] for more information on our goals and current scope.

Examples:

```rust
struct Foo<const N: usize> {
    field: [u8; N],
}

fn foo<const N: usize>() -> Foo<N> {
    Foo {
        field: [0; N],
    }
}

fn main() {
    match foo::<3>().field {
        [0, 0, 0] => {} // ok
        [_x, _y, _z] => panic!(),
    }
}
```

Welcome to the repository for the Const Generics Project Group! This is the
repository we use to organise our work. Please refer to our [charter] as well
as our [github pages website][gh-pages] for more information on our goals and
current scope.
[gh-pages]: https://rust-lang.github.io/{{GROUP_SLUG}}

[charter]: ./CHARTER.md


## How Can I Get Involved?

[You can find a list of the current members available
on `rust-lang/team`.][team-toml]

If you'd like to participate be sure to check out the relevant stream on [zulip][chat-link], feel free to introduce
yourself over there and ask us any questions you have.

[open issues]: /issues
[chat-link]: https://rust-lang.zulipchat.com/#narrow/stream/260443-project-const-generics
[team-toml]: https://github.com/rust-lang/team/blob/master/teams/project-const-generics.toml

## Building Documentation
This repository is also an mdbook project. You can view and build it using the
following command.

```
mdbook serve
```
