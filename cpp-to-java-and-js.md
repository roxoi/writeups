---
title: "C++ to Java and JavaScript: Competitive Programming Tutorial"
description: "Move from C++ STL to Java and JavaScript for competitive programming: fast I/O, number traps, heaps, TreeMap, DSU, and the mistakes that cause WA and TLE."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com"]
thumbnail: "/images/cpp-to-java-and-js.png"
tags: [Competitive-Programming, Cpp, Java, JavaScript, STL, DSA, Codeforces]
keywords: ["C++ to Java for competitive programming", "C++ STL equivalent in Java", "JavaScript competitive programming", "Java fast input BufferedReader", "priority_queue in Java and JavaScript", "TreeMap vs map C++"]
---

# C++ to Java and JavaScript: Competitive Programming Tutorial

![C++ to Java and JavaScript](/images/cpp-to-java-and-js.png)

## Start here: how this tutorial works

This tutorial assumes you know the C++ STL (`vector`, `map`, `set`, `priority_queue`). Each chapter first reminds you of the C++ way. Then it shows the Java and JavaScript equivalent, a small example, and the mistakes that cause WA or TLE.

Three things to know first:

- **Java** is the closest to C++ for competitive programming. `TreeMap`, `TreeSet` and `PriorityQueue` are all built in.
- **JavaScript** has no built-in heap, no ordered set or map, and no 64-bit integer. Where you must write something yourself, this tutorial says so clearly.
- In C++, `bits/stdc++.h` gave you everything. In Java you must `import` each class (`java.util.*`, `java.io.*`).

| Topic             | C++                 | Java                             | JavaScript           |
| ----------------- | ------------------- | -------------------------------- | -------------------- |
| Heap              | `priority_queue`    | `PriorityQueue`                  | write it yourself    |
| Ordered set / map | `set`, `map`        | `TreeSet`, `TreeMap`             | write it yourself    |
| 64-bit integer    | `long long`         | `long`                           | `BigInt` (slow)      |
| Fast input        | `cin` with sync off | `BufferedReader`                 | `readFileSync`       |
| Deep recursion    | works fine          | run in a thread with a big stack | write it iteratively |

**Roadmap:** Fast I/O → numbers → arrays and sorting → strings → stack and queue → heap → ordered set and map → hash → graph and DSU → utilities → common mistakes → practice plan.

## 1. First program and fast I/O

Reading input fast is the first skill you need. In C++, `ios::sync_with_stdio(false)` was enough. In Java, `Scanner` is slow, so use `BufferedReader` and `StringTokenizer`. In JavaScript, read the whole input once and split it into tokens.

**Problem:** You get `n` numbers. Print their sum.

**C++ (the one you know)**

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    ios::sync_with_stdio(false);
    cin.tie(nullptr);
    int n; cin >> n;
    long long sum = 0;
    for (int i = 0; i < n; i++) { long long x; cin >> x; sum += x; }
    cout << sum << '\n';
}
```

**Java**

```java
import java.util.*;
import java.io.*;

public class Main {
    static BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
    static StringTokenizer st;

    static String next() throws IOException {
        while (st == null || !st.hasMoreTokens()) st = new StringTokenizer(br.readLine());
        return st.nextToken();
    }
    static int ni() throws IOException { return Integer.parseInt(next()); }
    static long nl() throws IOException { return Long.parseLong(next()); }

    public static void main(String[] args) throws IOException {
        int n = ni();
        long sum = 0;
        for (int i = 0; i < n; i++) sum += nl();
        StringBuilder sb = new StringBuilder();
        sb.append(sum).append("\n");
        System.out.print(sb);
    }
}
```

`ni()` reads an int and `nl()` reads a long. Copy this template into every problem.

**JavaScript (Node.js)**

```js
const data = require('fs').readFileSync(0, 'utf8').split(/\s+/).filter(Boolean);
let p = 0;
const next = () => data[p++];

const n = Number(next());
let sum = 0; // if the sum can pass 2^53, use BigInt
for (let i = 0; i < n; i++) sum += Number(next());
console.log(sum);
```

**Output rule:** In Java, collect the output in a `StringBuilder` and print once at the end. In JavaScript, push lines into an `out` array and call `console.log(out.join('\n'))` once at the end. Printing once per test case is slow in both.

**Practice:** Read `n` numbers, sort them, and print how many distinct values there are. Write it in all three languages.

## 2. Numbers and their traps

This chapter causes the most WA, because numbers behave differently in each language.

| C++         | Java                | JavaScript                                       |
| ----------- | ------------------- | ------------------------------------------------ |
| `int`       | `int` (32-bit)      | `Number` (double, exact up to 2^53 - 1)          |
| `long long` | `long`              | `BigInt` (`5n`), or Number if the value is small |
| `double`    | `double`            | `Number`                                         |
| `bool`      | `boolean`           | `boolean`                                        |
| `char`      | `char`              | string of length 1                               |
| `INT_MAX`   | `Integer.MAX_VALUE` | `2 ** 31 - 1`                                    |
| `LLONG_MAX` | `Long.MAX_VALUE`    | `Infinity` or `BigInt`                           |

### Trap 1: int \* int in Java

```java
int a = 1_000_000, b = 1_000_000;
long wrong = a * b;          // the multiply happens in int first, so it overflows
long right = (long) a * b;   // make it long first, then multiply
```

C++ does the same thing. People forget it in Java because `long wrong = a * b` looks correct.

### Trap 2: JavaScript loses precision after 2^53

```js
console.log(2 ** 53 + 1); // 9007199254740992, which is wrong

