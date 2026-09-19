---
title: "DSA Pattern Index: Complete Field Manual"
description: "A comprehensive 22-pattern unified system for recognizing and solving any algorithmic problem. From operator precedence through advanced tree patterns, this field manual covers classic algorithms, recognition patterns, and optimized C++ implementations for competitive programming."
author: ["name": "Rajendra Pancholi", "email": "rpancholi522@gmail.com" ]
thumbnail: "/images/dsa-pattern-index.png"
tags: [Data Structures, Algorithms, Competitive Programming, C++ Programming, Pattern Recognition, Problem Solving, Binary Search, Dynamic Programming, Graph Algorithms, String Algorithms]
keywords: ["DSA Pattern Index", "Complete Data Structures and Algorithms tutorial", "Competitive programming guide", "C++ algorithm templates", "Pattern recognition for coding interviews", "22 essential algorithmic patterns", "Binary search techniques", "Dynamic programming patterns", "Graph traversal algorithms", "Advanced tree patterns", "Bit manipulation tricks", "String matching algorithms", "Segment tree implementation", "Union-Find data structure", "Greedy algorithm patterns"]
---

# DSA Pattern Index: Field Manual

![DSA Pattern Index: Complete Field Manual](/images/dsa-pattern-index.png)


Twenty-two advanced patterns, one unified filing system for recognizing any problem.

Every table follows the same rule: read the left column as the clue in the problem statement, the right column as the technique it's pointing to. Worked C++ examples follow the classic patterns in each section.

> Assume `#include <bits/stdc++.h>` and `using namespace std;` ahead of every snippet below.

## 00. C++ Operator Precedence

Higher precedence = evaluated first. Same precedence = follow associativity (left-to-right or right-to-left).

### Operator Precedence Table (Highest to Lowest)

| Precedence | Operator | Description | Associativity |
|---|---|---|---|
| 1 | `() [] . -> :: ++ --` | Parentheses, Array subscript, Member access, Scope, Post-increment/decrement | Left-to-right |
| 2 | `! ~ ++ -- + - * & sizeof` | Logical NOT, Bitwise NOT, Pre-increment/decrement, Unary +/-, Dereference, Address-of, sizeof | Right-to-left |
| 3 | `* / %` | Multiplication, Division, Modulo | Left-to-right |
| 4 | `+ -` | Addition, Subtraction | Left-to-right |
| 5 | `<< >>` | Bitwise left shift, right shift | Left-to-right |
| 6 | `< <= > >=` | Relational operators | Left-to-right |
| 7 | `== !=` | Equality operators | Left-to-right |
| 8 | `&` | Bitwise AND | Left-to-right |
| 9 | `^` | Bitwise XOR | Left-to-right |
| 10 | `\|` | Bitwise OR | Left-to-right |
| 11 | `&&` | Logical AND | Left-to-right |
| 12 | `\|\|` | Logical OR | Left-to-right |
| 13 | `? :` | Ternary conditional | Right-to-left |
| 14 | `= += -= *= /= %= &= ^= \|= <<= >>=` | Assignment operators | Right-to-left |
| 15 | `,` | Comma operator | Left-to-right |

### Common Mistakes in Precedence

| Code | Evaluates As | Correct? |
|---|---|---|
| `a + b * c` | `a + (b * c)` | ✓ YES (* before +) |
| `a && b \|\| c` | `(a && b) \|\| c` | ✓ YES (&& before \|\|) |
| `a \| b & c` | `a \| (b & c)` | ✓ YES (& before \|) |
| `a < b && c < d` | `(a < b) && (c < d)` | ✓ YES (< before &&) |
| `a = b = c` | `a = (b = c)` | ✓ YES (= is right-to-left) |
| `!a && b` | `(!a) && b` | ✓ YES (! before &&) |
| `a * -b` | `a * (-b)` | ✓ YES (unary - before *) |
| `a > b ? c : d + e` | `a > b ? c : (d + e)` | ✓ YES (+ before ?:) |

### Gotchas & Tips

**Bitwise vs Logical:**
- `&` and `|` = bitwise (higher precedence)
- `&&` and `||` = logical (lower precedence)
- `a & b == c` → `a & (b == c)` (NOT what you want!)
- Use: `(a & b) == c`

**Assignment vs Comparison:**
- `a = b = c` → `a = (b = c)` ✓ (right-to-left)
- `a == b == c` → `(a == b) == c` (different!)

**Shift vs Comparison:**
- `a < b << c` → `a < (b << c)`
- Use: `(a < b) << c` if needed

**Always use parentheses when unsure** — it's faster than debugging!

## 01. Sorting Techniques

Recognizing which sort a problem is secretly asking for — often the whole solution once you see it.

### Recognition

| When you see… | Pattern |
|---|---|
| Sort in-place, values already in range 1..n or 0..n-1 | Cyclic Sort |
| Count inversions / merge two sorted halves | Merge Sort (merge step) |
| Kth smallest / largest without a full sort | Quickselect (partition-based) |
| Array of only 0s, 1s, 2s (3 distinct values) | Dutch National Flag (3-way partition) |
| Sort by frequency, small integer range | Counting Sort / Bucket Sort |
| Need guaranteed O(n log n), in-place, stability not required | Heap Sort |
| Need a stable sort and can afford O(n) extra space | Merge Sort |
| General-purpose, fastest average case, in-place | Quick Sort |
| Fixed-width integers or strings, digit by digit | Radix Sort |

### Complexity at a Glance

| Algorithm | Time | Space | Stable |
|---|---|---|---|
| Bubble | O(n²) | O(1) | Yes |
| Selection | O(n²) | O(1) | No |
| Insertion | O(n²), O(n) best | O(1) | Yes |
| Merge | O(n log n) | O(n) | Yes |
| Quick | O(n log n) avg, O(n²) worst | O(log n) | No |
| Heap | O(n log n) | O(1) | No |
| Counting | O(n+k) | O(k) | Yes |
| Radix | O(d(n+k)) | O(n+k) | Yes |
| Cyclic | O(n) | O(1) | — |

### Example Algorithms

**Quick Sort (Lomuto partition)**
```cpp
int partition(vector<int>& a, int lo, int hi) {
    int pivot = a[hi], i = lo - 1;
    for (int j = lo; j < hi; j++) {
        if (a[j] < pivot) swap(a[++i], a[j]);
    }
    swap(a[i + 1], a[hi]);
    return i + 1;
}

void quickSort(vector<int>& a, int lo, int hi) {
    if (lo >= hi) return;
    int p = partition(a, lo, hi);
    quickSort(a, lo, p - 1);
    quickSort(a, p + 1, hi);
}
```

**Merge Sort**
```cpp
void merge(vector<int>& a, int lo, int mid, int hi) {
    vector<int> tmp;
    int i = lo, j = mid + 1;
    while (i <= mid && j <= hi)
        tmp.push_back(a[i] <= a[j] ? a[i++] : a[j++]);
    while (i <= mid) tmp.push_back(a[i++]);
    while (j <= hi)  tmp.push_back(a[j++]);
    for (int k = lo; k <= hi; k++) a[k] = tmp[k - lo];
}

void mergeSort(vector<int>& a, int lo, int hi) {
    if (lo >= hi) return;
    int mid = lo + (hi - lo) / 2;
    mergeSort(a, lo, mid);
    mergeSort(a, mid + 1, hi);
    merge(a, lo, mid, hi);
}
```

**Cyclic Sort — find the missing number in [0, n]**
```cpp
int missingNumber(vector<int>& a) {
    int n = a.size(), i = 0;
    while (i < n) {
        int correct = a[i];
        if (correct < n && a[correct] != a[i]) swap(a[i], a[correct]);
        else i++;
    }
    for (i = 0; i < n; i++)
        if (a[i] != i) return i;
    return n;
}
```

## 02. Arrays

Ordered easy → medium → hard, since later patterns often build on earlier ones.

### Easy Patterns

| When you see… | Pattern |
|---|---|
| Find the max/min sum contiguous subarray | Kadane's Algorithm |
| Move zeros / remove duplicates in-place | Two-pointer (read/write pointer) |
| Rotate array by k positions | Reversal trick (reverse whole, then parts) |
| Union / intersection of two sorted arrays | Two-pointer merge |
| Leaders in array / next greater on the right | Right-to-left scan, track running max |

### Medium Patterns

| When you see… | Pattern |
|---|---|
| Rearrange positives and negatives alternately | Two-pointer with placeholder swap |
| Next permutation | Find pivot from right, swap, reverse suffix |
| Subarray sum equals K | Prefix sum + HashMap |
| Majority element (appears > n/2 times) | Boyer–Moore Voting |
| Majority element (appears > n/3 times) | Extended Boyer–Moore (2 candidates) |
| 3Sum / 4Sum | Sort + two-pointer, fix k−2 elements |
| Merge overlapping intervals | Sort by start, sweep & merge |
| Merge two sorted arrays in-place | Gap method, or two-pointer from the back |
| Count inversions | Merge sort modification |
| Max product subarray | Track running max *and* min (negatives flip) |

