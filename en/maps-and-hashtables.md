# Maps and Hashtables

Lots of programming problems require dealing with data organized as
key/value pairs.  Maybe the simplest way of representing such data in
OCaml is an _association list_, which is simply a list of pairs of
keys and values.  For example, you could represent a mapping between
the 10 digits and their English names as follows.

```ocaml
# let digit_alist =
    [ 0, "zero"; 1, "one"; 2, "two"  ; 3, "three"; 4, "four"
    ; 5, "five"; 6, "six"; 7, "seven"; 8, "eight"; 9, "nine" ]
  ;;
```

We can use functions from the `List.Assoc` module to manipulate such
an association list.

```ocaml
# List.Assoc.find digit_alist 6;;
- : string option = Some "six"
# List.Assoc.find digit_alist 22;;
- : string option = None
# List.Assoc.add digit_alist 0 "zilch";;
- : (int, string) List.Assoc.t =
[(0, "zilch"); (1, "one"); (2, "two"); (3, "three"); (4, "four");
 (5, "five"); (6, "six"); (7, "seven"); (8, "eight"); (9, "nine")]
```

Association lists are simple and easy to use, but their performance is
not ideal, since almost every non-trivial operation on an association
list requires a linear-time scan of the list.

In this chapter, we'll talk about two more efficient alternatives to
association lists: _maps_ and _hashtables_.  A map is an immutable
tree-based data structure where most operations take time logarithmic
in the size of the map, whereas a hashtable is a mutable data
structure where most operations have constant time complexity.  We'll
describe both of these data structures in detail, and provide some
advice as to how to choose between them.

## Maps

Let's consider an example of how one might use a map in practice.  In
[xref](#files-modules-and-programs), we showed a module `Counter` for
keeping frequency counts on a set of strings.  Here's the interface.

```ocaml
(* counter.mli *)
open Core.Std

type t

val empty : t
val touch : t -> string -> t
val to_list : t -> (string * int) list
```

The intended behavior here is straightforward.  `Counter.empty`
represents an empty collection of frequency counts; `touch` increments
the frequency count of the specified string by 1; and `to_list`
returns the list of non-zero frequencies.

Here's the implementation.

```ocaml
(* counter.ml *)
open Core.Std

type t = int String.Map.t

let empty = String.Map.empty

let to_list t = Map.to_alist t

let touch t s =
  let count = Option.value ~default:0 (Map.find t s) in
  Map.add t ~key:s ~data:(count + 1)
```

Note that in some places the above code refers to `String.Map.t`, and
in others `Map.t`.  This has to do with the fact that maps are
implemented as ordered binary trees, and as such, need a way of
comparing keys.

To deal with this, a map, once created, stores the necessary
comparison function within the data structure.  Thus, operations like
`Map.find` or `Map.add` that access the contents of a map or create a
new map from an existing one, do so by using the comparison function
embedded within the map.

But in order to get a map in the first place, you need to get your
hands on the comparison function somehow.  For this reason, modules
like `String` contain a `Map` sub-module that have values like
`String.Map.empty` and `String.Map.of_alist` that are specialized to
strings, and thus have access to a string comparison function.  Such a
`Map` sub-module is included in every module that satisfies the
`Comparable.S` interface from Core.

### Creating maps with comparators

The specialized `Map` sub-module is convenient, but it's not the only
way of creating a `Map.t`.  The information required to compare values
of a given type is wrapped up in a value called a _comparator_, that
can be used to create maps using the `Map` module directly.

```ocaml
# let digit_map = Map.of_alist_exn digit_alist
                     ~comparator:Int.comparator;;
val digit_map : (int, string, Int.comparator) Map.t = <abstr>
# Map.find digit_map 3;;
- : string option = Some "three"
```

The above uses `Map.of_alist_exn` which creates a map from an
association list, throwing an exception if there are duplicate keys in
the list.

The comparator is only required for operations that create maps from
scratch.  Operations that update an existing map simply inherit the
comparator of the map they start with.

```ocaml
# let zilch_map = Map.add digit_map ~key:0 ~data:"zilch";;
val zilch_map : (int, string, Int.comparator) Map.t = <abstr>
```

The type `Map.t` has three type parameters: one for the key, one for
the value, and one to identify the comparator.  Indeed, the type `'a
Int.Map.t` is just a type alias for `(int,'a,Int.comparator) Map.t`

Including the comparator in the type is important because because
operations that work on multiple maps at the same time often require
that the maps share their comparison function.  Consider, for example,
`Map.symmetric_diff`, which computes a summary of the differences
between two maps.

```ocaml
# let left = String.Map.of_alist_exn ["foo",1; "bar",3; "snoo", 0]
  let right = String.Map.of_alist_exn ["foo",0; "snoo", 0]
  let diff = Map.symmetric_diff ~data_equal:Int.equal left right
  ;;
val left : int String.Map.t = <abstr>
val right : int String.Map.t = <abstr>
val diff :
  (string * [ `Left of int | `Right of int | `Unequal of int * int ]) list =
  [("foo", `Unequal (1, 0)); ("bar", `Left 3)]
