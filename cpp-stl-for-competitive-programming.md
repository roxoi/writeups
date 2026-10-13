---
title: "C++ STL for Competitive Programming: From Basics to Contest-Ready Speed"
description: "A detailed, example-driven guide to the C++ STL for competitive programming - fast I/O template, vector, string, stringstream, set, map, unordered containers, priority_queue, bitset, sorting, binary search, numeric helpers, classic recipes, pbds, pitfalls and a complexity cheat sheet."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/cpp-stl.png"
tags: [Cpp, STL, Competitive-Programming, Data-Structures, Algorithms, Interview-Preparation]
keywords: ["C++ STL tutorial for competitive programming", "STL containers and algorithms explained", "vector set map priority_queue examples", "C++ string and stringstream methods", "Competitive programming C++ template and fast IO"]
---

# C++ STL for Competitive Programming

_A simple guide: what each STL tool does, when to use it, and code you can copy. Read one section, solve two or three problems with it, then move on._

![C++ STL](/images/cpp-stl.png)

## 1. Setup for every problem

```cpp
#include <bits/stdc++.h>
using namespace std;
using ll = long long;
using pll = pair<ll, ll>;
#define all(x) (x).begin(), (x).end()

int main() {
    ios_base::sync_with_stdio(false);
    cin.tie(nullptr);
    int t = 1;
    cin >> t;
    while (t--) {
        // solve one test case
    }
}
```

- Print `'\n'`, not `endl`. `endl` is slow.
- Use `long long` whenever a value or a sum can pass about 2 billion.

| Constraint on n   | Intended complexity | Typical techniques                            |
| ----------------- | ------------------- | --------------------------------------------- |
| n <= 10           | O(n!)               | brute-force permutations                      |
| n <= 20           | O(2^n \* n)         | bitmask DP, meet in the middle                |
| n <= 500          | O(n^3)              | Floyd-Warshall, interval DP                   |
| n <= 5,000        | O(n^2)              | simple DP, all pairs                          |
| n <= 2\*10^5      | O(n log n)          | sort, set, map, priority_queue, binary search |
| n <= 10^6 to 10^7 | O(n)                | prefix sums, two pointers, monotonic stack    |
| n <= 10^18        | O(log n) or O(1)    | math, fast exponentiation                     |

## 2. Containers

### 2.1 vector (use this first)

| Method                       | What it does                            | Cost |
| ---------------------------- | --------------------------------------- | ---- |
| `v.push_back(x)`             | add x at the end                        | O(1) |
| `v.pop_back()`               | remove the last element                 | O(1) |
| `v[i]`                       | element at index i                      | O(1) |
| `v.front()`, `v.back()`      | first and last element                  | O(1) |
| `v.size()`                   | number of elements                      | O(1) |
| `v.empty()`                  | true if there are no elements           | O(1) |
| `v.clear()`                  | remove all elements                     | O(n) |
| `v.resize(n)`                | change the size to n                    | O(n) |
| `v.assign(n, x)`             | replace the contents with n copies of x | O(n) |
| `v.insert(v.begin() + i, x)` | insert x before index i                 | O(n) |
| `v.erase(v.begin() + i)`     | remove the element at index i           | O(n) |

```cpp
vector<int> v = {5, 1, 4};
v.push_back(9);                                  // 5 1 4 9
v.insert(v.begin() + 1, 7);                      // 5 7 1 4 9
v.erase(v.begin());                              // 7 1 4 9
vector<vector<int>> g(n);                        // graph: g[u].push_back(w);
vector<vector<int>> dp(n, vector<int>(m, 0));    // n x m table of zeros
sort(all(v)); v.erase(unique(all(v)), v.end());  // sort and remove duplicates
```

Careful:

- `v.size()` is unsigned, so `v.size() - 1` is huge when `v` is empty. Write `(int)v.size() - 1`.
- Do not use `vector<bool>`. Use `vector<char>` instead.
- Do not `push_back` into a vector while looping over it with `for (auto x : v)`.

### 2.2 pair and array

```cpp
vector<pair<int, int>> a = {{3, 1}, {1, 2}, {3, 0}};
sort(all(a));                    // by first, then second: (1,2) (3,0) (3,1)
auto [x, y] = a[0];              // x = 1, y = 2
pair<int, int> p = {4, 5};       // p.first = 4, p.second = 5
array<int, 3> e = {7, 8, 9};     // fixed size; e[0], e[1], e[2]; sorts like a pair
```