### Hard Patterns

| When you see… | Pattern |
|---|---|
| Median of two sorted arrays | Binary search on the smaller array's partition |
| Trapping rain water | Two-pointer with leftMax/rightMax |
| Largest rectangle in histogram / binary matrix | Monotonic stack |
| Count subarrays with XOR = K | Prefix XOR + HashMap |
| Smallest window containing all elements | Sliding window + HashMap |

### Example Algorithms

**Kadane's Algorithm — max subarray sum**
```cpp
int maxSubArray(vector<int>& a) {
    int best = a[0], cur = a[0];
    for (int i = 1; i < (int)a.size(); i++) {
        cur = max(a[i], cur + a[i]);
        best = max(best, cur);
    }
    return best;
}
```

**Next Permutation**
```cpp
void nextPermutation(vector<int>& a) {
    int n = a.size(), i = n - 2;
    while (i >= 0 && a[i] >= a[i + 1]) i--;
    if (i >= 0) {
        int j = n - 1;
        while (a[j] <= a[i]) j--;
        swap(a[i], a[j]);
    }
    reverse(a.begin() + i + 1, a.end());
}
```

**3Sum — all unique triplets summing to 0**
```cpp
vector<vector<int>> threeSum(vector<int> a) {
    sort(a.begin(), a.end());
    vector<vector<int>> res;
    int n = a.size();
    for (int i = 0; i < n - 2; i++) {
        if (i > 0 && a[i] == a[i - 1]) continue;
        int lo = i + 1, hi = n - 1;
        while (lo < hi) {
            int sum = a[i] + a[lo] + a[hi];
            if (sum < 0) lo++;
            else if (sum > 0) hi--;
            else {
                res.push_back({a[i], a[lo], a[hi]});
                lo++; hi--;
                while (lo < hi && a[lo] == a[lo - 1]) lo++;
                while (lo < hi && a[hi] == a[hi + 1]) hi--;
            }
        }
    }
    return res;
}
```

**Trapping Rain Water — two pointer**
```cpp
int trap(vector<int>& h) {
    int lo = 0, hi = h.size() - 1;
    int leftMax = 0, rightMax = 0, water = 0;
    while (lo < hi) {
        if (h[lo] <= h[hi]) {
            leftMax = max(leftMax, h[lo]);
            water += leftMax - h[lo];
            lo++;
        } else {
            rightMax = max(rightMax, h[hi]);
            water += rightMax - h[hi];
            hi--;
        }
    }
    return water;
}
```

## 03. Binary Search

Core signal: the search space is sorted *or* monotonic — even a yes/no feasibility check that flips once as a value increases counts.

### 1D Array — Direct Search

| When you see… | Pattern |
|---|---|
| Find an element in a sorted array | Standard binary search: lo=0, hi=n-1, while lo≤hi |
| Find first / last occurrence of X | Lower / upper bound: lo<hi, shrink towards X |
| Count total occurrences | Upper bound − lower bound |
| Search insert position | Return lo (first position ≥ target) |
| Floor and Ceil in sorted array | Maintain boundaries while shrinking |

### 1D Array — Rotated Sorted Array

| When you see… | Pattern |
|---|---|
| Search in rotated sorted array (no duplicates) | Check a[lo]≤a[mid], identify sorted half, eliminate unsorted |
| Search rotated array with duplicates | When a[lo]==a[mid]==a[hi], shrink: lo++, hi-- |
| Find minimum in rotated sorted array | Compare mid with hi, eliminate larger side |
| How many times array is rotated | Find index of minimum element |

### 1D Array — Peak / Extrema / Special Cases

| When you see… | Pattern |
|---|---|
| Find a peak element | Compare a[mid] with a[mid+1], move toward larger |
| Single element in sorted array (others ×2) | Binary search on index parity (odd/even) |
| Sqrt(x), Nth root | Binary search on answer value: mid² vs x |

### Binary Search on Answer — Minimize/Maximize

| When you see… | Pattern |
|---|---|
| Minimize X (speed, time, cost, capacity) | BS on answer + feasibility: hi=mid if valid, else lo=mid+1 |
| Maximize X (distance, hours, allocation) | BS on answer + feasibility: lo=mid+1 if valid, else hi=mid |
| Minimize eating speed (hours ≤ h) | Binary search on speed, check sum of ceil(pile/speed) |
| Ship packages in D days (min capacity) | Binary search on capacity, greedy day packing |
| Allocate books to K painters (minimize max) | Binary search on max pages, greedy allocation |
| Aggressive cows (maximize min distance) | Binary search on distance, greedy placement check |
| Kth missing positive number | Binary search on value, count missing ≤ mid |
| Minimize max distance to gas station | Binary search on distance, count stations needed |
| Minimize days to make M bouquets | Binary search on days, check adjacent bloom feasibility |
| Find smallest divisor (threshold) | Binary search on divisor, sum of divisions check |

### 2D Array

| When you see… | Pattern |
|---|---|
| Row-wise & column-wise sorted matrix (search) | Start top-right corner, move left if big, down if small |
| Find row with maximum 1's | Binary search each row, optimize pointer movement |
| Matrix sorted like flattened 1D array | Treat as 1D: row=mid/cols, col=mid%cols |
| Kth smallest in sorted matrix | Binary search on value, count elements ≤ mid |
| Peak element in 2D (no larger neighbor) | Binary search on column, compare mid neighbors |
| Matrix median | Binary search on value, binary count check |

### Two Sorted Arrays

| When you see… | Pattern |
|---|---|
| Median of two sorted arrays | Binary search on partition, balance left/right halves |
| Kth element of two sorted arrays | Binary search on position or elimination approach |

### Example Algorithms

**Binary Search — Standard Template**
```cpp
int binarySearch(vector<int>& a, int target) {
    int lo = 0, hi = a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        else if (a[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

**Lower Bound (First ≥ Target)**
```cpp
int lowerBound(vector<int>& a, int target) {
    int lo = 0, hi = a.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < target) lo = mid + 1;
        else hi = mid;
    }
    return lo;
}
```

**Search in Rotated Sorted Array**
```cpp
int search(vector<int>& a, int target) {
    int lo = 0, hi = a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        if (a[lo] <= a[mid]) {
            if (a[lo] <= target && target < a[mid]) hi = mid - 1;
            else lo = mid + 1;
        } else {
            if (a[mid] < target && target <= a[hi]) lo = mid + 1;
            else hi = mid - 1;
        }
    }
    return -1;
}
```

**Binary Search on Answer Template**
```cpp
bool canAchieve(vector<int>& arr, long long mid) {
    // Check if answer 'mid' satisfies constraint
    return true/false;
}

long long binarySearchAnswer(vector<int>& arr) {
    long long lo = MIN_VALUE;
    long long hi = MAX_VALUE;
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (canAchieve(arr, mid)) {
            hi = mid;  // try smaller (minimize)
        } else {
            lo = mid + 1;
        }
    }
    return lo;
}
```

**Staircase Walk (2D Sorted Matrix)**
```cpp
bool searchMatrix(vector<vector<int>>& matrix, int target) {
    int row = 0, col = matrix[0].size() - 1;  // top-right
    while (row < matrix.size() && col >= 0) {
        if (matrix[row][col] == target) return true;
        else if (matrix[row][col] > target) col--;
        else row++;
    }
    return false;
}
```

## 04. Strings

### Recognition

| When you see… | Pattern |
|---|---|
| Check anagram | Frequency array / HashMap comparison |
| Longest palindromic substring | Expand around center, DP, or Manacher's (optimal) |
| Check palindrome ignoring case / punctuation | Two-pointer from both ends |
| Longest common prefix | Vertical scanning, or sort & compare first/last |
| Pattern matching (substring search) | KMP / Z-function / Rabin–Karp |
| Minimum insertions/deletions to make palindrome | LCS with reversed string (DP) |
| Group anagrams | HashMap keyed by sorted string or char-count signature |
| Roman ↔ integer | Greedy symbol mapping |
| Word break | DP + HashSet dictionary lookup |
| Longest repeating substring / distinct substrings | Suffix Array or Trie |
| String compression / run-length encoding | Two-pointer, count consecutive chars |
| Valid parentheses / balanced brackets | Stack |

### Example Algorithms

**KMP — substring search in O(n+m)**
```cpp
vector<int> buildLPS(const string& p) {
    int m = p.size();
    vector<int> lps(m, 0);
    int len = 0, i = 1;
    while (i < m) {
        if (p[i] == p[len]) lps[i++] = ++len;
        else if (len) len = lps[len - 1];
        else lps[i++] = 0;
    }
    return lps;
}

