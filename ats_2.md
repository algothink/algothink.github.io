# View types

## Introduction

In the previous post, we've seen the special type of `argv` which represents the null-terminated strings passed to our program from the command line.
ATS represents them through an abstract _view_ type or `absvtype` which I'm going to try to understand here.

## At Views

Many low level operations such as `malloc`, `free` or pointer arithmetics are available in ATS. However, a well typed ATS program will require proofs that
such operations are safe at compile time. These proofs are called `Views`.

`Views` have the sort `view` the same way a proof has a sort `prop`. The difference is that `Views` are linear, they can and have to be consumed
only once when passed to a function.

The simplest kind of views is `at-views` which are denoted by `T @ L` where `T` is a type and `L` is of sort `addr`.
`T @ L` states that there's a value of type `T` at location `L` while `T? @ L` assumes the memory at location `L` is uninitialized.

It follows from above the importance for views to be linear. We obviously don't want to hold at the same time a proof of `T @ L` and `T? @ L`.

Let's try something completely unsafe and see what ATS says.

```ats
fun unsafeinit{l: addr}(p: ptr(l)): void =
  !p := 0
```

We get this error:
```
assignment cannot be performed: proof search for the view at [S2Evar(l(8482))] failed.
```

We need to provide ATS a view that we're not unsafely writing memory (at least there's an uninitialized memory for an int at addres `l`).


```ats
fun init{l: addr}(v: int? @ l | p: ptr(l)): void =
  !p := 0
```

However, we get another error:
```
the linear dynamic variable [v] needs to be consumed but it is preserved with the type [S2Eat(S2Eapp(S2Ecst(g1int_int_t0ype); S2Eextkind(atstype_int), S2Eintinf(0)), S2Evar(l(8480)))] instead.
```
So apparently we don't consume `int? @ l`, instead we preserve it with a mysterious type (spoiler `int(0) @ l`).

The fix is to use the `!` prefix which preserves the proof.

```ats
fun init{l: addr}(v: !int? @ l | p: ptr(l)): void =
  !p := 0

```

This compiles but if we try to use it now:
```ats
implement main0 () =
let
  var e: int with prf
  val () = init(prf | addr@(e))
  val () = println!(e)
in () end
```
We get another error:

```
the symbol [print] cannot be resolved due to too many matches:
D2ITMcst(print_option) of 0
D2ITMcst(print_list_vt) of 0
```
The error is rather ambiguous but we can see that ATS doesn't know what `e` is and hence can't find an override of `print` to print it.
The issue is that after calling `init` we kept the same initial belief that is `int? @ l`

To fix this we change the signature of init, to state that after the call the proof changes to `int @ l`.

```ats
#include "share/atspre_define.hats"
#include "share/atspre_staload.hats"


fun init{l: addr}(v: !int? @ l >> int @ l | p: ptr(l)): void =
  !p := 0

implement main0 () =
let
  var e: int with prf
  val () = init(prf | addr@(e))
  val () = println!(e)
in () end
```
In fact the view `!int? @ l >> int @ l` can be found in the signature of `ptr_set` (which I beleive is called behind the scenes by `!p := 0`)

## View types

At-views are not the only views available in ATS. In fact `@` is an alias to an abstract view (`absview`) defined in ATS:
```ats
absview // S2Eat
at_vt0ype_addr_view(a:vt@ype+, l:addr)
//
viewdef @ // HX: @ is infix
  (a:vt@ype, l:addr) = at_vt0ype_addr_view(a, l)
```


## Abstract View types

Sometimes it's a good practice to not give away the information about how an ADT is represented (it allows you to change the implementation without
affecting too much the client code). That's where the `absvtype` that we've seen in `argv` enters in place.

```ats
absvtype
argv_int_vtype (n:int) = ptr
stadef argv = argv_int_vtype
```
As we can see above, `argv` is a pointer so we need two things:
- Not expose the fact that it's a pointer which will allow clients to use any `ptr_xxx` functions to modify it.
- Track it as a resource which is what linearity is good at.

## Conclusion

ATS provides a way to combine code and proofs that can help better express invariants about our code. It also improves management of resources through
support of linearity.
