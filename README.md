**Key ordering requirements of associative containers are poorly specified** 

[[associative.reqmts]](https://eel.is/c++draft/associative.reqmts.general) mandates that
`Compare` induce a strict weak ordering (SWO) on elements of `Key`. A strict reading of
this requirement without further qualifications implies that using a container such
as `std::set<double>` is UB regardless of what elements go into it (because `std::less<double>` is
_not_ a SWO on all elements of `double` due to NaN). Assuming that the intent of the
standard is to allow for the use of `std::set<double>` and other containers where `Compare`
does not induce a full SWO on all possible values of `Key`, there are (at least) three possible
relaxations of the original requirement:

1. `Compare` induces a SWO on all keys _within_ the container at a given point in time. Let's call the set of
such keys `K`.
2. `Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion (either successful or not).
3. `Compare` induces a SWO on `K` and also on {`k`} ∪ `K` for any `k` involved in an attempted
insertion (either successful or not) _or_ a lookup operation.

Under 2 or 3, the following code is UB:

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
std::vector<double> v = {0., 3., 2., 1.};
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