```

The type of `Map.symmetric_diff`, shown below, requires that the two
maps it compares have the same comparator type.  Each comparator has a
fresh abstract type, so the type of a comparator identifies the
comparator uniquely.  

```ocaml
# Map.symmetric_diff;;
- : ('k, 'v, 'cmp) Map.t ->
    ('k, 'v, 'cmp) Map.t ->
    data_equal:('v -> 'v -> bool) ->
    ('k * [ `Left of 'v | `Right of 'v | `Unequal of 'v * 'v ]) list
= <fun>
```

This constraint is important because the algorithm that
`Map.symmetric_diff` uses depends on the fact that both maps have the
same comparator.

We can create a new comparator using the `Comparator.Make` functor,
which takes as its input a module containing the type of the object to
be compared, sexp-converter functions, and a comparison function.  The
sexp converters are included in the comparator to make it possible for
users of the comparator to generate better error messages.  Here's an
example.

```ocaml
# module Reverse = Comparator.Make(struct
    type t = string
    let sexp_of_t = String.sexp_of_t
    let t_of_sexp = String.t_of_sexp
    let compare x y = String.compare y x
  end);;
module Reverse :
  sig
    type t = string
    val compare : t -> t -> int
    val t_of_sexp : Sexp.t -> t
    val sexp_of_t : t -> Sexp.t
    type comparator
    val comparator : (t, comparator) Comparator.t_
  end
```

As you can see below, both `Reverse.comparator` and
`String.comparator` can be used to create maps with a key type of
`string`.

```ocaml
# let alist = ["foo", 0; "snoo", 3];;
val alist : (string * int) list = [("foo", 0); ("snoo", 3)]
# let ord_map = Map.of_alist_exn ~comparator:String.comparator alist;;
val ord_map : (string, int, String.comparator) Map.t = <abstr>
# let rev_map = Map.of_alist_exn ~comparator:Reverse.comparator alist;;
val rev_map : (string, int, Reverse.comparator) Map.t = <abstr>
```

`Map.min_elt` returns the key and value for the smallest key in the
map, which lets us see that these two maps do indeed use different
comparison functions.

```ocaml
# Map.min_elt ord_map;;
- : (string * int) option = Some ("foo", 0)
# Map.min_elt rev_map;;
- : (string * int) option = Some ("snoo", 3)
```

And accordingly, if we try to use `Map.symmetric_diff` on these two
maps, we'll get a compile-timer error.

```ocaml
# Map.symmetric_diff ord_map rev_map;;