// (a * b) % m when a and b are around 1e9:
const mul = (a, b, m) => Number((BigInt(a) * BigInt(b)) % BigInt(m));
```

You cannot mix `BigInt` and `Number`: `5n + 1` throws an error. Convert both sides to BigInt. BigInt is slow, so be careful if you do more than 10^6 multiplications.

### Trap 3: Division and modulo

| Expression          | C++                 | Java                  | JavaScript                      |
| ------------------- | ------------------- | --------------------- | ------------------------------- |
| `7 / 2`             | 3                   | 3                     | 3.5, so use `Math.trunc(7 / 2)` |
| `-7 / 2`            | -3                  | -3                    | `Math.trunc(-7 / 2)` = -3       |
| `-7 % 5`            | -2                  | -2                    | -2                              |
| Always-positive mod | `((a % m) + m) % m` | `Math.floorMod(a, m)` | `((a % m) + m) % m`             |

### Trap 4: INF

In Java, `Integer.MAX_VALUE + 1` overflows to a negative number. A safe INF is `int INF = 1_000_000_000;`, because adding two of them is still below 2^31. For long, use `Long.MAX_VALUE / 2`. In JavaScript, `Infinity` is safe: `Infinity + 5` is still `Infinity`.

**Practice:** Compute `(10^9 * 10^9) % (10^9 + 7)` correctly in all three languages.

## 3. Arrays, lists and sorting

In C++, `vector` did everything. Java has three separate things: fixed-size `int[]`, resizable `ArrayList<Integer>`, and 2D `int[][]`. JavaScript has one `Array` that does all of this.

```cpp
vector<int> a(n, 0);
vector<vector<int>> g(n, vector<int>(m));
vector<int> v; v.push_back(5);
```

```java
int[] a = new int[n];                    // default value is 0
int[][] g = new int[n][m];
ArrayList<Integer> v = new ArrayList<>();
v.add(5);
```

```js
const a = new Array(n).fill(0); // or new Int32Array(n), which is faster
const g = Array.from({ length: n }, () => new Array(m).fill(0));
const v = [];
v.push(5);
```

| Task            | C++                           | Java                                  | JavaScript        |
| --------------- | ----------------------------- | ------------------------------------- | ----------------- |
| Size            | `v.size()`                    | `a.length` (array), `v.size()` (list) | `v.length`        |
| Add at end      | `push_back(x)`                | `add(x)`                              | `push(x)`         |
| Remove from end | `pop_back()`                  | `remove(v.size() - 1)`                | `pop()`           |
| Get / set       | `v[i]`                        | `get(i)` / `set(i, x)`                | `v[i]`            |
| Last element    | `v.back()`                    | `get(size() - 1)`                     | `v.at(-1)`        |
| Insert at i     | `insert(begin() + i, x)`      | `add(i, x)`                           | `splice(i, 0, x)` |
| Erase at i      | `erase(begin() + i)`          | `remove(i)`                           | `splice(i, 1)`    |
| Fill            | `fill(a, a + n, x)`           | `Arrays.fill(a, x)`                   | `a.fill(x)`       |
| Copy            | `auto b = a;`                 | `a.clone()`                           | `[...a]`          |
| Reverse         | `reverse(a.begin(), a.end())` | `Collections.reverse(list)`           | `a.reverse()`     |
| Print           | loop                          | `Arrays.toString(a)`                  | `a.join(' ')`     |

### Sorting

```cpp
sort(a.begin(), a.end());
sort(a.begin(), a.end(), greater<int>());
```

```java
Arrays.sort(a);                              // for int[]
Collections.sort(list);                      // for ArrayList
Integer[] b = ...;
Arrays.sort(b, Collections.reverseOrder());  // descending, works only on Integer[]
```

```js
a.sort((x, y) => x - y); // ascending
a.sort((x, y) => y - x); // descending
```

**Sort pairs by the first element, and by the second element when the first is equal:**

```cpp
sort(v.begin(), v.end(), [](auto &a, auto &b) {
    return a[0] != b[0] ? a[0] < b[0] : a[1] < b[1];
});
```

```java
int[][] arr = new int[n][2];
Arrays.sort(arr, (a, b) -> a[0] != b[0] ? Integer.compare(a[0], b[0]) : Integer.compare(a[1], b[1]));
```

```js
arr.sort((a, b) => a[0] - b[0] || a[1] - b[1]);
```

**Watch out:**

- JavaScript `sort()` without a comparator sorts as strings: `[10, 9, 1]` becomes `[1, 10, 9]`. Always pass a comparator.
- A Java comparator returns an `int` (negative, zero or positive), not a bool. Do not write `a - b`, because it can overflow. Write `Integer.compare(a, b)`.
- In Java, `Arrays.sort(int[])` can be attacked with anti-quicksort tests on Codeforces and give TLE. Shuffle first, or use `Integer[]`:

```java
static void shuffleSort(int[] a) {
    Random r = new Random();
    for (int i = a.length - 1; i > 0; i--) {
        int j = r.nextInt(i + 1);
        int t = a[i]; a[i] = a[j]; a[j] = t;
    }
    Arrays.sort(a);
}
```

- In an `ArrayList`, `list.remove(i)` removes by index. To remove a value, write `list.remove(Integer.valueOf(x))`.
- In JavaScript, `new Array(n).fill([])` makes every row the same array. Use `Array.from` instead.

**Practice:** Sort a list of intervals `[l, r]` by `l`, then merge the overlapping ones.

## 4. Strings

| Task          | C++                           | Java                                          | JavaScript                  |
| ------------- | ----------------------------- | --------------------------------------------- | --------------------------- |
| Length        | `s.size()`                    | `s.length()`                                  | `s.length`                  |
| Char at i     | `s[i]`                        | `s.charAt(i)`                                 | `s[i]`                      |
| Substring     | `s.substr(i, len)`            | `s.substring(i, j)` (j is exclusive)          | `s.slice(i, j)`             |
| Find          | `s.find(t)`                   | `s.indexOf(t)` (-1 if not found)              | `s.indexOf(t)`              |
| Equal         | `s == t`                      | `s.equals(t)`                                 | `s === t`                   |
| Compare       | `s < t`                       | `s.compareTo(t)`                              | `s < t`                     |
| String to int | `stoi(s)`                     | `Integer.parseInt(s)`                         | `Number(s)`                 |
| Int to string | `to_string(x)`                | `String.valueOf(x)`                           | `String(x)`                 |
| Char to digit | `c - '0'`                     | `c - '0'`                                     | `s.charCodeAt(i) - 48`      |
| Reverse       | `reverse(s.begin(), s.end())` | `new StringBuilder(s).reverse().toString()`   | `[...s].reverse().join('')` |
| Sort chars    | `sort(s.begin(), s.end())`    | `char[] c = s.toCharArray(); Arrays.sort(c);` | `[...s].sort().join('')`    |

**Strings are immutable in Java.** If you write `s += c` in a loop, it becomes O(n²). Use `StringBuilder`: `append`, `insert`, `reverse`, `setCharAt`, `deleteCharAt`, `length`, `toString`.

**Example: count each letter in a lowercase string**

```cpp
int cnt[26] = {};
for (char c : s) cnt[c - 'a']++;
```

```java
int[] cnt = new int[26];
for (char c : s.toCharArray()) cnt[c - 'a']++;
```

```js
const cnt = new Array(26).fill(0);
for (const c of s) cnt[c.charCodeAt(0) - 97]++; // the code of 'a' is 97
```

**Watch out:**

- In Java, never compare strings with `==`. Always use `.equals()`.
- In JavaScript, `'a' - 'b'` gives NaN. Use `charCodeAt` and `String.fromCharCode` for char arithmetic.
- In JavaScript, `s += c` in a loop is fine (the engine optimizes it). In Java it is not.

**Practice:** Check whether two strings are anagrams, in all three languages.

## 5. Stack, Queue and Deque

In C++, `stack`, `queue` and `deque` were separate containers. In Java, one class is enough for all three: `ArrayDeque`. In JavaScript, a plain array works, but `shift()` is O(n), so for a queue we use a head pointer.

| Task               | C++                          | Java (`ArrayDeque<Integer>`) | JavaScript                                          |
| ------------------ | ---------------------------- | ---------------------------- | --------------------------------------------------- |
| Stack push         | `st.push(x)`                 | `push(x)`                    | `st.push(x)`                                        |
| Stack pop          | `st.pop()` (returns nothing) | `pop()` (returns the value)  | `st.pop()` (returns the value)                      |
| Stack top          | `st.top()`                   | `peek()`                     | `st[st.length - 1]`                                 |
| Queue push         | `q.push(x)`                  | `offer(x)` (or `add(x)`)     | `q.push(x)`                                         |
| Queue pop          | `q.front(); q.pop();`        | `poll()` (null if empty)     | `q[head++]`                                         |
| Queue front        | `q.front()`                  | `peek()` (null if empty)     | `q[head]`                                           |
| Deque push_back    | `push_back(x)`               | `offerLast(x)`               | `push(x)`                                           |
| Deque push_front   | `push_front(x)`              | `offerFirst(x)`              | `unshift(x)` (slow)                                 |
| Deque pop_back     | `pop_back()`                 | `pollLast()`                 | `pop()`                                             |
| Deque pop_front    | `pop_front()`                | `pollFirst()`                | `shift()` (slow)                                    |
| Deque front / back | `front()` / `back()`         | `peekFirst()` / `peekLast()` | `a[0]` / `a.at(-1)`                                 |
| Is it empty?       | `empty()`                    | `isEmpty()`                  | check the length (or `head < q.length` for a queue) |

In Java, do not use the old `Stack` class or `LinkedList`. `ArrayDeque` is faster. `ArrayDeque` does not allow null values.

### `offer`, `poll` and `peek` in Java

Java queues have two versions of each operation. One version throws an exception when it fails. The other returns a special value instead. In competitive programming we mostly use the second version, because it does not crash on an empty queue.

| Action                | Throws an exception | Returns a special value                         |
| --------------------- | ------------------- | ----------------------------------------------- |
| Add                   | `add(x)`            | `offer(x)` returns `false` if the queue is full |
| Remove from the front | `remove()`          | `poll()` returns `null` if the queue is empty   |
| Look at the front     | `element()`         | `peek()` returns `null` if the queue is empty   |

`ArrayDeque` and `PriorityQueue` are never full, so `add(x)` and `offer(x)` do the same thing for them. Pick one and use it everywhere. This tutorial uses `offer`.

Because `poll()` returns `null` on an empty queue, check `isEmpty()` before you call it. Writing `int u = q.poll();` on an empty queue throws NullPointerException.

In JavaScript, if you need a real O(1) deque (for 0-1 BFS), make `new Array(2 * n + 1)` and move `head` and `tail` pointers from the middle.

**Example: shortest distance with BFS (unweighted graph)**

```cpp
vector<int> dist(n, -1);
queue<int> q;
dist[s] = 0; q.push(s);
while (!q.empty()) {
    int u = q.front(); q.pop();
    for (int v : g[u])
        if (dist[v] == -1) { dist[v] = dist[u] + 1; q.push(v); }
}
```

```java
int[] dist = new int[n];
Arrays.fill(dist, -1);
ArrayDeque<Integer> q = new ArrayDeque<>();
dist[s] = 0; q.offer(s);
while (!q.isEmpty()) {
    int u = q.poll();
    for (int v : g[u])
        if (dist[v] == -1) { dist[v] = dist[u] + 1; q.offer(v); }
}
```

```js
const dist = new Array(n).fill(-1);
const q = [s];
let head = 0;
dist[s] = 0;
while (head < q.length) {
  const u = q[head++];
  for (const v of g[u])
    if (dist[v] === -1) {
      dist[v] = dist[u] + 1;
      q.push(v);
    }
}
```

**Practice:** Find the shortest path in an `n x m` grid with BFS (`#` is a wall, `.` is free space).