vector<int> kmpSearch(const string& text, const string& pat) {
    vector<int> lps = buildLPS(pat), matches;
    int i = 0, j = 0;
    while (i < (int)text.size()) {
        if (text[i] == pat[j]) { i++; j++; }
        if (j == (int)pat.size()) {
            matches.push_back(i - j);
            j = lps[j - 1];
        } else if (i < (int)text.size() && text[i] != pat[j]) {
            j ? j = lps[j - 1] : i++;
        }
    }
    return matches;
}
```

**Longest Palindromic Substring — expand around center**
```cpp
string longestPalindrome(string s) {
    int start = 0, best = 0;
    auto expand = [&](int l, int r) {
        while (l >= 0 && r < (int)s.size() && s[l] == s[r]) { l--; r++; }
        if (r - l - 1 > best) { best = r - l - 1; start = l + 1; }
    };
    for (int i = 0; i < (int)s.size(); i++) {
        expand(i, i);
        expand(i, i + 1);
    }
    return s.substr(start, best);
}
```

## 05. Linked List

### Single Linked List

| When you see… | Pattern |
|---|---|
| Detect a cycle | Floyd's slow/fast pointer |
| Find the start of the cycle | After meeting, reset one pointer to head, move both by 1 |
| Find the middle of the list | Slow/fast pointer (fast moves 2x) |
| Reverse a linked list | Iterative 3-pointer (prev/curr/next), or recursive |
| Reverse in groups of K | Recursive or iterative with a group-boundary check |
| Remove the Nth node from the end | Two-pointer with an N-node gap |
| Check palindrome list | Find middle → reverse second half → compare |
| Merge two sorted lists | Dummy node + two-pointer merge |
| Add two numbers as linked lists | Simulate addition with carry, dummy node |
| Intersection point of two lists | Two-pointer, switch heads to equalize path length |
| Sort a linked list | Merge sort (slow/fast pointer to split) |

### Doubly Linked List

| When you see… | Pattern |
|---|---|
| LRU Cache | DLL + HashMap — O(1) get/put, move node to front |
| Flatten a multilevel / child-pointer list | Recursive merge, similar to merge-sort's merge step |

### Medium / Hard

| When you see… | Pattern |
|---|---|
| Clone a list with a random pointer | HashMap old→new, or interweaving trick |
| Rotate list by K places | Make it circular, break at the right point |
| Reorder list: L0 → Ln → L1 → Ln−1 … | Find middle, reverse second half, merge alternately |
| Merge K sorted lists | Min-heap of size K, or divide & conquer merge |

### Example Algorithms

**Reverse a Linked List**
```cpp
struct ListNode { int val; ListNode* next; };

ListNode* reverseList(ListNode* head) {
    ListNode *prev = nullptr, *curr = head;
    while (curr) {
        ListNode* nxt = curr->next;
        curr->next = prev;
        prev = curr;
        curr = nxt;
    }
    return prev;
}
```

**Detect Cycle & Find Its Start — Floyd's algorithm**
```cpp
struct ListNode { int val; ListNode* next; };

ListNode* detectCycleStart(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) {
            slow = head;
            while (slow != fast) { slow = slow->next; fast = fast->next; }
            return slow;
        }
    }
    return nullptr;
}
```

**Merge Two Sorted Lists**
```cpp
struct ListNode { int val; ListNode* next; };

ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    ListNode dummy(0);
    ListNode* tail = &dummy;
    while (l1 && l2) {
        if (l1->val <= l2->val) { tail->next = l1; l1 = l1->next; }
        else                    { tail->next = l2; l2 = l2->next; }
        tail = tail->next;
    }
    tail->next = l1 ? l1 : l2;
    return dummy.next;
}
```

**LRU Cache — doubly linked list + hash map**
```cpp
class LRUCache {
    struct Node { int key, val; Node *prev, *next; };
    int capacity;
    unordered_map<int, Node*> cache;
    Node *head, *tail;

    void remove(Node* n) {
        n->prev->next = n->next;
        n->next->prev = n->prev;
    }
    void insertFront(Node* n) {
        n->next = head->next;
        n->prev = head;
        head->next->prev = n;
        head->next = n;
    }

public:
    LRUCache(int cap) : capacity(cap) {
        head = new Node{0, 0, nullptr, nullptr};
        tail = new Node{0, 0, nullptr, nullptr};
        head->next = tail;
        tail->prev = head;
    }

    int get(int key) {
        if (!cache.count(key)) return -1;
        Node* n = cache[key];
        remove(n);
        insertFront(n);
        return n->val;
    }

    void put(int key, int value) {
        if (cache.count(key)) {
            cache[key]->val = value;
            remove(cache[key]);
            insertFront(cache[key]);
            return;
        }
        if ((int)cache.size() == capacity) {
            Node* lru = tail->prev;
            remove(lru);
            cache.erase(lru->key);
            delete lru;
        }
        Node* n = new Node{key, value, nullptr, nullptr};
        cache[key] = n;
        insertFront(n);
    }
};
```

## 06. Recursion — Pattern-wise

### Recognition

| When you see… | Pattern |
|---|---|
| Generate all subsets / subsequences | Include/exclude recursion (pick / not-pick) |
| Subsets with duplicates, unique only | Sort first, skip adjacent duplicates at the same recursion level |
| Combination Sum (reuse allowed) | Pick/not-pick, stay at the same index when picking |
| Combination Sum II (no reuse, has duplicates) | Sort, skip `i > start && a[i]==a[i-1]` |
| Permutations | Swap-based recursion, or a visited[] array with backtracking |
| Permutations with duplicates | Sort + visited[] + skip same-adjacent unless the previous was used |
| Palindrome partitioning | Backtrack: try every prefix, recurse on the rest if it's a palindrome |
| N-Queens / Sudoku solver | Backtracking with a placement-validity check |
| Word search in a grid | DFS backtracking with visited marking/unmarking |
| Kth permutation sequence | Factorial number system — no need to generate all |

### Example Algorithms

**Subsets — pick / not-pick**
```cpp
void solve(vector<int>& nums, int i, vector<int>& cur, vector<vector<int>>& res) {
    if (i == (int)nums.size()) { res.push_back(cur); return; }
    solve(nums, i + 1, cur, res);
    cur.push_back(nums[i]);
    solve(nums, i + 1, cur, res);
    cur.pop_back();
}

vector<vector<int>> subsets(vector<int>& nums) {
    vector<vector<int>> res;
    vector<int> cur;
    solve(nums, 0, cur, res);
    return res;
}
```

**N-Queens — backtracking**
```cpp
bool isSafe(vector<string>& board, int row, int col, int n) {
    for (int i = 0; i < row; i++)
        if (board[i][col] == 'Q') return false;
    for (int i = row, j = col; i >= 0 && j >= 0; i--, j--)
        if (board[i][j] == 'Q') return false;
    for (int i = row, j = col; i >= 0 && j < n; i--, j++)
        if (board[i][j] == 'Q') return false;
    return true;
}