Error: This expression has type (string, int, Reverse.comparator) Map.t
       but an expression was expected of type
         ('a, 'b, 'c) Map.t = (string, int, String.comparator) Map.t
       Type Reverse.comparator is not compatible with type String.comparator 
```

### Trees

As we've discussed, maps carry within them the comparator that they
were created with.  Sometimes, often for space efficiency reasons, you
want a version of the map data structure that doesn't include the
comparator.  You can get such a representation with `Map.to_tree`,
which returns just the tree that the map is built out of, and not
including the comparator.

```ocaml
# let ord_tree = Map.to_tree ord_map;; 
val ord_tree : (string, int, String.comparator) Map.Tree.t = <abstr>
```

Even though a `Map.Tree.t` doesn't physically include a comparator, it
does include the comparator in its type.  This is what is known as a
_phantom type parameter_, because it reflects something about the
logic of value in question, even though it doesn't correspond to any
values directly represented in the underlying physical structure of
the value.

Since the comparator isn't included in the tree, we need to provide
the comparator explicitly when we, say, search for a key, as shown
below.

```ocaml
# Map.Tree.find ~comparator:String.comparator ord_tree "snoo";;
- : int option = Some 3
```

The algorithm of `Map.Tree.find` depends on the fact that it's using
the same comparator when looking a value up as you were when you
stored it.  That's the invariant that the phantom type is there to
enforce.  As you can see below, using the wrong comparator will lead
to a type error.

```ocaml
# Map.Tree.find ~comparator:Reverse.comparator ord_tree "snoo";;

Error: This expression has type (string, int, String.comparator) Map.Tree.t
       but an expression was expected of type
         ('a, 'b, 'c) Map.Tree.t = (string, 'b, Reverse.comparator) Map.Tree.t
       Type String.comparator is not compatible with type Reverse.comparator 
```

### The polymorphic comparator

We don't need to generate specialized comparators for every type we
want to build a map on.  We can instead use a comparator based on
OCaml's build-in polymorphic comparison function, which was discussed
in [xref](#lists-and-patterns).  This comparator is found in the
`Comparator.Poly` module, allowing us to write:

```ocaml
# Map.of_alist_exn ~comparator:Comparator.Poly.comparator digit_alist;;
- : (int, string, Comparator.Poly.comparator) Map.t = <abstr>
```

Or, equivalently:

```ocaml
# Map.Poly.of_alist_exn digit_alist;;
- : (int, string) Map.Poly.t = <abstr>
```

Note that maps based on the polymorphic comparator are not equivalent
to those based on the type-specific comparators from the point of view
of the type system.  Thus, the compiler rejects the following:

```ocaml
# Map.symmetric_diff (Map.Poly.singleton 3 "three")
                     (Int.Map.singleton  3 "four" ) ;;

Error: This expression has type 'a Int.Map.t = (int, 'a, Int.comparator) Map.t
       but an expression was expected of type
         ('b, 'c, 'd) Map.t = (int, string, Z.Poly.comparator) Map.t
       Type Int.comparator is not compatible with type Z.Poly.comparator 
```

This is rejected for good reason: there's no guarantee that the
comparator associated with a given type will order things in the same
way that polymorphic compare does.

### Sets

Sometimes, instead of keeping track of a set of key/value pairs, you
just want a data-type for keeping track of a set of keys.  You could
build this on top of a map by representing a set of values by a map
whose data type is `unit`.  But a more idiomatic (and efficient)
solution is to use Core's set type, which is similar in design and
spirit to the map type, while having an API better tuned to working
with sets, and a lower memory footprint.  Here's a simple example:

```ocaml
# let dedup ~comparator l =
    List.fold l ~init:(Set.empty ~comparator) ~f:Set.add
    |> Set.to_list
  ;;
val dedup : comparator:('a, 'b) Core.Comparator.t_ -> 'a list -> 'a list =
  <fun>
# dedup ~comparator:Int.comparator [8;3;2;3;7;8;10];;
- : int list = [2; 3; 7; 8; 10]
```

In addition to the operators you would expect to have for maps, sets
support the traditional set operations, including union, intersection
and set difference.  And, as with maps, we can create sets based on
type-specific comparators or on the polymorphic comparator.

<warning> <title> The perils of polymorphic compare </title>

Polymorphic compare is highly convenient, but it has serious downsides
as well, and should be used with care.  In particular, polymorphic
compare has a fixed algorithm for comparing values of any type, and
that algorithm can sometimes yield surprising results.

To understand what's wrong with polymorphic compare, you need to
understand a bit about how it works.  Polymorphic compare is
_structural_, in that it operates directly on the
runtime-representation of OCaml values, walking the structure of the
values in question without regard for their type.

This is convenient because it provides a comparison function that
works for most OCaml values, and largely behaves as you would expect.
For example, on `int`s and `float`s it acts as you would expect a
numeric comparison function to act.  For simple containers like
strings and lists and arrays it operates as a lexicographic
comparison.  And except for closures and values from outside of the
OCaml heap, it works on almost every OCaml type.

But sometimes, a structural comparison is not what you want.  Sets are
a great example of this.  Consider the following two sets.

```ocaml
# let (s1,s2) = (Int.Set.of_list [1;2],
                 Int.Set.of_list [2;1]);;