## 6. Priority Queue (Heap)

In C++, `priority_queue` is a **max-heap** by default. In Java, `PriorityQueue` is a **min-heap** by default. They are opposite, so this is where most confusion happens. JavaScript has no built-in heap, so you must write one.

| Task         | C++                                              | Java                                              | JavaScript                  |
| ------------ | ------------------------------------------------ | ------------------------------------------------- | --------------------------- |
| Default      | max-heap                                         | min-heap                                          | write it yourself           |
| Min-heap     | `priority_queue<int, vector<int>, greater<int>>` | `new PriorityQueue<>()`                           | `new Heap()`                |
| Max-heap     | `priority_queue<int>`                            | `new PriorityQueue<>(Collections.reverseOrder())` | `new Heap((a, b) => b - a)` |
| Push         | `push(x)`                                        | `offer(x)` (or `add(x)`)                          | `push(x)`                   |
| Pop          | `pop()` (returns nothing)                        | `poll()` (returns the value)                      | `pop()` (returns the value) |
| Top          | `top()`                                          | `peek()`                                          | `peek()`                    |
| Size / empty | `size()` / `empty()`                             | `size()` / `isEmpty()`                            | `size()`                    |

For pairs or arrays, give a comparator. In Java, the easiest way is to put `int[]` in the heap:

