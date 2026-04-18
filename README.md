**Key ordering requirements of associative containers are poorly specified** 

[[associative.reqmts.general]](https://wg21.link/associative.reqmts.general) mandates that
`Compare` induce a strict weak ordering (SWO) on elements of `Key`. A strict reading of
this requirement without further qualifications implies, for instance, that using a container such
as `std::set<double>` is UB regardless of what elements go into it (because `std::less<double>` is
_not_ a SWO on all values of `double` due to NaN). Assuming that the intent of the
standard is to allow for the use of `std::set<double>` and other containers where `Compare`
does not induce a full SWO on all possible values of `Key`, there are (at least) two possible
relaxations of the original requirement:

1. Let `K` be the set of keys in an associative container `a` at a given point in time:
`Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion into `a` (either successful or not).
2. `Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion (either successful or not) _or_ a lookup operation.

Now, consider the following code:

```cpp
// (A)
std::set<double> s = {0.0, 1.0, 2.0, 3.0};
auto nan = std::numeric_limits<double>::quiet_NaN();
return s.upper_bound(nan) == s.end();
```

Under interpretation 1, (A) is correct and returns `true`, whereas under interpretation 2
the code is UB because `std::less<double>` does not induce a SWO on {0.0, 1.0, 2.0, 3.0, NaN}.
Currently, libstdc++, Microsoft's C++ Standard Library, and libc++ v21 or lower behave as if
interpretation 1 is in effect. [Starting with version 22](https://github.com/llvm/llvm-project/pull/161366),
libc++ goes with interpretation 2 and assumes (A) is UB (and the code in fact returns `false`),
see https://godbolt.org/z/789qr74es.

Note, however, that the following pieces of code are _not_ UB:

```cpp
// (B) Global binary search funtions
std::vector<double> v = {0.0, 3.0, 2.0, 1.0};
std::sort(v.begin(), v.end());
return std::upper_bound(v.begin(), v.end(), nan) == v.end(); // returns true
```
```cpp
// (C) Heterogeneous lookup
std::set<double, std::less<>> s = {0.0, 1.0, 2.0, 3.0};
return s.upper_bound(std::cref(nan)) == s.end(); // returns true
```
because `upper_bound` in (B) and (C) only requires that the passed key
_partition_ the range where lookup happens, and ¬( · < NaN) partitions
the range `R` = [0.0, 1.0, 2.0, 3.0] into `R` and [] (see
[[upper.bound]](https://wg21.link/upper.bound) and
[[associative.reqmts.genera]/167](https://wg21.link/associative.reqmts.general#167)).
The fact that (C) is well defined is an argument against interpretation 2, given that
(C) can be seen as a mere syntactic variation of (A).

The status quo (`Compare` must induce a full SWO on all possible values of `Key`)
is unacceptably strict and hardly the intent of the standard. As reasonable interpretation
is left open, we're beginning to see divergences between C++ standard library implementations.

**Arguments in favor of partition-based lookup semantics**

We posit that interpretation 1, supplemented with partition-based requirements for
non-heterogeneous lookup:

_Let `K` be the set of keys in an associative container `a` at a given point in time:
`Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion into `a` (either successful or not). Non-heterogeneous lookup for `k` only requires
that `k` partition the range [`a.begin()`, `a.end()`)._

is the most useful and consistent interpretation. Some arguments in favor of this thesis:

* Partition-based lookup semantics is maximally aligned and consistent with current
requirements for global lookup algorithms and associative container heterogeneus lookup.
In particular, it makes (A) and (C) equivalent, which is a reasonable assumption any
non-expert user could make.
* In the [N1313](https://wg21.link/n1313) foundational paper where partition-based semantics
was introduced, David Abrahams states that _"[s]trict weak ordering is a great concept for
sorting, but maybe it's not appropriate for searching"_.
In [N3465](https://wg21.link/n3465), an early version of the paper upon which heterogeneous
lookup for associative containers was defined, the wording for non-heterogeneous lookup
was indeed partition-based in accordance with Abrahams's conceptual setup (though this
exact wording didn't make it into the standard as is, for reasons unknown to the author).
* For a unique associative container `a`, if `Compare` induces a strict _partial_ order
on all its possible key values, then partition-based lookup is well defined everywhere.
Formally, if `Compare` is a SPO and a set `K` of keys is totally ordered wrt `Compare`
(i.e., `K` is a [_chain_](https://en.wikipedia.org/wiki/Total_order#Chains), which
the range [`a.begin()`, `a.end()`) always satisfies), then `K` is partitioned by any
arbitrary `k` (proof trivial).

**Proposed resolution:**

Wording is relative to [N5032](https://wg21.link/n5032):

* [associative.reqmts.general]/2: Each associative container is parameterized on `Key` and
an ordering relation `Compare` that induces a strict weak ordering ([alg.sorting]) on
<del>elements of Key</del><ins>the values of `Key` in the container at any point in time</ins>.
* [associative.reqmts.general]/7.8: <del>-- `a_tran` denotes a value of type `X` or `const X
when the _qualified-id_ `X​::​key_compare​::​is_transparent` is valid and denotes a type
([temp.deduct]),</del>
* [associative.reqmts.general]/7.17: `t` denotes a value of type `X​::​value_type`, <ins> and</ins>
* [associative.reqmts.general]/7.18: <del>`k` denotes a value of type X​::​key_type, and</del>
* [associative.reqmts.general]/7.23.2: `kx` is not convertible to either `iterator` or `const_iterator`; <del>and<del>
* Insert before [associative.reqmts.general]/7.24: <ins>-- if _qualified-id_
`X​::​key_compare​::​is_transparent` is not valid or does not denote a type ([temp.deduct]), then
`kl`, `ku`, `ke` and `kx` are of type `X​::​key_type`;
* In [associative.reqmts.general], remove the clauses for `a.extract(k)`, `a.erase(k)`,
`b.find(k)`, `b.count(k)`, `b.contains(k)`, `b.lower_bound(k)`, `b.upper_bound(k)`,
`b.equal_range(k)`.
* In [associative.reqmts.general], do the following replacements:
  * <del>`a_tran`</del><ins>`a`</ins> everywhere in the clause for `a_tran.extract(kx)`,
  * <del>`a_tran`</del><ins>`a`</ins> everywhere in the clause for `a_tran.erase(kx)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.find(ke)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.count(ke)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.contains(ke)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.lower_bound(kl)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.upper_bound(ku)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.equal_range(ke)`.
* [associative.reqmts.general]/48 (`a_uniq.emplace(args)`):
_Preconditions_: `value_type` is _Cpp17EmplaceConstructible_ into `X` from `args`.
<ins>If `t` is a `value_type` object constructed with `std​::​forward<Args>(args)...`,  `Compare`
induces a strict weak ordering on the set of values comprising the key of `t` and all the keys
in `a_uniq`.</ins>
* [associative.reqmts.general]/53 (`a_eq.emplace(args)`):
_Preconditions_: `value_type` is _Cpp17EmplaceConstructible_ into `X` from `args`.
<ins>If `t` is a `value_type` object constructed with `std​::​forward<Args>(args)...`,  `Compare`
induces a strict weak ordering on the set of values comprising the key of `t` and all the keys
in `a_eq`.</ins>
* [associative.reqmts.general]/62 (`a_uniq.insert(t)`):
_Preconditions_: If `t` is a non-const rvalue, `value_type` is _Cpp17MoveInsertable_ into `X`;
otherwise, `value_type` is _Cpp17CopyInsertable_ into `X`.
<ins>`Compare` induces a strict weak ordering on the set of values comprising the key of `t` and all the keys
in `a_uniq`.</ins>
* [associative.reqmts.general]/67 (`a_eq.insert(t)`):
_Preconditions_: If `t` is a non-const rvalue, `value_type` is _Cpp17MoveInsertable_ into `X`;
otherwise, `value_type` is _Cpp17CopyInsertable_ into `X`.
<ins>`Compare` induces a strict weak ordering on the set of values comprising the key of `t` and all the keys
in `a_eq`.</ins>
* [associative.reqmts.general]/71 (`a.insert(p, t)`):
_Preconditions_: If `t` is a non-const rvalue, `value_type` is _Cpp17MoveInsertable_ into `X`;
otherwise, `value_type` is _Cpp17CopyInsertable_ into `X`.
<ins>`Compare` induces a strict weak ordering on the set of values comprising the key of `t` and all the keys
in `a`.</ins>
* [associative.reqmts.general]/76 (`a.insert(i, j)`):
_Preconditions_: `value_type` is _Cpp17EmplaceConstructible_ into `X` from `*i`.
Neither `i` nor `j` are iterators into `a`.
<ins>`Compare` induces a strict weak ordering on the set of values comprising all the keys in `[i, j)` and all the keys
in `a`.</ins>
* [associative.reqmts.general]/80 (`a.insert_range(rng)`):
_Preconditions_: `value_type` is _Cpp17EmplaceConstructible_ into `X` from `*ranges​::​begin(rg)`.
`rg` and `a` do not overlap.
<ins>`Compare` induces a strict weak ordering on the set of values comprising all the keys in `rg` and all the keys
in `a`.</ins>
* [associative.reqmts.general]/85 (`a_uniq.insert(nh)`):
_Preconditions_: `nh` is empty or `a_uniq.get_allocator() == nh.get_allocator()` is `true`.
<ins>If `nh` is not empty, `Compare` induces a strict weak ordering on the set of values
comprising the key of `nh` and all the keys in `a_uniq`.</ins>
* [associative.reqmts.general]/90 (`a_eq.insert(nh)`):
_Preconditions_: `nh` is empty or `a_eq.get_allocator() == nh.get_allocator()` is `true`.
<ins>If `nh` is not empty, `Compare` induces a strict weak ordering on the set of values
comprising the key of `nh` and all the keys in `a_eq`.</ins>
* [associative.reqmts.general]/95 (`a.insert(p, nh)`):
_Preconditions_: `nh` is empty or `a.get_allocator() == nh.get_allocator()` is `true`.
<ins>If `nh` is not empty, `Compare` induces a strict weak ordering on the set of values
comprising the key of `nh` and all the keys in `a`.</ins>
* [associative.reqmts.general]/113 (`a.merge(a2)`):
_Preconditions_: `a.get_allocator() == a2.get_allocator()` is `true`.
<ins>`Compare` induces a strict weark ordering on the set of values comprising all
the keys in `a` and all the keys in `a2`.</ins>