val s1 : Int.Set.t = <abstr>
val s2 : Int.Set.t = <abstr>
```

Logically, these two sets should be equal, and that's the result that
you get if you call `Set.equal` on them.

```ocaml
# Set.equal s1 s2;;
- : bool = true
```

But because the elements were added in different orders, the layout of
the trees underlying the sets will be different.  As such, a
structural comparison function will conclude that they're different.

Let's see what happens if we use polymorphic compare to test for
equality by way of the `=` operator.  Comparing the maps directly will
fail at runtime because the comparators stored within the sets contain
function values.

```ocaml
# s1 = s2;;
Exception: (Invalid_argument "equal: functional value").
```

We can however use the function `Set.to_tree` to expose the underlying
tree without the attached comparator.

```ocaml
# Set.to_tree s1 = Set.to_tree s2;;
- : bool = false
```

This can cause real and quite subtle bugs.  If, for example, you use a
map whose keys contain sets, then the map built with the polymorphic
comparator will behave incorrectly, separating out keys that should be
aggregated together.  Even worse, it will work sometimes and fail
others, since if the sets are built in a consistent order, then they
will work as expected, but once the order changes, the behavior will
change.

For this reason, it's preferable to avoid polymorphic compare for
serious applications.

</warning>

### Satisfying the `Comparable.S` interface

Core's `Comparable.S` interface includes a lot of useful
functionality, including support for working with maps and sets.  In
particular, `Comparable.S` requires the presence of the `Map` and
`Set` sub-modules as well as a comparator.

`Comparable.S` is satisfied by most of the types in Core, but the
question arises of how to satisfy the comparable interface for a new
type that you design.  Certainly implementing all of the required
functionality from scratch would be an absurd amount of work.

The module `Comparable` contains a number of functors to help you do
just this.  The simplest one of these is `Comparable.Make`, which
takes as an input any module that satisfies the following interface:

```ocaml
sig
  type t
  val sexp_of_t : t -> Sexp.t
  val t_of_sexp : Sexp.t -> t
  val compare : t -> t -> int
end
```

In other words, it expects a type with a comparison function as well
as functions for converting to and from _s-expressions_.
S-expressions are a serialization format used commonly in Core, which
we'll discuss more in [xref](#data-serialization-with-s-expressions).
In the meantime, we can just use the `with sexp` declaration that
comes from the `sexplib` syntax extension to create s-expression
converters for us.  S-expression converters can also be written by
hand.

The following example shows how this all fits together, following the
same basic pattern for using functors described in
[xref](#extending-modules).

```ocaml
# module Foo_and_bar : sig
    type t = { foo: Int.Set.t; bar: string }
    include Comparable.S with type t := t
  end = struct
    module T = struct
      type t = { foo: Int.Set.t; bar: string } with sexp
      let compare t1 t2 =
        let c = Int.Set.compare t1.foo t2.foo in
        if c <> 0 then c else String.compare t1.bar t2.bar
    end
    include T
    include Comparable.Make(T)
  end;;
```

We don't include the full response from the top-level because it is
quite lengthy, but `Foo_and_bar` does satisfy `Comparable.S`.

In the above, we wrote the comparison function by hand, but this isn't
strictly necessary.  Core ships with a syntax extension called
`comparelib` which will create a comparison function from a type
definition.  Using it, we can rewrite the above example as follows.

```ocaml
# module Foo_and_bar : sig
    type t = { foo: Int.Set.t; bar: string }
    include Comparable.S with type t := t
  end = struct
    module T = struct
      type t = { foo: Int.Set.t; bar: string } with sexp, compare
    end
    include T
    include Comparable.Make(T)
  end;;
```

The comparison function created by `comparelib` for a given type will
call out to the comparison functions for its component types.  As a
result, the `foo` field will be compared using `Int.Set.compare`.
This is different, and sander, than the structural comparison done by
polymorphic compare.  

If you want your comparison function to behave in a specific way, you
should still write your own comparison function by hand; but if all
you want is a total order suitable for creating maps and sets with,
then `comparelib` is a good way to go.

You can also satisfy the `Comparable.S` interface using polymorphic
compare.

```ocaml
# module Foo_and_bar : sig
    type t = { foo: int; bar: string }
    include Comparable.S with type t := t
  end = struct
    module T = struct
      type t = { foo: int; bar: string } with sexp
    end
    include T
    include Comparable.Poly(T)
  end;;