```java
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> Integer.compare(a[0], b[0]));
pq.offer(new int[]{5, 2});   // {priority, node}
```

### A Heap class for JavaScript (copy and paste)

```js
class Heap {
  constructor(cmp = (a, b) => a - b) {
    this.h = [];
    this.cmp = cmp;
  }
  size() {
    return this.h.length;
  }
  peek() {
    return this.h[0];
  }
  push(x) {
    const h = this.h,
      c = this.cmp;
    h.push(x);
    let i = h.length - 1;
    while (i > 0) {
      const p = (i - 1) >> 1;
      if (c(h[i], h[p]) >= 0) break;
      [h[i], h[p]] = [h[p], h[i]];
      i = p;
    }
  }
  pop() {
    const h = this.h,
      c = this.cmp;
    const top = h[0],
      last = h.pop();
    if (h.length) {
      h[0] = last;
      let i = 0;
      const n = h.length;
      while (true) {
        const l = 2 * i + 1,
          r = l + 1;
        let m = i;
        if (l < n && c(h[l], h[m]) < 0) m = l;
        if (r < n && c(h[r], h[m]) < 0) m = r;
        if (m === i) break;
        [h[i], h[m]] = [h[m], h[i]];
        i = m;
      }
    }
    return top;
  }
}
// min-heap: new Heap()
// max-heap: new Heap((a, b) => b - a)
// Dijkstra: new Heap((a, b) => a[0] - b[0]) with elements [dist, node]
```

