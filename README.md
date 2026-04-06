**Key ordering requirements of associative containers are poorly specified** 

[[associative.reqmts]](https://eel.is/c++draft/associative.reqmts.general) mandates that
`Compare` induce a strict weak ordering (SWO) on elements of `Key`. A strict reading of
this requirement without further qualifications implies that using a container such
as `std::set<double>` is UB regardless of what elements go into it (because `std::less<double>` is
_not_ a SWO on all values of `double` due to NaN). Assuming that the intent of the
standard is to allow for the use of `std::set<double>` and other containers where `Compare`
does not induce a full SWO on all possible values of `Key`, there are (at least) three possible
relaxations of the original requirement:

1. `Compare` induces a SWO on all keys _within_ the container at a given point in time. Let's call the
(multi)set of such keys `K`.
3. `Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion (either successful or not).
4. `Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion (either successful or not) _or_ a lookup operation.

Under 2 and 3, the following code is UB:

```cpp
// (A)
std::set<double> s = {0.0, 1.0, 2.0, 3.0};
auto nan = std::numeric_limits<double>::quiet_NaN();
s.upper_bound(nan);
```

because `std::less<double>` does not induce a SWO on {0.0, 1.0, 2.0, 3.0, NaN}. Note, however,
that the following pieces of code are _not_ UB:

```cpp
// (B)
std::vector<double> v = {0.0, 3.0, 2.0, 1.0};
std::sort(v.begin(), b.end());
std::upper_bound(v.begin(), v.end(), nan);
```
```cpp
// (C)
std::set<double, std::less<>> s = {0.0, 1.0, 2.0, 3.0};
s.upper_bound(std::cref(nan));
```
because `upper_bound` in (B) and (C) only requires that the passed key
_partition_ the range where lookup happens, and ¬( · < NaN) partitions
the range `R` = [0.0, 1.0, 2.0, 3.0] into `R` and [] (see
[[upper.bound]](https://eel.is/c++draft/upper.bound) and
[[associative.reqmts.genera]/167](https://eel.is/c++draft/associative.reqmts.general#167)).
The fact that (C) is well defined is an argument against interpretations 2 or 3, given that
(C) can be seen as a mere syntactic variation of (A).

Currently, libstdc++, Microsoft's C++ Standard Library and libc++ v21 or lower
behave as if interpretation 1 is in effect. [Starting with version 22](https://github.com/llvm/llvm-project/pull/161366),
libc++ goes with 2 or 3, and assumes (A) is UB.

The status quo (`Compare` must induce a full SWO on all possible values of `Key`)
is unacceptably strict and hardly the intent of the standard. As reasonable interpretation
is left open, we're beginning to see divergences between C++ standard library implementations.

**Proposed resolution:**

Relax/clarify the requirements along approach 1,
which is maximally aligned with current requirements for global lookup algorithms
and associative container heterogeneus lookup, and define insertion and
lookup in terms of partitioning. Wording is relative to https://eel.is/c++draft
as of Apr 07, 2026:

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
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.upper_bound(ku)`.
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.lower_bounds(kl)`,
  * <del>`a_tran`</del><ins>`b`</ins> everywhere in the clause for `a_tran.equal_range(ke)`,
* CONTINUE WITH INSERTION