void solve(int row, int n, vector<string>& board, vector<vector<string>>& res) {
    if (row == n) { res.push_back(board); return; }
    for (int col = 0; col < n; col++) {
        if (isSafe(board, row, col, n)) {
            board[row][col] = 'Q';
            solve(row + 1, n, board, res);
            board[row][col] = '.';
        }
    }
}
```

## 07. Bit Manipulation

### Recognition

| When you see… | Pattern |
|---|---|
| Single number, all others appear twice | XOR every element (a ^ a = 0) |
| Two numbers appear once, rest appear twice | XOR all → find a differing bit → split into two groups |
| One number appears three times, rest twice | Bit counting mod 3 |
| Count set bits from 0 to n | DP: bits[i] = bits[i>>1] + (i & 1) |
| Generate all subsets / power set | Bitmask, iterate 0 to 2ⁿ − 1 |
| Minimum bit flips to convert A to B | Count set bits in A ^ B |
| Maximum XOR of two numbers in an array | Binary Trie over bits |
| Divide two integers without * or / | Subtract powers of two via bit shifting |

### Example Algorithms

**Bit tricks cheat sheet**
```cpp
bool getBit(int n, int i)   { return (n >> i) & 1; }
int  setBit(int n, int i)   { return n | (1 << i); }
int  clearBit(int n, int i) { return n & ~(1 << i); }
int  toggleBit(int n, int i){ return n ^ (1 << i); }
int  removeLastSetBit(int n){ return n & (n - 1); }
int  isolateLastSetBit(int n){ return n & (-n); }
bool isPowerOfTwo(int n)    { return n > 0 && (n & (n - 1)) == 0; }
```

**Single Number — XOR trick**
```cpp
int singleNumber(vector<int>& a) {
    int result = 0;
    for (int x : a) result ^= x;
    return result;
}
```

**Count Set Bits — Brian Kernighan's algorithm**
```cpp
int countSetBits(int n) {
    int count = 0;
    while (n) {
        n &= (n - 1);
        count++;
    }
    return count;
}
```

## 08. Stack and Queues

### Prefix / Infix / Postfix

| Task | Approach |
|---|---|
| Infix → Postfix | Operator stack, precedence + associativity check. Pop higher/equal precedence to output before pushing current. |
| Infix → Prefix | Reverse infix, swap ( ↔ ), convert to postfix, reverse result. |
| Postfix evaluation | Scan left→right. Push operands; on an operator, pop 2, apply (op2 OP op1), and push result back. |
| Prefix evaluation | Scan right→left. Push operands; on an operator, pop 2, apply (op1 OP op2), and push result back. |
| Postfix → Infix | Scan left→right. Push operands as strings; on an operator, pop 2, concatenate as (op2 OP op1), and push back. |
| Prefix → Infix | Scan right→left. Push operands as strings; on an operator, pop 2, concatenate as (op1 OP op2), and push back. |
| Postfix → Prefix | Scan left→right. Push operands as strings; on an operator, pop 2, concatenate as OP op2 op1, and push back. |
| Prefix → Postfix | Scan right→left. Push operands as strings; on an operator, pop 2, concatenate as op1 op2 OP, and push back. |

### Monotonic Stack

| When you see… | Pattern |
|---|---|
| Next Greater Element | Monotonic decreasing stack, traverse right→left |
| Next Smaller Element | Monotonic increasing stack |
| Previous Greater / Smaller Element | Same idea, traverse left→right |
| Largest rectangle in histogram | Monotonic stack of indices, nearest smaller left/right |
| Maximal rectangle in a binary matrix | Per-row histogram trick + largest-rectangle-in-histogram |
| Stock span problem | Monotonic decreasing stack of (price, span) |
| Remove K digits to form the smallest number | Monotonic increasing stack, pop larger digits |
| Sum of subarray minimums / maximums | Monotonic stack, contribution via nearest smaller/greater bound |

### Implementation

| When you see… | Pattern |
|---|---|
| Implement a stack using queues, or vice versa | Two-container simulation |
| Min Stack (O(1) getMin) | Auxiliary stack, or store (val, currentMin) pairs |
| Circular queue / deque | Array with front/rear modulo indexing |
| Valid parentheses | Stack: push opens, pop & match on close |

### Example Algorithms

**Infix to Postfix conversion**
```cpp
int precedence(char op) {
    if (op == '+' || op == '-') return 1;
    if (op == '*' || op == '/') return 2;
    return 0;
}

string infixToPostfix(string s) {
    stack<char> st;
    string res;
    for (char c : s) {
        if (isalnum(c)) res += c;
        else if (c == '(') st.push(c);
        else if (c == ')') {
            while (!st.empty() && st.top() != '(') { res += st.top(); st.pop(); }
            st.pop();
        } else {
            while (!st.empty() && precedence(st.top()) >= precedence(c)) {
                res += st.top(); st.pop();
            }
            st.push(c);
        }
    }
    while (!st.empty()) { res += st.top(); st.pop(); }
    return res;
}
```

**Next Greater Element — monotonic stack**
```cpp
vector<int> nextGreaterElement(vector<int>& a) {
    int n = a.size();
    vector<int> res(n, -1);
    stack<int> st;
    for (int i = n - 1; i >= 0; i--) {
        while (!st.empty() && a[st.top()] <= a[i]) st.pop();
        if (!st.empty()) res[i] = a[st.top()];
        st.push(i);
    }
    return res;
}
```

**Min Stack — O(1) getMin**
```cpp
class MinStack {
    stack<pair<int,int>> st;
public:
    void push(int val) {
        int mn = st.empty() ? val : min(val, st.top().second);
        st.push({val, mn});
    }
    void pop() { st.pop(); }
    int top() { return st.top().first; }
    int getMin() { return st.top().second; }
};
```

## 09. Sliding Window & Two Pointer

Core signal: a contiguous subarray/substring with a condition on sum, count, or distinct elements — or two sorted sequences being compared from both ends.

### Recognition

| When you see… | Pattern |
|---|---|
| Fixed-size K subarray, max/min sum | Fixed window: add the new element, remove the old one |
| Longest substring without repeating characters | Variable window + HashSet/HashMap, shrink on violation |
| Longest substring with at most K distinct characters | Variable window + frequency map |
| Minimum window substring containing all of T | Expand until valid, then contract while still valid |
| Max consecutive ones with at most K flips | Variable window tracking the zero count |
| Subarray with sum exactly K (all positive) | Variable window, shrink when sum > K |
| Subarray with sum exactly K (has negatives) | Prefix sum + HashMap — *not* sliding window |
| Longest repeating character replacement | Window + track the max-frequency char |
| Fruits into baskets / at most 2 distinct types | Variable window, frequency-map size ≤ 2 |
| Two Sum in a sorted array | Two-pointer from both ends |
| Container with most water | Two-pointer, move the pointer with the smaller height |
| Sliding window maximum | Monotonic deque (decreasing) |
| Count subarrays with exactly K distinct elements | atMost(K) − atMost(K−1) trick |

### Example Algorithms

**Variable window — the template**
```cpp
int slidingWindowTemplate(vector<int>& a) {
    int left = 0, best = 0;
    for (int right = 0; right < (int)a.size(); right++) {
        // add(a[right]) into the window state

        while (/* window is invalid */ false) {
            // remove(a[left]) from the window state
            left++;
        }
        best = max(best, right - left + 1);
    }
    return best;
}
```

**Longest Substring Without Repeating Characters**
```cpp
int lengthOfLongestSubstring(string s) {
    unordered_map<char, int> last;
    int left = 0, best = 0;
    for (int right = 0; right < (int)s.size(); right++) {
        if (last.count(s[right]) && last[s[right]] >= left)
            left = last[s[right]] + 1;
        last[s[right]] = right;
        best = max(best, right - left + 1);
    }
    return best;
}
```

**Minimum Window Substring**
```cpp
string minWindow(string s, string t) {
    unordered_map<char, int> need;
    for (char c : t) need[c]++;
    int required = need.size(), formed = 0;
    unordered_map<char, int> window;
    int left = 0, bestLen = INT_MAX, bestStart = 0;

    for (int right = 0; right < (int)s.size(); right++) {
        char c = s[right];
        window[c]++;
        if (need.count(c) && window[c] == need[c]) formed++;

        while (formed == required) {
            if (right - left + 1 < bestLen) {
                bestLen = right - left + 1;
                bestStart = left;
            }
            char lc = s[left];
            window[lc]--;
            if (need.count(lc) && window[lc] < need[lc]) formed--;
            left++;
        }
    }
    return bestLen == INT_MAX ? "" : s.substr(bestStart, bestLen);
}
```

## 10. Heaps

### Recognition

| When you see… | Pattern |
|---|---|
| Kth largest / smallest element | Min-heap of size K (largest) / Max-heap of size K (smallest) |
| K closest points to the origin | Max-heap of size K by distance |
| Top K frequent elements | Min-heap of size K by frequency, or bucket sort |
| Merge K sorted lists / arrays | Min-heap of size K, one element per list |
| Median from a data stream | Two heaps: max-heap (lower half) + min-heap (upper half) |
| Task scheduler / rearrange with a cooldown | Max-heap by frequency + a cooldown queue |
| Connect ropes with minimum cost | Min-heap, always combine the two smallest |
| Dijkstra's / Prim's shortest path | Min-heap (priority queue) for greedy edge selection |
| Reorganize string (no adjacent same characters) | Max-heap by frequency |

### Example Algorithms

**Kth Largest Element in an Array**
```cpp
int findKthLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;
    for (int x : nums) {
        minHeap.push(x);
        if ((int)minHeap.size() > k) minHeap.pop();
    }
    return minHeap.top();
}
```

**Merge K Sorted Lists**
```cpp
struct ListNode { int val; ListNode* next; };

ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto l : lists) if (l) pq.push(l);

    ListNode dummy(0);
    ListNode* tail = &dummy;
    while (!pq.empty()) {
        ListNode* node = pq.top(); pq.pop();
        tail->next = node;
        tail = tail->next;
        if (node->next) pq.push(node->next);
    }
    return dummy.next;
}
```

**Median from a Data Stream — two heaps**
```cpp
class MedianFinder {
    priority_queue<int> lower;
    priority_queue<int, vector<int>, greater<int>> upper;
public:
    void addNum(int num) {
        lower.push(num);
        upper.push(lower.top()); lower.pop();
        if (upper.size() > lower.size()) {
            lower.push(upper.top()); upper.pop();
        }
    }
    double findMedian() {
        if (lower.size() > upper.size()) return lower.top();
        return (lower.top() + upper.top()) / 2.0;
    }
};
```

## 11. Greedy Algorithms

Works when a locally optimal choice provably leads to a globally optimal one. If you can find a counterexample, it's probably DP instead.

### Easy

| When you see… | Pattern |
|---|---|
| Assign cookies / minimize resource matching | Sort both, greedy two-pointer |
| Lemonade change | Track denominations greedily |
| Minimum coins (canonical coin system) | Greedy from the largest denomination |
| Fractional knapsack | Sort by value/weight ratio, descending |

### Medium / Hard

| When you see… | Pattern |
|---|---|
| Activity selection / max non-overlapping intervals | Sort by end time, greedily pick |
| Minimum meeting rooms / platforms | Sort starts & ends separately, sweep, or min-heap |
| Job sequencing with deadlines | Sort by profit desc, place in the latest free slot ≤ deadline |
| Minimum jumps to reach the end | Greedy range-extension (BFS-like) |
| Gas station (circuit tour) | If total gas ≥ total cost, a single pass finds the start |
| Candy distribution | Two-pass greedy (left → right, then right → left) |
| Insert interval / merge intervals | Sort + sweep |
| Partition labels | Track last occurrence, greedily widen the window |
| Huffman encoding | Min-heap, always combine the two smallest |

### Example Algorithms

**Activity Selection / N Meetings in One Room**
```cpp
int maxMeetings(vector<int>& start, vector<int>& end) {
    int n = start.size();
    vector<int> idx(n);
    iota(idx.begin(), idx.end(), 0);
    sort(idx.begin(), idx.end(), [&](int i, int j) {
        return end[i] < end[j];
    });

    int count = 0, lastEnd = -1;
    for (int i : idx) {
        if (start[i] > lastEnd) {
            count++;
            lastEnd = end[i];
        }
    }
    return count;
}
```

**Job Sequencing with Deadlines**
```cpp
struct Job { int id, deadline, profit; };