Comparator rule: if `cmp(a, b) < 0`, then `a` comes out first.

### Example: Dijkstra

`g[u]` holds `(v, w)` pairs. A heap has no decrease-key, so we skip old entries when they are popped (`if (d > dist[u]) continue;`).

```cpp
vector<long long> dist(n, LLONG_MAX);
priority_queue<pair<long long, int>, vector<pair<long long, int>>, greater<>> pq;
dist[s] = 0; pq.push({0, s});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;
    for (auto [v, w] : g[u])
        if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
}
```

```java
// g: List<int[]>[] g, each edge is int[]{v, w}
long[] dist = new long[n];
Arrays.fill(dist, Long.MAX_VALUE);
PriorityQueue<long[]> pq = new PriorityQueue<>((a, b) -> Long.compare(a[0], b[0]));
dist[s] = 0; pq.offer(new long[]{0, s});
while (!pq.isEmpty()) {
    long[] cur = pq.poll();
    long d = cur[0]; int u = (int) cur[1];
    if (d > dist[u]) continue;
    for (int[] e : g[u]) {
        int v = e[0]; int w = e[1];
        if (d + w < dist[v]) { dist[v] = d + w; pq.offer(new long[]{dist[v], v}); }
    }
}
```

```js
// g[u] = [[v, w], ...]
const dist = new Array(n).fill(Infinity);
const pq = new Heap((a, b) => a[0] - b[0]);
dist[s] = 0;
pq.push([0, s]);
while (pq.size()) {
  const [d, u] = pq.pop();
  if (d > dist[u]) continue;
  for (const [v, w] of g[u]) {
    if (d + w < dist[v]) {
      dist[v] = d + w;
      pq.push([dist[v], v]);
    }
  }
}
```

**Watch out:**

- In Java, `pq.remove(x)` is O(n), and printing a `PriorityQueue` does not show sorted order. To get sorted output, keep calling `poll()`.
- C++ `pop()` returns nothing (take `top()` first), but Java `poll()` and JavaScript `pop()` return the value.
- In Java, `poll()` and `peek()` return `null` on an empty heap. Check `isEmpty()` before you use them.
- In Dijkstra, `Long.MAX_VALUE + w` overflows. The code above is safe because `d` is always a valid distance that was just popped.

**Practice:** Merge K sorted arrays (keep `[value, array_index, element_index]` in a min-heap).

## 7. Ordered Set and Ordered Map

C++ `set` / `map` are `TreeSet` / `TreeMap` in Java. Both stay sorted, and every operation is O(log n). **JavaScript has no equivalent.**

### Set

| Task               | C++ `set<int>`           | Java `TreeSet<Integer>`      |
| ------------------ | ------------------------ | ---------------------------- |
| Insert             | `insert(x)`              | `add(x)`                     |
| Erase              | `erase(x)`               | `remove(x)`                  |
| Exists             | `count(x)`               | `contains(x)`                |
| First element >= x | `lower_bound(x)`         | `ceiling(x)`                 |
| First element > x  | `upper_bound(x)`         | `higher(x)`                  |
| Last element <= x  | `prev(upper_bound(x))`   | `floor(x)`                   |
| Last element < x   | `prev(lower_bound(x))`   | `lower(x)`                   |
| Min / max          | `*begin()` / `*rbegin()` | `first()` / `last()`         |
| Remove min / max   | `erase(begin())`         | `pollFirst()` / `pollLast()` |

In Java, `ceiling`, `floor`, `higher` and `lower` return **null** when nothing is found, and `first()` / `last()` throw an exception on an empty set. So store the result in an `Integer` and check for null:

```java
TreeSet<Integer> s = new TreeSet<>();
Integer c = s.ceiling(x);
if (c != null) { /* use c */ }
```

### Map