```

That said, for reasons we discussed earlier, polymorphic compare
should be used sparingly.

## Hashtables

Hashtables are the imperative cousin of maps.  We walked over a basic
hashtable implementation in [xref](#imperative-programming-1), so in
this section we'll mostly discuss the pragmatics of Core's `Hashtbl`
module.  We'll cover this material more briefly than we did with maps,
because many of the concepts are shared.

Hashtables differ from maps in a few key ways.  First, hashtables are
mutable, meaning that adding a key/value pair to a hashtable modifies
the table, rather than creating a new table with the binding added.
Second, hashtables generally have better time-complexity than maps,
providing constant time lookup and modifications as opposed to
logarithmic for maps.  And finally, just as maps depend on having a
comparison function for creating the ordered binary tree that
underlies a map, hashtables depend on having a _hash function_,
_i.e._, a function for converting a key to an integer.

When creating a hashtable, we need to provide a value of type
_hashable_ which includes among other things the function for hashing
the key type.  This is analogous to the comparator used for creating
maps.

```ocaml
# let table = Hashtbl.create ~hashable:String.hashable ();;
val table : (string, '_a) Hashtbl.t = <abstr>
# Hashtbl.replace table ~key:"three" ~data:3;;
- : unit = ()
# Hashtbl.find table "three";;
- : int option = Some 3
```

The `hashable` value is included as part of the `Hashable.S`
interface, which is satisfied by most types in Core.  The `Hashable.S`
interface also includes a `Table` sub-module which provides more
convenient creation functions.

```ocaml
# let table = String.Table.create ();;
val table : '_a String.Table.t = <abstr>
```

There is also a polymorphic `hashable` value, corresponding to the
polymorphic hash function provided by the OCaml runtime, for cases
where you don't have a hash function for your specific type.

```ocaml
# let table = Hashtbl.create ~hashable:Hashtbl.Poly.hashable ();;
val table : ('_a, '_b) Hashtbl.t = <abstr>
```

Or, equivalently:

```ocaml
# let table = Hashtbl.Poly.create ();;
val table : ('_a, '_b) Hashtbl.t = <abstr>
```

Note that, unlike the comparators used with maps and sets, hashables
don't show up in the type of a `Hashtbl.t`.  That's because hashtables
don't have operations that operate on multiple hashtables that depend
on those tables having the same hash function, in that way that
`Map.symmetric_diff` and `Set.union` depend on their arguments using
the same comparison function.

### Satisfying the `Hashable.S` interface

Most types in Core satisfy the `Hashable.S` interface, but as with the
`Comparable.S` interface, the question remains of how one should
satisfy this interface with a new type.  Again, the answer is to use a
functor to build the necessary functionality; in this case,
`Hashable.Make`.  Note that we use OCaml's `lxor` operator for doing
the "logical" (_i.e._, bit-wise) exclusive-or of the hashes from the
component values.

```ocaml
# module Foo_and_bar : sig
    type t = { foo: int; bar: string }
    include Hashable.S with type t := t
  end = struct
    module T = struct
      type t = { foo: int; bar: string } with sexp, compare
      let hash t =
        (Int.hash t.foo) lxor (String.hash t.bar)
    end
    include T
    include Hashable.Make(T)
  end;;
```

Note that in order to satisfy hashable, one also needs to provide a
comparison function.  That's because Core's hashtables use ordered
binary tree data-structure for the hash-buckets, so that performance
of the table degrades gracefully in the case of pathologically bad
choice of hash function.

There is currently no analogue of `comparelib` for auto-generation of
hash-functions, so you do need to either write the hash-function by
hand, or use the built-in polymorphic hash function, `Hashtbl.hash`.

## Choosing between maps and hashtables

Maps and hashtables overlap enough in functionality that it's not
always clear when to choose one or the other.  Maps, by virtue of
being immutable, are generally the default choice in OCaml by virtue
of fitting most naturally with otherwise functional code.  OCaml also
has good support for imperative programming, though, and when
programming in an imperative idiom, hashtables are often the more
natural choice.

Programming idioms aside, there are significant performance
differences between maps and hashtables as well.  For code that is
dominated by updates and lookups, hashtables are a clear performance
win, and the win is clearer the larger the size of the tables.

The best way of answering a performance question is by running a
benchmark, so let's do just that.  The following benchmark uses the
`core_bench` library, and it compares maps and hashtables under a very
simple workload.  Here, we're keeping track of a set of 1000 different
integer keys, and cycling over the keys and updating the values they
contain.  Note that we use the `Map.change` and `Hashtbl.change`
functions to update the respective data structures.

```ocaml
(* file: map_vs_hash.ml *)

open Core.Std
open Core_bench.Std

let map_iter ~num_keys ~iterations =
  let rec loop i map =
    if i <= 0 then ()
    else loop (i - 1)
           (Map.change map (i mod num_keys) (fun current ->
              Some (1 + Option.value ~default:0 current)))
  in
  loop iterations Int.Map.empty

let table_iter ~num_keys ~iterations =
  let table = Int.Table.create ~size:num_keys () in
  let rec loop i =
    if i <= 0 then ()
    else (
      Hashtbl.change table (i mod num_keys) (fun current ->
        Some (1 + Option.value ~default:0 current));
      loop (i - 1)
    )
  in
  loop iterations

let tests ~num_keys ~iterations =
  let test name f = Bench.Test.create f ~name in
  [ test "map"   (fun () -> map_iter   ~num_keys ~iterations)
  ; test "table" (fun () -> table_iter ~num_keys ~iterations)
  ]

let () =
  tests ~num_keys:1000 ~iterations:100_000
  |> Bench.make_command
  |> Command.run
```


The results, shown below, show the hashtable version to be around four
times faster than the map version.  

```
bench $ ./map_vs_hash.native -clear-columns name time speedup
Estimated testing time 20s (change using -quota SECS).
┌───────┬────────────┬─────────┐
│ Name  │  Time (ns) │ Speedup │
├───────┼────────────┼─────────┤
│ map   │ 31_584_468 │    1.00 │
│ table │  8_157_439 │    3.87 │
└───────┴────────────┴─────────┘
```

We can make the speedup smaller or larger depending on the details of
the test; for example, it will very with the number of distinct keys.
But overall, for code that is heavy on sequences of querying and
updating a set of key/value pairs, hashtables will significantly
outperform maps.

Hashtables are not always the faster choice, though.  In particular,
maps are often more performant in situations where you want to take
advantage of maps as a persistent data-structure.  In particular, if
you create map `m'` by calling `Map.add` on some other map `m`, then
`m` and `m'` can be used independently, and in fact share most of
their underlying storage.  Thus, if you need to keep in memory at the
same time multiple different related collections of key/value pairs,
then a map is typically a much more efficient data structure to do it
with.

Here's a benchmark to demonstrates this.  In it, we create a list of
maps (or hashtables) that are built up by iteratively applying
updates, starting from an empty map.  In the hashtable implementation,
we do this by calling `Hashtbl.copy` to get the list entries.

```ocaml
(* file: map_vs_hash2.ml *)

open Core.Std
open Core_bench.Std

let create_maps ~num_keys ~iterations =
  let rec loop i map =
    if i <= 0 then []
    else
      let new_map =
        Map.change map (i mod num_keys) (fun current ->
          Some (1 + Option.value ~default:0 current))
      in
      new_map :: loop (i - 1) new_map
  in
  loop iterations Int.Map.empty

let create_tables ~num_keys ~iterations =
  let table = Int.Table.create ~size:num_keys () in
  let rec loop i =
    if i <= 0 then []
    else (
      Hashtbl.change table (i mod num_keys) (fun current ->
        Some (1 + Option.value ~default:0 current));
      let new_table = Hashtbl.copy table in
      new_table :: loop (i - 1)
    )
  in
  loop iterations

let tests ~num_keys ~iterations =
  let test name f = Bench.Test.create f ~name in
  [ test "map"   (fun () -> ignore (create_maps   ~num_keys ~iterations))
  ; test "table" (fun () -> ignore (create_tables ~num_keys ~iterations))
  ]

let () =
  tests ~num_keys:50 ~iterations:1000
  |> Bench.make_command
  |> Command.run
```

Unsurprisingly, maps perform far better than hashtables on this
benchmark, in this case by more than a factor of ten.  

```
$ ./map_vs_hash2.native -clear-columns name time speedup
Estimated testing time 20s (change using -quota SECS).
┌───────┬───────────┬─────────┐
│ Name  │ Time (ns) │ Speedup │
├───────┼───────────┼─────────┤
│ map   │   208_438 │   12.62 │
│ table │ 2_630_707 │    1.00 │
└───────┴───────────┴─────────┘
```

These numbers can be made more extreme by increasing the size of the
tables or the length of the list.

As you can see, the relative performance of trees and maps depends a
great deal on the details of how they're used, and so whether to
choose one data structure or the other will depend on the details of
the application.