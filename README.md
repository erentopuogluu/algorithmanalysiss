**# DiffViz — Myers O(ND) Difference Algorithm

![Course](https://img.shields.io/badge/Course-Algorithm%20Analysis-60a5fa?style=flat-square)
![Algorithm](https://img.shields.io/badge/Algorithm-Myers%201986-4ade80?style=flat-square)
![Time](https://img.shields.io/badge/Time-O(ND)-fbbf24?style=flat-square)
![Space](https://img.shields.io/badge/Space-O(N)-c084fc?style=flat-square)
![Deploy](https://img.shields.io/badge/Live-GitHub%20Pages-2dd4bf?style=flat-square)

> Myers, E.W. (1986). *An O(ND) Difference Algorithm and Its Variations.* Algorithmica, 1(1–4), 251–266.

**Author:** Eren Topuoglu  
**Live demo:** https://erentopuogluu.github.io/algorithmanalysiss

---

## What this is

For the Algorithm Analysis final I had to implement a paper and build something around it. I picked the Myers diff paper because it's actually used everywhere — git diff, VS Code, GNU diff all run on this algorithm (or a variant of it). Seemed more interesting than doing another sorting algorithm.

The result is a single HTML file that:
- takes two text files and computes the minimal diff using Myers' algorithm
- shows the output in standard unified diff format (same as `git diff`)
- lets you watch the algorithm explore the edit graph step by step
- includes the theory, proofs, and complexity analysis

No server, no npm, no dependencies. Just open `index.html` in a browser.

---

## The algorithm — how it actually works

### Why not just use LCS?

The obvious approach is Longest Common Subsequence: find the longest sequence of lines that both files share, then everything else is an insertion or deletion. The standard DP solution needs an $O(m \times n)$ table. For two 1000-line files that's a million cells. For the Linux kernel (70k+ files) it'd need gigabytes just for the tables. Not usable.

Myers reframes the whole thing as a pathfinding problem, which is the clever part.

### The edit graph

Build a grid with $(m+1)$ columns and $(n+1)$ rows. Each point $(i, j)$ means "I've processed $i$ lines from file A and $j$ lines from file B." You can move three ways:

```
→  right:    (i,j) → (i+1, j  )   delete A[i]       cost = 1
↓  down:     (i,j) → (i,   j+1)   insert B[j]       cost = 1
↘ diagonal: (i,j) → (i+1, j+1)   A[i] == B[j]      cost = 0  ← "snake"
```

Finding the minimal diff = finding the cheapest path from $(0,0)$ to $(m,n)$. Diagonal moves are free because matching lines don't count as edits. The edit distance $D$ is just how many paid moves the shortest path needs.

The key formula connecting this back to LCS:

$$D = |A| + |B| - 2 \cdot \text{LCS}(A, B)$$

So if $D$ is small (files are similar), the LCS is large and the algorithm finds the answer quickly.

### The V[k] trick

This is what makes Myers fast. He defines a diagonal index $k = x - y$. Every point on the same diagonal shares the same $k$ value. Instead of storing the whole grid, you only need to track the furthest $x$ you've reached on each diagonal — that's $V[k]$.

So instead of an $O(mn)$ table you have a single array of size $2N+1$. The algorithm expands outward for $d = 0, 1, 2, \ldots$ and at each step only checks diagonals $k \in \{-d, -d+2, \ldots, d\}$. The first time it hits $(m,n)$, $D$ is found.

```python
def myers_forward(A, B):
    m, n = len(A), len(B)
    V = {1: 0}    # V[k] = furthest x reached on diagonal k
    trace = []    # save each step so we can backtrack later

    for d in range(0, m + n + 1):
        trace.append(dict(V))
        for k in range(-d, d + 1, 2):
            # came from above (insert) or left (delete)?
            if k == -d or (k != d and V[k - 1] < V[k + 1]):
                x = V[k + 1]        # vertical move = insert
            else:
                x = V[k - 1] + 1    # horizontal move = delete
            y = x - k

            # follow free diagonal moves (matching lines)
            while x < m and y < n and A[x] == B[y]:
                x += 1
                y += 1

            V[k] = x
            if x >= m and y >= n:
                return backtrack(trace, A, B, d)
```

The backtrack phase walks the saved snapshots in reverse and rebuilds the full list of operations.

### Why the inner loop only checks every other diagonal

After $D$ moves, reachable diagonals satisfy $k \equiv D \pmod{2}$. A delete increments $k$ by $+1$, an insert decrements it by $-1$, a diagonal (snake) leaves $k$ unchanged. Starting at $k=0$, after $d$ deletes and $i$ inserts: $k = d - i = D - 2i$, so $k \equiv D \pmod{2}$ and $|k| \leq D$. Odd-$D$ and even-$D$ diagonals are mutually exclusive — so `range(-d, d+1, 2)` is exactly right, not an optimization. It's the only diagonals that are actually reachable.

---

## Complexity

### Time: $\Theta(ND)$

At step $d$, $(2d+1)$ diagonals are scanned. Snake steps total at most $m + n \leq 2N$ across the whole run (each element matches at most once). Total work:

$$\sum_{d=0}^{D}(2d+1) + O(N) = O(D^2) + O(N) = O(ND)$$

Since $D \leq N$ this is $O(ND)$. The tight bound $\Theta(ND)$ holds because the algorithm cannot terminate without examining all diagonals at each frontier.

**In practice:** A 500-line file with a 3-line bugfix has $D = 3$. Myers does ~1500 operations. Naive LCS does 250,000. That's the difference.

### Best case: $\Omega(N)$

When $D = 0$ (identical files), the algorithm still has to check every line to confirm they all match. So $\Omega(N)$ is the floor — nothing can beat this.

### Worst case: $O(N^2)$

When $D = N$ (completely different files), substituting gives $O(N \cdot N) = O(N^2)$. This basically never happens in real diffs — typical commits change a tiny fraction of the file.

### Space: from O(mn) to O(N) to O(min(m,n))

The $V[k]$ working array is always $2N+2$ elements regardless of input — $\Theta(N)$.

For backtracking I save one $V$ snapshot per frontier $d$, giving $O(D \cdot N)$ total trace storage. Myers describes a divide-and-conquer approach that gets this down to $O(N)$, but I went with the snapshot approach since it's much easier to visualize.

For the LCS *length* only (not the full path), Myers' Variation 1 gets down to $O(\min(m, n))$ using a two-row DP technique — only `prev[]` and `curr[]` are kept:

```python
def lcs_len(A, B):
    if len(A) < len(B):
        A, B = B, A            # B is always the shorter side
    m, n = len(A), len(B)
    prev = [0] * (n + 1)
    curr = [0] * (n + 1)

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if A[i - 1] == B[j - 1]:
                curr[j] = prev[j - 1] + 1
            else:
                curr[j] = max(prev[j], curr[j - 1])
        prev, curr = curr, [0] * (n + 1)   # rotate rows

    return prev[n]

# for N=1000: 2x1001 integers vs 1001x1001 = ~500x less memory
```

The catch: with only two rows you can't backtrack, so this variant only returns the length. That's why the main implementation uses $V[k]$ instead.

| Approach | Time | Space | Output |
|---|---|---|---|
| Naive LCS DP | $O(mn)$ | $O(mn)$ | LCS length + path |
| Two-row (Variation 1) | $O(mn)$ | $O(\min(m,n))$ | LCS length only |
| Myers $V[k]$ | $O(ND)$ | $O(N)$ | Full edit script |
| Myers + trace (this impl.) | $O(ND)$ | $O(D \cdot N)$ | Edit script + visualization |

---

## Architecture

Everything runs in the browser. No server, no build step.

```
┌─────────────────────────────────────────────────────────┐
│                     index.html                          │
│                                                         │
│  INPUT LAYER          MYERS CORE         OUTPUT LAYER   │
│  ─────────────        ──────────         ────────────   │
│  Line Parser    →     Forward Pass  →    Hunk Builder   │
│  (split on \n)        V[k] array         (3-line ctx)   │
│                                                         │
│                       Trace Store   →    Unified Diff   │
│                       (V snapshots)      Formatter      │
│                                          @@ -a,b +c,d   │
│                       Backtrack                         │
│                       (ops[] list)                      │
│                                                         │
│  VISUALIZATION LAYER                                    │
│  ────────────────────                                   │
│  HTML5 Canvas (Edit Graph, Architecture, Benchmark)     │
│  Step Controller (Play / Pause / Back / Next / Speed)   │
└─────────────────────────────────────────────────────────┘
```

**Input Layer:** splits textarea content into line arrays, handles empty files and trailing newlines

**Myers Core:** `myersDiff()` runs the forward pass and trace store; `backtrack()` reconstructs operations

**Output Layer:** `buildHunks()` groups changes with 3-line context; renders `@@ -a,b +c,d @@` headers

**Visualization Layer:** canvas redraws on every step; step controller manages play/pause/speed state

---

## Features

| Tab | What it does |
|---|---|
| Diff Tool | Paste two files, get unified diff with `@@` hunk headers |
| Step Tracer | Watch the algorithm run — play, pause, step forward/back, adjust speed |
| Theory & Proof | Edit graph model, V[k] pseudocode, complexity proofs |
| Architecture | System diagram with component descriptions |
| Complexity | Θ/Ω/O notation cards + live benchmark chart |

### Edge cases handled
- Both files empty
- One file empty
- Identical files ($D = 0$)
- Completely different files
- Files with only whitespace differences
- Unicode characters

---

## Running locally

Just open the file:

```bash
open index.html        # macOS
xdg-open index.html    # Linux
start index.html       # Windows
```

Or with a static server:

```bash
python -m http.server 8000
# open http://localhost:8000
```

---

## Project structure

```
algorithmanalysis/
├── index.html    # everything — HTML, CSS, JS in one file
└── README.md
```

---

## References

1. Myers, E.W. (1986). An O(ND) Difference Algorithm and Its Variations. *Algorithmica*, 1(1–4), 251–266.
2. Fowler, J. (2014). The Myers diff algorithm: part 1. jfowler.com
3. GNU diffutils source: `lib/diffseq.h`
4. Git source: `xdiff/xdiffi.c`

---

*Algorithm Analysis Final — Eren Topuoglu***