int jobSequencing(vector<Job>& jobs) {
    sort(jobs.begin(), jobs.end(), [](Job& a, Job& b) {
        return a.profit > b.profit;
    });
    int maxDeadline = 0;
    for (auto& j : jobs) maxDeadline = max(maxDeadline, j.deadline);

    vector<bool> slot(maxDeadline + 1, false);
    int totalProfit = 0;
    for (auto& j : jobs) {
        for (int d = j.deadline; d > 0; d--) {
            if (!slot[d]) { slot[d] = true; totalProfit += j.profit; break; }
        }
    }
    return totalProfit;
}
```

## 12. Binary Trees

### Traversals

| Type | Method |
|---|---|
| Preorder (Root–L–R) | Recursive, or a stack (push right then left) |
| Inorder (L–Root–R) | Recursive, or a stack, unwinding via `cur = cur->left` |
| Postorder (L–R–Root) | Recursive, or two stacks / one stack + reversed preorder |
| Level order (BFS) | Queue, snapshot the level size for level-wise grouping |
| Morris Traversal | O(1) space, threaded tree via predecessor links |

### Medium / Hard

| When you see… | Pattern |
|---|---|
| Height / diameter of the tree | DFS returning height, update a global max at each node |
| Balanced binary tree check | DFS returning height, −1 sentinel signals imbalance |
| Lowest common ancestor | Recursive — return the node if found in both subtrees |
| Max path sum (any node to any node) | DFS returning the max single-path gain, track a global max |
| Zigzag / spiral level order | BFS + an alternating direction flag |
| Construct tree from inorder + preorder/postorder | Recursive split using an index map for inorder |
| Symmetric / mirror tree check | Recursive compare of left.left vs right.right, left.right vs right.left |
| Serialize / deserialize a tree | Preorder with null markers, or level order with a queue |
| Right / left view of the tree | BFS take last/first per level, or DFS with depth tracking |
| Flatten tree to a linked list | Reverse postorder (right, left, root) with a prev pointer |
| Vertical order traversal | BFS/DFS + HashMap<column, list> |

### Example Algorithms

**Traversals — recursive + level order**
```cpp
struct TreeNode { int val; TreeNode *left, *right; };

void inorder(TreeNode* root, vector<int>& out) {
    if (!root) return;
    inorder(root->left, out);
    out.push_back(root->val);
    inorder(root->right, out);
}

vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int sz = q.size();
        vector<int> level;
        for (int i = 0; i < sz; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        res.push_back(level);
    }
    return res;
}
```

**Diameter of a Binary Tree**
```cpp
int diameter = 0;

int height(TreeNode* node) {
    if (!node) return 0;
    int lh = height(node->left);
    int rh = height(node->right);
    diameter = max(diameter, lh + rh);
    return 1 + max(lh, rh);
}

int diameterOfBinaryTree(TreeNode* root) {
    diameter = 0;
    height(root);
    return diameter;
}
```

**Lowest Common Ancestor**
```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;
    TreeNode* left  = lowestCommonAncestor(root->left, p, q);
    TreeNode* right = lowestCommonAncestor(root->right, p, q);
    if (left && right) return root;
    return left ? left : right;
}
```

## 13. Binary Search Trees

### Recognition

| When you see… | Pattern |
|---|---|
| Search / insert / delete in a BST | Use the BST property to go left/right — O(h) |
| Validate a BST | Inorder must be strictly increasing, or a min/max bound recursion |
| Kth smallest / largest in a BST | Inorder traversal (Morris for O(1) space) |
| LCA in a BST | Compare values with root — go left/right/stop based on range |
| Convert a sorted array to a balanced BST | Recursive: pick the middle element as root |
| BST iterator with O(1) amortized next() | Controlled inorder traversal with an explicit stack |
| Two Sum in a BST | Inorder → two-pointer, or a forward + backward BST iterator |
| Recover a BST (two nodes swapped) | Inorder traversal, track the two violating nodes |
| Floor / ceiling in a BST | O(h) traversal tracking the closest ≤ / ≥ |

### Example Algorithms

**Validate a Binary Search Tree**
```cpp
bool valid(TreeNode* node, long low, long high) {
    if (!node) return true;
    if (node->val <= low || node->val >= high) return false;
    return valid(node->left, low, node->val) &&
           valid(node->right, node->val, high);
}

bool isValidBST(TreeNode* root) {
    return valid(root, LONG_MIN, LONG_MAX);
}
```

**Insert into a BST**
```cpp
TreeNode* insertIntoBST(TreeNode* root, int val) {
    if (!root) return new TreeNode{val, nullptr, nullptr};
    if (val < root->val) root->left  = insertIntoBST(root->left, val);
    else                 root->right = insertIntoBST(root->right, val);
    return root;
}
```

**Kth Smallest Element in a BST**
```cpp
int kthSmallest(TreeNode* root, int k) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (true) {
        while (cur) { st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        if (--k == 0) return cur->val;
        cur = cur->right;
    }
}
```

## 14. Graphs

### Concepts

| Concept | Notes |
|---|---|
| Representation | Adjacency list (sparse) vs. matrix (dense) |
| BFS | Level-by-level; shortest path in an *unweighted* graph |
| DFS | Recursion/stack; connectivity, cycles, topological sort |
| Topological sort | DFS (finish-time stack) or Kahn's BFS (indegree) — DAG only |
| Union–Find (DSU) | Path compression + union by rank/size, near O(1) ops |
| MST | Prim's (heap-based, dense) or Kruskal's (DSU-based, sparse) |
| Shortest path, non-negative weights | Dijkstra's (min-heap) |
| Shortest path, negative weights | Bellman–Ford (also detects negative cycles) |
| All-pairs shortest path | Floyd–Warshall, O(V³) |

### Problems

| When you see… | Pattern |
|---|---|
| Number of islands / connected components | DFS/BFS flood fill, or Union–Find |
| Shortest path in an unweighted graph/grid | BFS |
| Shortest path with weights | Dijkstra's |
| Shortest path with negative weights | Bellman–Ford |
| Cycle detection (undirected) | DFS with parent tracking, or Union–Find |
| Cycle detection (directed) | DFS with a recursion-stack, or Kahn's (leftover nodes) |
| Course schedule / task ordering | Topological sort |
| Clone a graph | DFS/BFS + HashMap old→new |
| Word ladder (min transformations) | BFS over a graph of word transformations |
| Rotten oranges / multi-source spread | Multi-source BFS |
| Bipartite graph check | BFS/DFS 2-coloring, or Union–Find |
| Minimum spanning tree | Prim's or Kruskal's |
| Alien dictionary (letter order) | Build a graph from adjacent-word comparison + topological sort |
| Critical connections / bridges | Tarjan's bridge-finding |
| Cheapest flights within K stops | Bellman–Ford limited to K iterations |

### Example Algorithms

**BFS & DFS traversal**
```cpp
void bfs(int src, vector<vector<int>>& adj, vector<bool>& visited) {
    queue<int> q;
    q.push(src);
    visited[src] = true;
    while (!q.empty()) {
        int node = q.front(); q.pop();
        for (int next : adj[node]) {
            if (!visited[next]) { visited[next] = true; q.push(next); }
        }
    }
}