### 2.3 string

| Method                     | What it does                                      | Cost      |
| -------------------------- | ------------------------------------------------- | --------- |
| `s.size()`                 | length                                            | O(1)      |
| `s += c`, `s.push_back(c)` | add a character at the end                        | O(1)      |
| `s.pop_back()`             | remove the last character                         | O(1)      |
| `s.substr(i, len)`         | copy of len characters starting at index i        | O(len)    |
| `s.find(t)`                | index of the first t, or `string::npos` if absent | O(n \* m) |
| `s.rfind(t)`               | index of the last t, or `string::npos`            | O(n \* m) |
| `s.insert(i, t)`           | insert t at index i                               | O(n)      |
| `s.erase(i, len)`          | remove len characters from index i                | O(n)      |
| `s.replace(i, len, t)`     | replace those characters with t                   | O(n)      |
| `reverse(all(s))`          | reverse the string                                | O(n)      |
| `to_string(x)`             | number to string                                  | O(len)    |
| `stoi(s)`, `stoll(s)`      | string to int, string to long long                | O(len)    |
| `getline(cin, s)`          | read a whole line                                 | O(len)    |

```cpp
string s = "hello world";
size_t p = s.find(' ');                       // 5
string w = s.substr(p + 1);                   // world
if (s.find('z') == string::npos) { /* z is not in s */ }
```

**stringstream** (`<sstream>`) splits a line into words or numbers:

| Call                           | What it does                                                       |
| ------------------------------ | ------------------------------------------------------------------ |
| `istringstream in(line);`      | a stream that reads from the string `line`                         |
| `in >> x`                      | read the next whitespace-separated value; false when none are left |
| `getline(in, tok, ',')`        | read up to the next comma (any character works as the separator)   |
| `ostringstream out;`           | a stream that builds a string                                      |
| `out << x`                     | write x into it                                                    |
| `out.str()`                    | get the string built so far                                        |
| `ss.str("")` then `ss.clear()` | empty a stream so you can reuse it (do both)                       |

```cpp
string line; getline(cin, line);
istringstream in(line);
vector<int> nums;
for (int x; in >> x;) nums.push_back(x);        // every integer on one line

istringstream csv(line); string tok;
while (getline(csv, tok, ',')) { /* each comma-separated part */ }
```

### 2.4 stack, queue, deque

Every operation below is O(1).

| Container        | Methods                                                                                                         | Use it for                              |
| ---------------- | --------------------------------------------------------------------------------------------------------------- | --------------------------------------- |
| `stack<int> st;` | `push(x)`, `pop()`, `top()`, `empty()`, `size()`                                                                | matching brackets, DFS, monotonic stack |
| `queue<int> q;`  | `push(x)`, `pop()`, `front()`, `back()`, `empty()`, `size()`                                                    | BFS                                     |
| `deque<int> dq;` | `push_back(x)`, `push_front(x)`, `pop_back()`, `pop_front()`, `front()`, `back()`, `dq[i]`, `empty()`, `size()` | sliding window maximum, 0-1 BFS         |

```cpp
// BFS: shortest distance from s in an unweighted graph
queue<int> q; vector<int> dist(n, -1);
dist[s] = 0; q.push(s);
while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : g[u]) if (dist[v] == -1) { dist[v] = dist[u] + 1; q.push(v); }
}

// maximum of every window of size k (deque holds indices, values decreasing)
deque<int> dq; vector<int> best;
for (int i = 0; i < n; i++) {
    while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();
    dq.push_back(i);
    if (dq.front() <= i - k) dq.pop_front();
    if (i >= k - 1) best.push_back(a[dq.front()]);
}
```

Careful: `pop()` returns nothing. Read `top()` or `front()` first, then pop.

### 2.5 priority_queue (heap)

| Method                    | What it does                             | Cost     |
| ------------------------- | ---------------------------------------- | -------- |
| `pq.push(x)`              | insert x                                 | O(log n) |
| `pq.top()`                | the best element (largest in a max-heap) | O(1)     |
| `pq.pop()`                | remove the best element                  | O(log n) |
| `pq.empty()`, `pq.size()` | state of the heap                        | O(1)     |