| Task          | C++ `map<int,int>`       | Java `TreeMap<Integer,Integer>`                                                       |
| ------------- | ------------------------ | ------------------------------------------------------------------------------------- |
| Put           | `m[k] = v`               | `put(k, v)`                                                                           |
| Get           | `m[k]`                   | `get(k)` / `getOrDefault(k, 0)`                                                       |
| Exists        | `count(k)`               | `containsKey(k)`                                                                      |
| Erase         | `erase(k)`               | `remove(k)`                                                                           |
| Min / max key | `m.begin()->first`       | `firstKey()` / `lastKey()`                                                            |
| lower_bound   | `m.lower_bound(k)`       | `ceilingKey(k)` / `ceilingEntry(k)`                                                   |
| Floor         | `prev(m.upper_bound(k))` | `floorKey(k)`                                                                         |
| Iterate       | `for (auto [k, v] : m)`  | `for (Map.Entry<Integer,Integer> e : m.entrySet())` with `e.getKey()`, `e.getValue()` |

### How to make a multiset

Java has no `multiset`. Build one with `TreeMap<value, count>`.

```cpp
multiset<int> ms;
ms.insert(x);
ms.erase(ms.find(x));   // removes only one copy. ms.erase(x) removes all of them!
```

```java
TreeMap<Integer, Integer> ms = new TreeMap<>();
ms.merge(x, 1, Integer::sum);          // insert

int c = ms.get(x);                     // erase one copy
if (c == 1) ms.remove(x); else ms.put(x, c - 1);

// min: ms.firstKey()   max: ms.lastKey()
```

### What to do in JavaScript

There are three ways:

1. **Sorted array + binary search.** Insert and erase are O(n) because of `splice`, but this works fine up to about n = 10^5.
2. **Coordinate compression + Fenwick tree.** Best for rank, count and k-th queries.
3. **Write your own treap or AVL tree.** The most flexible, but the code is long.

```js
// first index where a[i] >= x   (C++ lower_bound)
function lowerBound(a, x) {
  let lo = 0,
    hi = a.length;
  while (lo < hi) {
    const mid = (lo + hi) >> 1;
    if (a[mid] < x) lo = mid + 1;
    else hi = mid;
  }
  return lo;
}

const a = []; // works like a sorted set
a.splice(lowerBound(a, x), 0, x); // insert
const i = lowerBound(a, x); // ceiling(x) = a[i], if i < a.length
if (i < a.length && a[i] === x) a.splice(i, 1); // erase
```

```js
// Fenwick tree (1-indexed), n = compressed size
const bit = new Array(n + 1).fill(0);
const update = (i, d) => {
  for (; i <= n; i += i & -i) bit[i] += d;
};
const query = (i) => {
  let s = 0;
  for (; i > 0; i -= i & -i) s += bit[i];
  return s;
}; // prefix [1..i]
```

**Watch out:** In a Java `TreeSet`, `headSet(x).size()` is O(n), and in C++ `distance(s.begin(), it)` is also O(n). If you need rank or the k-th element, use a Fenwick tree (C++ also has pb_ds, Java does not).

**Practice:** For a stream of numbers, tell for each new number which earlier number is closest to it.

## 8. Hash Map and Hash Set

| Task             | C++ `unordered_map`     | Java `HashMap`                       | JavaScript `Map`                |
| ---------------- | ----------------------- | ------------------------------------ | ------------------------------- |
| Put              | `m[k] = v`              | `put(k, v)`                          | `set(k, v)`                     |
| Get              | `m[k]`                  | `get(k)` (null if missing)           | `get(k)` (undefined if missing) |
| Get with default | `m[k]`                  | `getOrDefault(k, 0)`                 | `get(k) ?? 0`                   |
| Exists           | `count(k)`              | `containsKey(k)`                     | `has(k)`                        |
| Erase            | `erase(k)`              | `remove(k)`                          | `delete(k)`                     |
| Size             | `size()`                | `size()`                             | `.size` (no brackets)           |
| Iterate          | `for (auto [k, v] : m)` | `entrySet()`, `keySet()`, `values()` | `for (const [k, v] of m)`       |

| Task   | C++ `unordered_set` | Java `HashSet` | JavaScript `Set` |
| ------ | ------------------- | -------------- | ---------------- |
| Insert | `insert(x)`         | `add(x)`       | `add(x)`         |
| Exists | `count(x)`          | `contains(x)`  | `has(x)`         |
| Erase  | `erase(x)`          | `remove(x)`    | `delete(x)`      |
| Size   | `size()`            | `size()`       | `.size`          |

**Example: frequency count**

```cpp
unordered_map<int, int> f;
for (int x : a) f[x]++;
```

```java
HashMap<Integer, Integer> f = new HashMap<>();
for (int x : a) f.merge(x, 1, Integer::sum);
```

```js
const f = new Map();
for (const x of a) f.set(x, (f.get(x) ?? 0) + 1);
```