void dfs(int node, vector<vector<int>>& adj, vector<bool>& visited) {
    visited[node] = true;
    for (int next : adj[node])
        if (!visited[next]) dfs(next, adj, visited);
}
```

**Dijkstra's Algorithm**
```cpp
vector<int> dijkstra(int src, int n, vector<vector<pair<int,int>>>& adj) {
    vector<int> dist(n, INT_MAX);
    priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;
    dist[src] = 0;
    pq.push({0, src});
    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;
        for (auto& [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}
```

**Bellman–Ford — handles negative weights**
```cpp
struct Edge { int u, v, weight; };

vector<long long> bellmanFord(int n, int src, vector<Edge>& edges) {
    vector<long long> dist(n, LLONG_MAX);
    dist[src] = 0;

    for (int i = 0; i < n - 1; i++) {
        for (auto& e : edges) {
            if (dist[e.u] != LLONG_MAX && dist[e.u] + e.weight < dist[e.v]) {
                dist[e.v] = dist[e.u] + e.weight;
            }
        }
    }

    for (auto& e : edges) {
        if (dist[e.u] != LLONG_MAX && dist[e.u] + e.weight < dist[e.v]) {
            throw runtime_error("negative weight cycle detected");
        }
    }
    return dist;
}
```

**Topological Sort — Kahn's algorithm (BFS)**
```cpp
vector<int> topoSort(int n, vector<vector<int>>& adj) {
    vector<int> indegree(n, 0);
    for (int u = 0; u < n; u++)
        for (int v : adj[u]) indegree[v]++;

    queue<int> q;
    for (int i = 0; i < n; i++) if (indegree[i] == 0) q.push(i);

    vector<int> order;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        order.push_back(u);
        for (int v : adj[u])
            if (--indegree[v] == 0) q.push(v);
    }
    return order;
}
```

**Union–Find (Disjoint Set) with path compression**
```cpp
struct DSU {
    vector<int> parent, rank_;
    DSU(int n) : parent(n), rank_(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }
    void unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return;
        if (rank_[rx] < rank_[ry]) swap(rx, ry);
        parent[ry] = rx;
        if (rank_[rx] == rank_[ry]) rank_[rx]++;
    }
};
```

## 15. Dynamic Programming

Overlapping subproblems + optimal substructure. Ask "what changes as I move forward?" — that becomes your state.

### Pattern Recognition

| When you see… | Pattern |
|---|---|
| Ways to reach step N — depends linearly on i−1, i−2 | 1D DP (climbing stairs, house robber) |
| Paths in a grid, min/max path sum | 2D grid DP |
| Include or exclude an item, weight limit | 0/1 Knapsack (subset sum, equal partition) |
| Unlimited use of items | Unbounded Knapsack (coin change, rod cutting) |
| Two strings, common subsequence/substring | LCS family (LCS, LPS, edit distance) |
| Longest increasing subsequence | LIS — O(n²) or O(n log n) with binary search |
| Count/find a subsequence satisfying a condition | DP on subsequences (distinct subsequences) |
| Partition or count palindromic substrings | DP on strings / palindromes |
| Optimal way to split or parenthesize a range | Interval DP (matrix chain multiplication, burst balloons) |
| Max path sum or diameter in a tree | DP on trees |
| Assign N items to N slots optimally, small N (≤20) | Bitmask DP (TSP, assignment problem) |
| Count numbers in a range with a digit property | Digit DP |
| Buy/sell stock with cooldown, fee, or K transactions | State machine DP |

### Example Algorithms

**0/1 Knapsack**
```cpp
int knapsack(vector<int>& wt, vector<int>& val, int W) {
    int n = wt.size();
    vector<vector<int>> dp(n + 1, vector<int>(W + 1, 0));
    for (int i = 1; i <= n; i++) {
        for (int w = 0; w <= W; w++) {
            dp[i][w] = dp[i - 1][w];
            if (wt[i - 1] <= w)
                dp[i][w] = max(dp[i][w], val[i - 1] + dp[i - 1][w - wt[i - 1]]);
        }
    }
    return dp[n][W];
}
```

**Longest Common Subsequence**
```cpp
int longestCommonSubsequence(string a, string b) {
    int n = a.size(), m = b.size();
    vector<vector<int>> dp(n + 1, vector<int>(m + 1, 0));
    for (int i = 1; i <= n; i++)
        for (int j = 1; j <= m; j++)
            dp[i][j] = (a[i - 1] == b[j - 1])
                ? 1 + dp[i - 1][j - 1]
                : max(dp[i - 1][j], dp[i][j - 1]);
    return dp[n][m];
}
```

**Longest Increasing Subsequence — O(n log n)**
```cpp
int lengthOfLIS(vector<int>& a) {
    vector<int> tails;
    for (int x : a) {
        auto it = lower_bound(tails.begin(), tails.end(), x);
        if (it == tails.end()) tails.push_back(x);
        else *it = x;
    }
    return tails.size();
}
```

**Coin Change — unbounded knapsack, min coins**
```cpp
int coinChange(vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, INT_MAX - 1);
    dp[0] = 0;
    for (int a = 1; a <= amount; a++)
        for (int c : coins)
            if (c <= a) dp[a] = min(dp[a], 1 + dp[a - c]);
    return dp[amount] >= INT_MAX - 1 ? -1 : dp[amount];
}
```

**Matrix Chain Multiplication — interval DP**
```cpp
int matrixChainOrder(vector<int>& dims) {
    int n = dims.size() - 1;
    vector<vector<int>> dp(n + 1, vector<int>(n + 1, 0));

    for (int len = 2; len <= n; len++) {
        for (int i = 1; i <= n - len + 1; i++) {
            int j = i + len - 1;
            dp[i][j] = INT_MAX;
            for (int k = i; k < j; k++) {
                int cost = dp[i][k] + dp[k + 1][j]
                         + dims[i - 1] * dims[k] * dims[j];
                dp[i][j] = min(dp[i][j], cost);
            }
        }
    }
    return dp[1][n];
}
```

## 16. Tries

### Recognition

| When you see… | Pattern |
|---|---|
| Implement a trie: insert / search / startsWith | Node with children[26] (or a map) + isEndOfWord flag |
| Longest word built entirely from other words in the dict | Trie + DFS, track valid words |
| Word search II — multiple words in a grid | Build a trie of the words + DFS/backtracking on the grid |
| Maximum XOR of two numbers | Binary trie (32-bit), greedy bit by bit |
| Autocomplete / prefix matching | Trie + DFS collecting words under a node |
| Replace words with their shortest dictionary root | Insert dictionary into a trie, search each word for its shortest valid prefix |

### Example Algorithms

**Trie — insert, search, startsWith**
```cpp
struct TrieNode {
    TrieNode* children[26] = {};
    bool isEnd = false;
};

class Trie {
    TrieNode* root = new TrieNode();
public:
    void insert(const string& word) {
        TrieNode* node = root;
        for (char c : word) {
            int i = c - 'a';
            if (!node->children[i]) node->children[i] = new TrieNode();
            node = node->children[i];
        }
        node->isEnd = true;
    }

    bool search(const string& word) {
        TrieNode* node = find(word);
        return node && node->isEnd;
    }

    bool startsWith(const string& prefix) {
        return find(prefix) != nullptr;
    }

private:
    TrieNode* find(const string& s) {
        TrieNode* node = root;
        for (char c : s) {
            int i = c - 'a';
            if (!node->children[i]) return nullptr;
            node = node->children[i];
        }
        return node;
    }
};
```

## 17. Strings — Advanced

### Recognition

| When you see… | Pattern |
|---|---|
| Need guaranteed O(n+m) search, or a reusable prefix-failure function | KMP (LPS array) |
| Match a pattern anywhere using one linear array of match-lengths | Z-algorithm (concat pattern + '#' + text) |
| Very large text, want to avoid re-comparing substrings char by char | Rabin–Karp rolling hash |
| Check whether string B is a rotation of string A | Search B inside A+A with KMP or Z |
| Need every palindrome's radius in O(n) total, not O(n²) | Manacher's algorithm |

### Example Algorithms

**KMP — Pattern Matching**
```cpp
vector<int> computeLPS(const string& pattern) {
    int n = pattern.size();
    vector<int> lps(n, 0);
    int len = 0, i = 1;

    while (i < n) {
        if (pattern[i] == pattern[len]) {
            lps[i] = ++len;
            i++;
        } else if (len != 0) {
            len = lps[len - 1];
        } else {
            i++;
        }
    }
    return lps;
}

vector<int> findPattern(const string& text, const string& pattern) {
    vector<int> result;
    vector<int> lps = computeLPS(pattern);
    int j = 0;

    for (int i = 0; i < (int)text.size(); i++) {
        while (j > 0 && text[i] != pattern[j]) j = lps[j - 1];
        if (text[i] == pattern[j]) j++;
        if (j == (int)pattern.size()) {
            result.push_back(i - j + 1);
            j = lps[j - 1];
        }
    }
    return result;
}
```

**Z-Algorithm — Pattern Matching**
```cpp
vector<int> computeZ(const string& s) {
    int n = s.size();
    vector<int> z(n, 0);
    int l = 0, r = 0;

    for (int i = 1; i < n; i++) {
        if (i <= r) z[i] = min(r - i + 1, z[i - l]);
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) z[i]++;
        if (i + z[i] - 1 > r) { l = i; r = i + z[i] - 1; }
    }
    return z;
}