```cpp
priority_queue<int> mx;                                    // max-heap: top() is the largest
priority_queue<int, vector<int>, greater<int>> mn;         // min-heap: top() is the smallest

// Dijkstra: g[u] holds pairs (neighbour, weight)
vector<ll> dist(n, LLONG_MAX);
priority_queue<pll, vector<pll>, greater<pll>> pq;         // (distance, node), smallest first
dist[s] = 0; pq.push({0, s});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;                             // old entry, skip it
    for (auto [v, w] : g[u])
        if (dist[u] + w < dist[v]) { dist[v] = dist[u] + w; pq.push({dist[v], v}); }
}
```

Careful: there is no decrease-key and no `clear()`. To update a value, push the new one and skip stale entries when they come out, as in the code above.

### 2.6 set and multiset (sorted, with neighbours)

| Method                               | What it does                                     | Cost             |
| ------------------------------------ | ------------------------------------------------ | ---------------- |
| `s.insert(x)`                        | add x (a set ignores duplicates)                 | O(log n)         |
| `s.erase(x)`                         | remove value x (in a multiset: **all** copies)   | O(log n)         |
| `s.erase(it)`                        | remove the element at an iterator (one copy)     | O(1)             |
| `s.find(x)`                          | iterator to x, or `s.end()`                      | O(log n)         |
| `s.count(x)`                         | number of copies of x                            | O(log n)         |
| `s.contains(x)`                      | true if x is present (C++20)                     | O(log n)         |
| `s.lower_bound(x)`                   | iterator to the first element >= x, or `s.end()` | O(log n)         |
| `s.upper_bound(x)`                   | iterator to the first element > x, or `s.end()`  | O(log n)         |
| `*s.begin()`, `*s.rbegin()`          | smallest and largest element                     | O(1)             |
| `s.size()`, `s.empty()`, `s.clear()` | size, emptiness, remove everything               | O(1), O(1), O(n) |

```cpp
set<int> s = {10, 20, 30};
auto it = s.lower_bound(15);                  // points to 20 (first element >= 15)
auto jt = s.upper_bound(20);                  // points to 30 (first element > 20)
auto pr = s.lower_bound(20);                  // largest element smaller than 20:
if (pr != s.begin()) { --pr; cout << *pr; }   // prints 10

multiset<int> ms = {1, 1, 2};
ms.erase(ms.find(1));                         // removes ONE copy. ms.erase(1) would remove both.
```

Careful: call `s.lower_bound(x)`, not `lower_bound(all(s), x)` (that one is O(n) on a set). A set has no index and no rank: `distance(s.begin(), it)` is O(n).

### 2.7 map

| Method                                  | What it does                                           | Cost             |
| --------------------------------------- | ------------------------------------------------------ | ---------------- |
| `m[k]`                                  | value for key k; **creates** k with value 0 if missing | O(log n)         |
| `m.at(k)`                               | value for key k; error if missing                      | O(log n)         |
| `m.insert({k, v})`                      | add the pair only if k is absent                       | O(log n)         |
| `m.erase(k)`, `m.erase(it)`             | remove a key, or remove at an iterator                 | O(log n)         |
| `m.find(k)`                             | iterator to the pair, or `m.end()`                     | O(log n)         |
| `m.count(k)`                            | 1 if the key exists, else 0                            | O(log n)         |
| `m.contains(k)`                         | true if the key exists (C++20)                         | O(log n)         |
| `m.lower_bound(k)`, `m.upper_bound(k)`  | first pair with key >= k, or > k                       | O(log n)         |
| `m.begin()->first`, `m.rbegin()->first` | smallest and largest key                               | O(1)             |
| `m.size()`, `m.empty()`, `m.clear()`    | size, emptiness, remove everything                     | O(1), O(1), O(n) |

```cpp
map<int, int> freq;
for (int x : a) freq[x]++;                       // count occurrences
for (auto& [value, cnt] : freq) { /* visits values in sorted order */ }
if (freq.count(7)) { /* 7 exists, and no entry was created */ }
```

Careful: `m[k]` inserts the key when it is missing. To only check, use `count` or `find`.

### 2.8 unordered_map and unordered_set