Java has no `f[x]++` shortcut, so write `merge` or `put(x, getOrDefault(x, 0) + 1)`.

### How to use a pair as a key

In C++, `map<pair<int,int>, int>` works directly. In Java and JavaScript, an array used as a key is compared by **reference**, not by value, so the lookup fails.

```java
record Pair(int a, int b) {}                       // Java 16+, equals and hashCode come for free
HashSet<Pair> seen = new HashSet<>();
seen.add(new Pair(1, 2));

long key = (long) a * 1_000_003 + b;               // or pack the pair into one long
```

```js
const seen = new Set();
seen.add(a * 1000003 + b); // number key, fast
seen.add(`${a},${b}`); // string key, easy but slow
```

**Watch out:**

- In Java, `int c = m.get(k);` throws NullPointerException if `k` is missing. Use `getOrDefault`.
- In JavaScript, do not use a plain object `{}` as a map: keys become strings. Use `Map`.
- JavaScript `Map` and `Set` remember insertion order. They are not sorted.
- C++ `unordered_map` can be hacked with anti-hash tests on Codeforces. Java `HashMap` turns long collision chains into trees, so it is mostly safe.
- `offer`, `poll` and `peek` are not used with `HashMap` or `HashSet`. They belong to queues, deques and heaps (Chapters 5 and 6). For hash containers, use `put`, `get`, `remove`, `add` and `contains`.

**Practice:** Two Sum: given an array and a target, find the two indices whose sum is the target.

## 9. Graph, DFS and DSU

### Adjacency list

```cpp
vector<vector<int>> g(n);
g[u].push_back(v);
vector<vector<pair<int,int>>> w(n);     // weighted
```

```java
@SuppressWarnings("unchecked")
List<Integer>[] g = new ArrayList[n];
for (int i = 0; i < n; i++) g[i] = new ArrayList<>();
g[u].add(v);
// weighted: make List<int[]>[] and write g[u].add(new int[]{v, w})
```

```js
const g = Array.from({ length: n }, () => []);
g[u].push(v);
// weighted: g[u].push([v, w])
```

In Java, the compiler shows a warning when you create a generic array. This is normal. `@SuppressWarnings` hides it.

### DFS and recursion depth

In C++, a recursive DFS with depth 10^5 ran fine. In Java and JavaScript the default stack is small, so the same code can crash.

**JavaScript:** write DFS iteratively.

```js
function dfs(start) {
  const stack = [start];
  vis[start] = true;
  while (stack.length) {
    const u = stack.pop();
    for (const v of g[u]) {
      if (!vis[v]) {
        vis[v] = true;
        stack.push(v);
      }
    }
  }
}
```

This is fine for connectivity and counting components. If you need exact preorder or postorder, keep an `index` pointer for each node.

**Java:** write it iteratively, or run the program in a thread with a big stack.

```java
public static void main(String[] args) {
    new Thread(null, Main::solve, "run", 1 << 28).start();   // 256 MB stack
}
```

With this method, handle IOException inside `solve()` with `try/catch`, because a Runnable cannot use `throws`.

### DSU (Union-Find)

The logic is the same as the C++ you already write. Here we use path halving instead of recursion, so a deep chain cannot crash. In C++, `union` is a keyword, so the function is named `unite`.

```java
static int[] par, sz;

static void init(int n) {
    par = new int[n]; sz = new int[n];
    for (int i = 0; i < n; i++) { par[i] = i; sz[i] = 1; }
}
static int find(int x) {
    while (par[x] != x) { par[x] = par[par[x]]; x = par[x]; }   // path halving
    return x;
}
static boolean unite(int a, int b) {
    a = find(a); b = find(b);
    if (a == b) return false;
    if (sz[a] < sz[b]) { int t = a; a = b; b = t; }
    par[b] = a; sz[a] += sz[b];
    return true;
}
```

```js
const par = Array.from({ length: n }, (_, i) => i);
const sz = new Array(n).fill(1);

function find(x) {
  while (par[x] !== x) {
    par[x] = par[par[x]];
    x = par[x];
  }
  return x;
}
function unite(a, b) {
  a = find(a);
  b = find(b);
  if (a === b) return false;
  if (sz[a] < sz[b]) [a, b] = [b, a];
  par[b] = a;
  sz[a] += sz[b];
  return true;
}
```

**Practice:** Write Kruskal's MST with DSU. Sort the edges by weight (Chapter 3), then call `unite` on each one.

## 10. Utility functions