vector<int> findPattern(const string& text, const string& pattern) {
    string combined = pattern + "#" + text;
    vector<int> z = computeZ(combined);
    vector<int> result;
    int patLen = pattern.size();

    for (int i = patLen + 1; i < (int)combined.size(); i++) {
        if (z[i] == patLen) result.push_back(i - patLen - 1);
    }
    return result;
}
```

**Rabin–Karp — Rolling Hash**
```cpp
const long long MOD = 1e9 + 7, BASE = 31;

vector<int> rabinKarp(const string& text, const string& pattern) {
    int n = text.size(), m = pattern.size();
    vector<int> result;

    long long patHash = 0, textHash = 0, pow_base = 1;
    for (int i = 0; i < m; i++) {
        patHash = (patHash * BASE + pattern[i]) % MOD;
        textHash = (textHash * BASE + text[i]) % MOD;
        if (i < m - 1) pow_base = (pow_base * BASE) % MOD;
    }

    for (int i = 0; i <= n - m; i++) {
        if (patHash == textHash && text.substr(i, m) == pattern) {
            result.push_back(i);
        }
        if (i < n - m) {
            textHash = (textHash - (long long)text[i] * pow_base % MOD + MOD) % MOD;
            textHash = (textHash * BASE + text[i + m]) % MOD;
        }
    }
    return result;
}
```

## 18. Segment Trees & Fenwick

Efficient range queries and updates in logarithmic time.

### Recognition

| When you see… | Pattern |
|---|---|
| Range sum/min/max queries + point updates | Segment Tree or Fenwick Tree |
| Range update (lazy propagation needed) | Segment Tree with lazy propagation |
| Find kth smallest in range | Segment Tree + binary search |
| Range GCD / LCM queries | Segment Tree combining GCD |
| Count queries with updates | Segment Tree or Fenwick Tree |

### Example Algorithms

**Segment Tree — Range Sum Query**
```cpp
class SegmentTree {
    vector<int> tree;
    int n;

    void build(vector<int>& arr, int node, int start, int end) {
        if (start == end) tree[node] = arr[start];
        else {
            int mid = (start + end) / 2;
            build(arr, 2*node, start, mid);
            build(arr, 2*node+1, mid+1, end);
            tree[node] = tree[2*node] + tree[2*node+1];
        }
    }

    void update(int node, int start, int end, int idx, int val) {
        if (start == end) tree[node] = val;
        else {
            int mid = (start + end) / 2;
            if (idx <= mid) update(2*node, start, mid, idx, val);
            else update(2*node+1, mid+1, end, idx, val);
            tree[node] = tree[2*node] + tree[2*node+1];
        }
    }

    int query(int node, int start, int end, int l, int r) {
        if (r < start || end < l) return 0;
        if (l <= start && end <= r) return tree[node];
        int mid = (start + end) / 2;
        return query(2*node, start, mid, l, r) +
               query(2*node+1, mid+1, end, l, r);
    }

public:
    SegmentTree(vector<int>& arr) {
        n = arr.size();
        tree.resize(4 * n);
        build(arr, 1, 0, n-1);
    }

    void update(int idx, int val) { update(1, 0, n-1, idx, val); }
    int query(int l, int r) { return query(1, 0, n-1, l, r); }
};
```

**Fenwick Tree (Binary Indexed Tree)**
```cpp
class FenwickTree {
    vector<int> tree;
    int n;

public:
    FenwickTree(int n) : n(n), tree(n+1, 0) {}

    void update(int i, int delta) {
        i++;
        while (i <= n) {
            tree[i] += delta;
            i += i & (-i);
        }
    }

    int query(int i) {
        i++;
        int sum = 0;
        while (i > 0) {
            sum += tree[i];
            i -= i & (-i);
        }
        return sum;
    }

    int rangeQuery(int l, int r) {
        if (l == 0) return query(r);
        return query(r) - query(l - 1);
    }
};
```

## 19. Math & Number Theory

Mathematical foundations for competitive programming: modular arithmetic, number theory, combinatorics.

### Recognition

| When you see… | Pattern |
|---|---|
| GCD / LCM problems | Euclidean algorithm; LCM = (a*b)/GCD |
| Modular exponentiation | Fast exponentiation with % MOD |
| Modular inverse | Fermat's little theorem (if MOD is prime) |
| Prime checking | Sieve of Eratosthenes or Miller–Rabin |
| Factorization / divisors | Trial division up to sqrt(n) |
| Combinatorics (nCr, nPr) | Precompute factorials & inverse factorials |
| Chinese Remainder Theorem | Solve x ≡ a1 (mod m1), x ≡ a2 (mod m2) |
| Extended GCD (linear Diophantine) | Find x, y such that ax + by = gcd(a, b) |

### Example Algorithms

**Sieve of Eratosthenes**
```cpp
vector<bool> sieve(int n) {
    vector<bool> is_prime(n+1, true);
    is_prime[0] = is_prime[1] = false;

    for (int i = 2; i * i <= n; i++) {
        if (is_prime[i]) {
            for (int j = i * i; j <= n; j += i) {
                is_prime[j] = false;
            }
        }
    }
    return is_prime;
}
```

**Extended GCD — Linear Diophantine**
```cpp
long long extgcd(long long a, long long b, long long &x, long long &y) {
    if (b == 0) {
        x = 1; y = 0;
        return a;
    }
    long long x1, y1;
    long long g = extgcd(b, a % b, x1, y1);
    x = y1;
    y = x1 - (a / b) * y1;
    return g;
}

bool diophantine(long long a, long long b, long long c,
                 long long &x, long long &y) {
    long long x0, y0;
    long long g = extgcd(a, b, x0, y0);
    if (c % g != 0) return false;
    x = x0 * (c / g);
    y = y0 * (c / g);
    return true;
}
```

**Combinatorics with Precomputed Factorials**
```cpp
const int MAXN = 1e5 + 5, MOD = 1e9 + 7;
long long fact[MAXN], inv_fact[MAXN];

long long power(long long a, long long b, long long m) {
    long long r = 1; a %= m;
    while (b) { if (b & 1) r = r * a % m; a = a * a % m; b >>= 1; }
    return r;
}

void precompute() {
    fact[0] = 1;
    for (int i = 1; i < MAXN; i++) fact[i] = fact[i-1] * i % MOD;
    inv_fact[MAXN-1] = power(fact[MAXN-1], MOD-2, MOD);
    for (int i = MAXN-2; i >= 0; i--)
        inv_fact[i] = inv_fact[i+1] * (i+1) % MOD;
}

long long C(int n, int r) {
    if (r < 0 || r > n) return 0;
    return fact[n] * inv_fact[r] % MOD * inv_fact[n-r] % MOD;
}
```

## 20. Advanced DP

Beyond standard recurrence: digit DP, tree DP, and state compression.

### Recognition

| When you see… | Pattern |
|---|---|
| Count numbers up to N with a digit property | Digit DP (pos, tight, leading_zero) |
| Tree DP: count/maximize on a tree structure | DFS returning DP values from children |
| State compression (small bitmask states) | DP[mask] where mask encodes a subset |
| Min/max matching / pairing | Profile DP or bitmask DP |
| Game theory / minimax | DP[state] = can the current player win? |

### Example Algorithms

**Digit DP — Count Numbers with Property**
```cpp
string num;
int memo[20][2][200];

int solve(int pos, int tight, int sum) {
    if (sum < 0) return 0;
    if (pos == (int)num.size()) return sum == 0 ? 1 : 0;

    if (memo[pos][tight][sum] != -1)
        return memo[pos][tight][sum];

    int limit = tight ? (num[pos] - '0') : 9;
    int result = 0;

    for (int digit = 0; digit <= limit; digit++) {
        int new_tight = tight && (digit == limit);
        result += solve(pos + 1, new_tight, sum - digit);
    }

    return memo[pos][tight][sum] = result;
}

int countDigitSum(int n, int target_sum) {
    num = to_string(n);
    memset(memo, -1, sizeof(memo));
    return solve(0, 1, target_sum);
}
```

**Tree DP — Max Path Sum**
```cpp
struct TreeNode {
    int val;
    TreeNode *left, *right;
};

int maxPathSum;

int dfs(TreeNode* node) {
    if (!node) return 0;

    int left_sum = max(0, dfs(node->left));
    int right_sum = max(0, dfs(node->right));

    maxPathSum = max(maxPathSum, node->val + left_sum + right_sum);

    return node->val + max(left_sum, right_sum);
}