Same methods as `map` and `set` (`[]`, `insert`, `erase`, `find`, `count`), but average O(1) instead of O(log n). Elements are not sorted and there is no `lower_bound`.

On Codeforces, use this hash, because the default one can be attacked into O(n) per operation:

```cpp
struct custom_hash {
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t R = chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + R);
    }
};
unordered_map<ll, int, custom_hash> mp;
```

Not sure which to pick? Use `map`. A `pair` cannot be a key of an `unordered_map` without writing a hash for it.

### 2.9 bitset (fast yes/no tables)

| Call                                                 | What it does                                          |
| ---------------------------------------------------- | ----------------------------------------------------- |
| `b.set(i)`, `b.reset(i)`, `b.flip(i)`                | turn bit i on, off, or toggle it                      |
| `b[i]`, `b.test(i)`                                  | read bit i                                            |
| `b.count()`                                          | number of bits that are on                            |
| `b1 & b2`, `b1 \| b2`, `b1 ^ b2`, `b << k`, `b >> k` | bit operations on all bits at once (about N/64 steps) |

```cpp
bitset<100001> dp;                           // subset sum: which totals are reachable?
dp[0] = 1;
for (int w : weights) dp |= dp << w;         // dp[s] == 1 means total s can be made
```

## 3. Algorithms

All of these take a range. `all(v)` means `v.begin(), v.end()`.

| Call                                             | What it does                                                                       | Cost         |
| ------------------------------------------------ | ---------------------------------------------------------------------------------- | ------------ |
| `sort(all(v))`                                   | sort ascending                                                                     | O(n log n)   |
| `sort(all(v), greater<int>())`                   | sort descending                                                                    | O(n log n)   |
| `sort(all(v), cmp)`                              | sort with your own rule                                                            | O(n log n)   |
| `stable_sort(all(v))`                            | sort, equal elements keep their order                                              | O(n log n)   |
| `reverse(all(v))`                                | reverse                                                                            | O(n)         |
| `unique(all(v))`                                 | remove neighbouring duplicates, returns the new end (use after sort, with `erase`) | O(n)         |
| `lower_bound(all(v), x)`                         | iterator to the first element >= x (sorted vector)                                 | O(log n)     |
| `upper_bound(all(v), x)`                         | iterator to the first element > x                                                  | O(log n)     |
| `binary_search(all(v), x)`                       | true if x exists                                                                   | O(log n)     |
| `min_element(all(v))`, `max_element(all(v))`     | iterator to the smallest or largest; write `*max_element(all(v))` for the value    | O(n)         |
| `accumulate(all(v), 0LL)`                        | sum of all elements                                                                | O(n)         |
| `partial_sum(all(a), pre.begin() + 1)`           | prefix sums (`pre` has n + 1 entries)                                              | O(n)         |
| `iota(all(v), 0)`                                | fill with 0, 1, 2, ...                                                             | O(n)         |
| `fill(all(v), x)`                                | set every element to x                                                             | O(n)         |
| `count(all(v), x)`                               | how many elements equal x                                                          | O(n)         |
| `count_if(all(v), pred)`                         | how many elements satisfy pred                                                     | O(n)         |
| `find(all(v), x)`                                | iterator to the first x, or `v.end()`                                              | O(n)         |
| `next_permutation(all(v))`                       | next permutation in order; false after the last                                    | O(n)         |
| `nth_element(v.begin(), v.begin() + k, v.end())` | puts the k-th smallest at index k                                                  | O(n) average |
| `min(a, b)`, `max(a, b)`                         | both values must have the same type; `max({a, b, c})` for three                    | O(1)         |
| `gcd(a, b)`, `lcm(a, b)`                         | greatest common divisor, least common multiple                                     | O(log)       |
| `__builtin_popcountll(x)`                        | number of 1 bits in a 64-bit number                                                | O(1)         |

```cpp
// custom sort: second value descending, ties by first value ascending
sort(all(a), [](const pair<int, int>& p, const pair<int, int>& q) {
    if (p.second != q.second) return p.second > q.second;
    return p.first < q.first;
});

// sort indices by value, without moving the values
vector<int> idx(n); iota(all(idx), 0);
sort(all(idx), [&](int i, int j) { return a[i] < a[j]; });

int i = lower_bound(all(v), x) - v.begin();                    // index of the first element >= x
int cnt = upper_bound(all(v), x) - lower_bound(all(v), x);     // how many times x appears

vector<ll> pre(n + 1, 0);
for (int i = 0; i < n; i++) pre[i + 1] = pre[i] + a[i];       // sum of a[l..r] = pre[r + 1] - pre[l]
```

