# Learning the ATS (Applied Type System) programming language

# Motivation

I've always been a big proponent of statically typed programming languages and specially the ones such as Haskell, Scala or Rust where the types can guide the programmer to avoid at least the most obvious mistakes.

I kept ATS on my radar as the next language to learn. I was curious to experiment programming with a low level language that supports dependent types, linear types and proofs .
## Learning ATS
The best way for me to learn is having a browser with some documentation, access to the standard library code source,  and a terminal with a simple editor (nvim) and a compiler.
Starting with a hello world and then introducing concepts one after the other, solving the compiler/runtime issue, rinse and repeat.

### Setup
I use `nix` and `direnv` for most of my programming activity, this is on a MacOS
```nix
# shell.nix
let pkgs = import <nixpkgs> {}; in
with pkgs;
mkShell {
  buildInputs = [ats2];
  PATSHOME = "${ats2}";
}
```
I then use `direnv` to have my environment ready whenever I enter the directory.
```
# .envrc
use nix
```

The simplest "hello X" example:
```ats
#include "share/atspre_define.hats"
#include "share/atspre_staload.hats"

implement main0 (argc, argv) =
	if argc > 1 then println!("hello ", argv[1])
```

There's already a lot going on here.
The `#include` lines have the same semantics as in C. In this case, they will add definitions and implementations of the prelude library of ATS.

`implement` indicates our intention to implement the `main0` interface, a signature we're going to have a look at shortly.

The `main0` interface is included  and we're required to implement its interface.
Here's how it's defined ( in `prelude/basics_dyn.sats`):
```ats
fun
main_0_argc_argv
  {n:int | n >= 1}
  (argc: int n, argv: !argv(n)): void = "ext#mainats_0_argc_argv"
// ...
overload main0 with main_0_argc_argv
```
This should be intimidating! Otherwise I'd like to know who is reading this? an LLM maybe?

In ATS, you can `overload` symbols (`main0`) with some functions (`main0_argc_argv`). So what we're really implementing is the interface of `main_0_argc_argv`

The `= "ext#mainats_0_argc_argv"` means that `main0_argc_argv` is treated as a global function in C with the name `mainats_0_argc_argv`. This we can see indeed with an `objdump` of our binary:
`0000000100003904 g     F __TEXT,__text _mainats_0_argc_argv`

Let's look at the most interesting part now, the `main0` interface:

```ats 
fun
main_0_argc_argv
  {n:int | n >= 1}
  (argc: int n, argv: !argv(n)): void = "ext#mainats_0_argc_argv"
```

1. `fun` : this is _one_ way to declare a function in ATS, there's also `fn` for non-recursive functions and `lam` for anonymous functions (obviously non-recursive)

2. `{n: int | n >= 1}`: the function is dependent at compile/static time on an int `n` that is greater or equal to 1.
3. `(argc: int n, argv: !argv(n))`: `argc` is an `int n`, not any int but the only int that has the value `n`. `argv` is of type `!argv(n)`. The `!` indicates that this value, if it is a linear type, will not be consumed by the end of the call.  It is also dependent on that same  `n`.

All we're doing here is constraining `main0`, we're saying that if the function is called then, there's an int `n`, greater or equal to 1 such that the value of `argc` is equal to `n` and `argv` is of type `argv(n)`.

I'll get back to what `argv` is later, also I'm intentionally avoiding introducing important concepts yet.

Continuing with the example we get to the implementation:
```ats
	if argc > 1 then println!("hello ", argv[1])
```

If we remove the test `argc > 1`, ATS complains with this error:
```
error(3): unsolved constraint: C3NSTRprop(C3TKmain(); S2Eapp(S2Ecst(<); S2EVar(5245->S2Eintinf(1)), S2Evar(n$3357(8480))))
typechecking has failed: there are some unsolved constraints: please inspect the above reported error message(s) for information.
```
Welcome to ATS compiler errors!
For this kind of errors, I was only equipped with some intuition. Like an LLM we're going to ignore tokens that look irrelevant:
```
unsolved constraint: (main(); < , 1 , n)
```
This is saying that the constraint `1 < n` can't be solved. However, because it compiled and worked fine before, it seems that the test `if argc > 1` solves the constraint.
Remember that `argc` is of type `int n` and hence equal to `n`, so what we're writing there is equivalent to `if n > 1`, which enables ATS to deduce that under that branch the constraint is solved, and we can access the element of `argv` at position `1` because `argv` has a size bigger than `1`
## `argv`

Usually in different languages, `argv` is most of the time an array of strings. So let's look at how `argv` is defined in ATS.

```ats
absvtype
argv_int_vtype (n:int) = ptr
stadef argv = argv_int_vtype
```
There's a new keyword (ignoring `stadef`): `absvtype`

`absvtype` defines a new abstract _view_ type `argv_int_vtype`. I'll explain views letter, and just focus on abstract now.

Here's a definition for ADT from [Liskov ADT](https://willguimont.github.io/cs/2019/01/27/abstract-data-type.html) 

> An abstract data type defines a class of abstract objects which is completely characterized by the operations available on those objects. This means that an abstract data type can be defined by defining the characterizing operations for that type. […] Implementation information, such as how the object is represented in storage, is only needed when defining how the characterizing operations are to be implemented. The user of the object is not required to know or supply this information.
   – Liskov & Zilles[1](https://willguimont.github.io/cs/2019/01/27/abstract-data-type.html#fn:Liskov)

In the code above `argv_int_vtype` is defined as a type constructor that take a type `int(n)` (remember that `int` up till now is a type constructor ) and returns the type `ptr` (pointer)
It will be not safe if we expose this pointer to the end user. So defining this as an abstract type makes sense. We're hiding this information and only exposing the possible operations that can be performed with the pointer.

Let's look at one of those operations:
```ats
fun
argv_get_at{n:int}
  (argv: !argv(n), i: natLt(n)):<> string = "mac#%"
```
The `"mac#%"` similarly to `"ext#"` is related to compatibility with C, and in this case it's saying that the function can be called as a macro (not sure what the  `%` means).

`argv_get_at` is dependent on a number `n`, and it says that given an `argv(n)`, and a number `i` that is smaller than `n` then we get a string.

The syntax `:<>` is for tagging functions with effects. In this case we explicitly say that there's no effect. However ATS won't let you throw that on a recursive function without proving that the function actually terminates. We'll get into tags in another post.
## Conclusion
ATS is an interesting programming language that can expose you to concepts such as dependent types, linearity or theorem proving. This post scratched the surface using a toy program. Surprisingly, the toy program exposed us almost to most of the features already. In the next article, I'll use the same toy program to explore in depth those concepts.