int maxPathSumTree(TreeNode* root) {
    maxPathSum = INT_MIN;
    dfs(root);
    return maxPathSum;
}
```

**Bitmask DP — Traveling Salesman (TSP)**
```cpp
int n;
vector<vector<int>> dist;
int dp[1 << 16][16];

int tsp(int mask, int last) {
    if (mask == (1 << n) - 1) return dist[last][0];

    if (dp[mask][last] != -1) return dp[mask][last];

    int ans = INT_MAX;
    for (int city = 0; city < n; city++) {
        if (!(mask & (1 << city))) {
            int new_mask = mask | (1 << city);
            ans = min(ans, dist[last][city] + tsp(new_mask, city));
        }
    }

    return dp[mask][last] = ans;
}

int solveTSP(vector<vector<int>>& distance) {
    dist = distance;
    n = dist.size();
    memset(dp, -1, sizeof(dp));
    return tsp(1, 0);
}
```

## 21. Advanced Union-Find

Union-Find extensions: weighted edges, cycle detection, bipartite checking, and MST applications.

### Recognition

| When you see… | Pattern |
|---|---|
| Minimum spanning tree (Kruskal) | Sort edges by weight, union-find to add safely |
| Bipartite graph checking | Union-find with an enemy/opposite-group trick |
| Cycle detection in undirected graph | Union returns false if nodes already connected |
| Connected components with weights | Weighted union-find tracking rank/height |
| Potential equation (difference constraint) | Weighted union-find storing relative distances |

### Example Algorithms

**Kruskal's Algorithm — MST**
```cpp
struct Edge {
    int u, v, weight;
    bool operator<(const Edge& other) const {
        return weight < other.weight;
    }
};

struct DSU {
    vector<int> parent, rank_;
    DSU(int n) : parent(n), rank_(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }
    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }
    bool unite(int x, int y) {
        int rx = find(x), ry = find(y);
        if (rx == ry) return false;
        if (rank_[rx] < rank_[ry]) swap(rx, ry);
        parent[ry] = rx;
        if (rank_[rx] == rank_[ry]) rank_[rx]++;
        return true;
    }
};

long long kruskal(int n, vector<Edge>& edges) {
    sort(edges.begin(), edges.end());
    DSU dsu(n);
    long long mst_weight = 0;

    for (auto& e : edges) {
        if (dsu.unite(e.u, e.v)) {
            mst_weight += e.weight;
        }
    }
    return mst_weight;
}
```

**Bipartite Checking with Union–Find**
```cpp
struct BipartiteDSU {
    vector<int> parent;
    int n;

    BipartiteDSU(int n_) : n(n_), parent(2 * n_) {
        iota(parent.begin(), parent.end(), 0);
    }

    int find(int x) {
        if (parent[x] != x) parent[x] = find(parent[x]);
        return parent[x];
    }

    void unite(int x, int y) {
        parent[find(x)] = find(y);
    }

    bool addEdge(int u, int v) {
        if (find(u) == find(v)) return false;
        unite(u, v + n);
        unite(v, u + n);
        return true;
    }
};
```

**Weighted Union–Find — Potential Graph**
```cpp
struct WeightedDSU {
    vector<int> parent;
    vector<long long> weight;

    WeightedDSU(int n) : parent(n), weight(n, 0) {
        iota(parent.begin(), parent.end(), 0);
    }

    pair<int, long long> find(int x) {
        if (parent[x] == x) return {x, 0};
        auto [root, w] = find(parent[x]);
        weight[x] += w;
        parent[x] = root;
        return {root, weight[x]};
    }

    bool unite(int x, int y, long long w) {
        auto [rx, wx] = find(x);
        auto [ry, wy] = find(y);
        if (rx == ry) return false;

        parent[ry] = rx;
        weight[ry] = wx - wy + w;
        return true;
    }
};
```

## 22. Advanced Tree Patterns

Efficient algorithms on tree structures: LCA, rerooting, and tree diameter.

### Recognition

| When you see… | Pattern |
|---|---|
| Lowest common ancestor (LCA) queries, many of them | Binary lifting or sparse table |
| Subtree / path queries on a tree | DFS + Euler tour + segment tree |
| Rerooting (DP on all nodes as root) | DFS down + DFS up pass |
| Heavy-light decomposition | Path decomposition + segment trees on chains |
| Tree centroid / center | Find the node with min max subtree size |
| Find diameter of a tree | DFS twice: farthest from any node, then farthest from that |

### Example Algorithms

**LCA — Binary Lifting**
```cpp
const int LOG = 20;

vector<int> adj[100005];
int up[100005][LOG];
int depth[100005];

void dfs(int u, int p) {
    up[u][0] = p;
    for (int i = 1; i < LOG; i++) {
        up[u][i] = up[up[u][i-1]][i-1];
    }
    for (int v : adj[u]) {
        if (v != p) {
            depth[v] = depth[u] + 1;
            dfs(v, u);
        }
    }
}

int lca(int u, int v) {
    if (depth[u] < depth[v]) swap(u, v);

    int diff = depth[u] - depth[v];
    for (int i = 0; i < LOG; i++) {
        if ((diff >> i) & 1) u = up[u][i];
    }

    if (u == v) return u;

    for (int i = LOG - 1; i >= 0; i--) {
        if (up[u][i] != up[v][i]) {
            u = up[u][i];
            v = up[v][i];
        }
    }
    return up[u][0];
}
```

**Tree Diameter**
```cpp
vector<int> adj[100005];
pair<int, int> farthest;

void dfs(int u, int p, int dist) {
    if (dist > farthest.second) {
        farthest = {u, dist};
    }
    for (int v : adj[u]) {
        if (v != p) dfs(v, u, dist + 1);
    }
}

int treeDiameter(int n) {
    farthest = {0, 0};
    dfs(0, -1, 0);

    int first_end = farthest.first;

    farthest = {0, 0};
    dfs(first_end, -1, 0);

    return farthest.second;
}
```

**Rerooting DP — Sum of Distances in Tree**
```cpp
vector<int> adj[100005];
long long subtreeSum[100005];
int subtreeCount[100005];
long long ans[100005];
int n;

void dfs1(int u, int p) {
    subtreeSum[u] = 0;
    subtreeCount[u] = 1;
    for (int v : adj[u]) {
        if (v != p) {
            dfs1(v, u);
            subtreeSum[u] += subtreeSum[v] + subtreeCount[v];
            subtreeCount[u] += subtreeCount[v];
        }
    }
}

void dfs2(int u, int p) {
    for (int v : adj[u]) {
        if (v != p) {
            ans[v] = ans[u] - subtreeCount[v] + (n - subtreeCount[v]);
            dfs2(v, u);
        }
    }
}

long long sumOfDistances(int nodes) {
    n = nodes;
    dfs1(0, -1);
    ans[0] = subtreeSum[0];
    dfs2(0, -1);
    return ans[0];
}
```

## Master Recognition Index

Cross-topic quick lookup — the fastest way in when you're not sure which section to open.

| When you see… | Pattern |
|---|---|
| Contiguous subarray/substring | Sliding Window |
| Sorted array | Two Pointer / Binary Search |
| Kth largest/smallest | Heap or Quickselect |
| All subsets/permutations/combinations | Recursion / Backtracking |
| Minimum/maximum ways to… | Dynamic Programming |
| Next greater/smaller element | Monotonic Stack |
| Shortest path | BFS (unweighted) / Dijkstra (weighted) |
| Connected components / grouping | DFS/BFS / Union–Find |
| Prefix matching | Trie |
| Range sum queries | Prefix Sum / Segment Tree / Fenwick Tree |
| Values in range [1, n], O(1) space | Cyclic Sort / Index Marking |
| Top K / frequency | Heap or Bucket Sort |
| Overlapping intervals | Sort + Sweep |
| Two sequences comparison | DP (LCS family) |
| Minimize the max / maximize the min | Binary Search on the Answer |
| Tree path / ancestor | DFS with return-value propagation |
| Circular array | Modulo indexing, or double the array |
| In-place O(1) space array trick | Two-pointer / Cyclic sort / index-as-marker |
| Bitwise operations / subset encoding | Bit Manipulation |
| Range queries with updates | Segment Tree or Fenwick Tree |
| Pattern matching / substring search | KMP, Z-algorithm, or Rabin–Karp |
| GCD / LCM / primes / modular math | Math & Number Theory |
| Local optimization / non-overlapping selection | Greedy Patterns |
| Counting with digit constraints / state-based DP | Advanced DP (Digit, Tree, Bitmask) |
| Minimum spanning tree / bipartite checking | Advanced Union-Find |
| LCA / tree path queries / rerooting | Advanced Tree Patterns |

**When stuck, ask three questions:**
1. Is there a monotonic/sorted property? → binary search
2. Do choices repeat with overlapping subproblems? → DP
3. Do I need *all* ways rather than the *best* way? → backtracking