Two rules: a comparator must use `<`, never `<=` (that crashes `sort`). And write `accumulate(all(v), 0LL)`, because plain `0` makes the sum an `int`.

## 4. Patterns you can copy

### 4.1 Binary search on the answer

Use it for "minimise the maximum", "maximise the minimum", "smallest time such that it all fits". It needs `ok(x)` to go false, false, ..., true, true.

```cpp
// smallest x with ok(x) == true
ll lo = 0, hi = 1e18;
while (lo < hi) {
    ll mid = lo + (hi - lo) / 2;
    if (ok(mid)) hi = mid; else lo = mid + 1;
}
// answer is lo

// largest x with ok(x) == true: round mid up
while (lo < hi) {
    ll mid = lo + (hi - lo + 1) / 2;
    if (ok(mid)) lo = mid; else hi = mid - 1;
}
```

### 4.2 Coordinate compression

Turns big values into small ranks 0, 1, 2, ... so you can use them as array indices.

```cpp
vector<int> vals = a;
sort(all(vals));
vals.erase(unique(all(vals)), vals.end());
for (int& x : a) x = lower_bound(all(vals), x) - vals.begin();
```

### 4.3 Longest increasing subsequence, O(n log n)

```cpp
vector<int> tails;
for (int x : a) {
    auto it = lower_bound(all(tails), x);        // use upper_bound if equal values are allowed
    if (it == tails.end()) tails.push_back(x); else *it = x;
}
int lis = tails.size();
```

### 4.4 Sliding window with a multiset

Longest subarray where `max - min <= k`:

```cpp
multiset<int> ms; int best = 0;
for (int l = 0, r = 0; r < n; r++) {
    ms.insert(a[r]);
    while (*ms.rbegin() - *ms.begin() > k) ms.erase(ms.find(a[l++]));
    best = max(best, r - l + 1);
}
```

### 4.5 Next greater element (monotonic stack)

```cpp
vector<int> nxt(n, -1), st;                      // nxt[i] = index of the next bigger element, or -1
for (int i = 0; i < n; i++) {
    while (!st.empty() && a[st.back()] < a[i]) { nxt[st.back()] = i; st.pop_back(); }
    st.push_back(i);
}
```

## 5. Mistakes that lose contests

1. Overflow: with `int a, b`, write `(ll)a * b`. Use `1LL << k`, and `accumulate(all(v), 0LL)`.
2. `v.size() - 1` is huge when `v` is empty. Write `(int)v.size() - 1`.
3. A comparator with `<=` crashes `sort`. Use `<`.
4. `m[k]` creates the key. To only check, use `m.count(k)`.
5. `ms.erase(x)` removes every copy. `ms.erase(ms.find(x))` removes one.
6. `lower_bound(all(s), x)` on a set is slow. Write `s.lower_bound(x)`.
7. Do not `push_back` into a vector while looping over it with `for (auto x : v)`.
8. `endl` is slow. After `sync_with_stdio(false)`, never mix `scanf` and `cin`.
9. `x % m` can be negative. Use `((x % m) + m) % m`.
10. `max(a, b)` fails when one is `int` and the other `long long`. Write `max<ll>(a, b)`.

## 6. Practice (LeetCode)

| Topic                  | Problems                                                                                                 |
| ---------------------- | -------------------------------------------------------------------------------------------------------- |
| priority_queue         | 295 Find Median from Data Stream, 347 Top K Frequent Elements                                            |
| deque, monotonic stack | 239 Sliding Window Maximum                                                                               |
| set, multiset          | 218 The Skyline Problem, 1438 Longest Continuous Subarray With Absolute Diff Less Than or Equal to Limit |
| map                    | 352 Data Stream as Disjoint Intervals                                                                    |
| Dijkstra               | 743 Network Delay Time                                                                                   |
| LIS, binary search     | 300 Longest Increasing Subsequence                                                                       |