| C++                            | Java                                                     | JavaScript                               |
| ------------------------------ | -------------------------------------------------------- | ---------------------------------------- |
| `min`, `max`, `abs`            | `Math.min`, `Math.max`, `Math.abs`                       | same                                     |
| `sqrt`, `pow`, `floor`, `ceil` | `Math.sqrt`, `Math.pow`, `Math.floor`, `Math.ceil`       | same, and `**` for power                 |
| `__gcd(a, b)`                  | write it yourself                                        | write it yourself                        |
| `lower_bound(a, a + n, x)`     | write it yourself                                        | write it yourself                        |
| `unique`                       | `TreeSet` or `LinkedHashSet`                             | `[...new Set(a)]`                        |
| `__builtin_popcount`           | `Integer.bitCount(x)` / `Long.bitCount(x)`               | write a loop                             |
| `__builtin_clz`                | `Integer.numberOfLeadingZeros(x)`                        | `Math.clz32(x)`                          |
| `__builtin_ctz`                | `Integer.numberOfTrailingZeros(x)`                       | `31 - Math.clz32(x & -x)`                |
| `iota`                         | `IntStream.range(0, n)`                                  | `Array.from({ length: n }, (_, i) => i)` |
| `bitset<N>`                    | `BitSet`                                                 | manual, with `Uint32Array`               |
| Big integer                    | `BigInteger` (`add`, `multiply`, `mod`, `modPow`, `gcd`) | `BigInt`                                 |
| `next_permutation`             | write it yourself                                        | write it yourself                        |

### GCD

```java
static long gcd(long a, long b) { return b == 0 ? a : gcd(b, a % b); }
```

```js
const gcd = (a, b) => {
  while (b) [a, b] = [b, a % b];
  return a;
};
```

### Modular power

```java
static long power(long b, long e, long m) {
    long r = 1; b %= m;
    while (e > 0) {
        if ((e & 1) == 1) r = r * b % m;
        b = b * b % m;
        e >>= 1;
    }
    return r;
}
```

```js
// all values are BigInt: power(2n, 10n, 1000000007n)
function power(b, e, m) {
  let r = 1n;
  b %= m;
  while (e > 0n) {
    if (e & 1n) r = (r * b) % m;
    b = (b * b) % m;
    e >>= 1n;
  }
  return r;
}
```

### Lower bound (Java)

The return value of `Arrays.binarySearch` is confusing (if the value is missing, it gives a negative insertion point), so it is better to write your own. The JavaScript version is in Chapter 7.

```java
static int lowerBound(int[] a, int x) {   // first index where a[i] >= x
    int lo = 0, hi = a.length;
    while (lo < hi) {
        int mid = (lo + hi) >>> 1;
        if (a[mid] < x) lo = mid + 1; else hi = mid;
    }
    return lo;
}
```

`next_permutation` is not built in for Java or JavaScript. Generate permutations with a bitmask or backtracking instead, or write the algorithm yourself.

## 11. Mistake checklist

Look at this list once before you submit.

**In Java:**

- `Scanner` and `System.out.println` inside a loop cause TLE. Use `BufferedReader` and `StringBuilder`.
- Cast with `(long)` before you multiply two ints and store the result in a `long`.
- Do not compare `Integer` or `String` with `==`. Use `.equals()`.
- `list.remove(i)` removes by index. To remove a value, use `list.remove(Integer.valueOf(x))`.
- `Arrays.sort(int[])` can get TLE from anti-quicksort tests. Shuffle first, or use `Integer[]`.
- Do not write `a - b` in a comparator. Write `Integer.compare`.
- `m.get(k)` can return null, and unboxing it throws NullPointerException.
- `poll()` and `peek()` return `null` on an empty queue, but `remove()` and `element()` throw an exception. Check `isEmpty()` first.
- For deep recursion, use a `Thread` with a big stack.

**In JavaScript:**

- `sort()` without a comparator sorts as strings.
- If `a * b` is bigger than 2^53, the answer is wrong. Use `BigInt` for problems with `% 1e9+7`.
- Bitwise operators work on 32 bits only: `1 << 31` is negative. `>>>` is an unsigned shift, and `Math.imul` does a 32-bit multiply.
- The recursion limit is about 10^4 frames, so write DFS iteratively.
- `Math.max(...arr)` overflows the stack on a big array. Use a loop or `reduce`.
- `shift()` and `unshift()` are O(n). Use a head pointer in BFS.
- `new Array(n).fill([])` makes every row the same array. Use `Array.from`.
- Mixing `BigInt` and `Number` throws a TypeError.

## 12. Practice plan

| Step | What to do                                                                                                   | Goal                                   |
| ---- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------- |
| 1    | Save the Chapter 1 I/O template and solve 5 easy problems                                                    | Stop getting stuck on input and output |
| 2    | Rewrite your old C++ solutions (arrays, sorting, strings) in Java                                            | Get used to the syntax                 |
| 3    | Rewrite solved problems on map, set, BFS and Dijkstra                                                        | Remember the STL equivalents           |
| 4    | Build a template file. Java: I/O, DSU, `power`, `lowerBound`. JavaScript: I/O, `Heap`, `lowerBound`, Fenwick | Copy and paste in contests             |
| 5    | Solve new problems directly in Java                                                                          | Stop thinking in C++ and translating   |
