# DiffLens — Myers O(ND) Diff Visualizer

> **Project 12 — Version Control Algorithms**  
> Reference: Myers, E.W. (1986). *"An O(ND) Difference Algorithm and Its Variations"*. Algorithmica.

---

## Overview

DiffLens is an interactive web application that implements the **Myers O(ND) difference algorithm** — the same algorithm powering `git diff`, GNU `diff`, and most modern version control systems. It lets users compare two text files, see a color-coded diff, copy unified diff output, and **watch the algorithm explore the edit graph step by step**.

---

## The Algorithm

### From LCS to Edit Graph

The naïve approach frames text diffing as the **Longest Common Subsequence (LCS)** problem and solves it with a 2D DP table of size `O(m × n)`. For two 10,000-line files this requires ~100 million cells — impractical.

Myers reframes the problem as finding the **shortest path through an edit graph**:

- Each node `(x, y)` represents having consumed `x` lines from the original and `y` lines from the modified file.
- A **horizontal edge** `(x,y) → (x+1,y)` means *delete* line `x` from the original.
- A **vertical edge** `(x,y) → (x,y+1)` means *insert* line `y` from the modified file.
- A **diagonal edge** `(x,y) → (x+1,y+1)` (free) means *match* — both lines are equal.

The shortest edit script = path from `(0,0)` to `(n,m)` with minimum non-diagonal (edit) moves.

### The Diagonal Key Insight

Myers introduces **diagonal index k = x − y**. Every path of edit distance `D` stays within diagonals `−D ≤ k ≤ D`, advancing only 2 diagonals per unit of D. This allows the algorithm to:

1. For `d = 0, 1, 2, …`: explore only diagonals `k ∈ {−d, −d+2, …, d}`.
2. On each diagonal, greedily extend as far as possible along matches (**snakes**).
3. Stop when `(n, m)` is reached.

```
V[k] = furthest x reached on diagonal k

for d = 0 to N:
    for k = -d to d step 2:
        x = choose best predecessor (delete or insert)
        y = x - k
        while x < n and y < m and A[x] == B[y]:   # snake
            x++; y++
        V[k] = x
        if x >= n and y >= m: return d  # done
```

### Space Optimization: O(N) instead of O(m×n)

The standard DP LCS requires an `O(m × n)` matrix. Myers' algorithm only needs:

- `V[k]` — one array of size `O(N)` where `N = m + n`
- A snapshot of `V` per D-level for backtracking (also `O(N)` per level, but only `D` levels ≤ `N`)

This implementation uses **offset indexing** (`k + offset`) so negative diagonals map to valid array indices without extra allocation.

### Complexity

| | This implementation | Naïve LCS DP |
|---|---|---|
| **Time** | O(ND) | O(mn) |
| **Space** | O(N) | O(mn) |
| **Edit distance D** | insertions + deletions | — |
| **N** | \|original\| + \|modified\| | — |

---

## Project Structure

```
diff-tool/
├── myers.py          # Core algorithm — zero dependencies
├── app.py            # Flask REST API (POST /api/diff)
├── static/
│   └── index.html    # Full single-page web UI
├── requirements.txt
└── README.md
```

---

## Running Locally

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Start the server
python app.py

# 3. Open in browser
open http://localhost:5000
```

---

## Features

| Feature | Description |
|---|---|
| **Split diff view** | Side-by-side original vs modified, line numbers, color coding |
| **Unified diff** | Standard `git diff` format, copyable to clipboard |
| **Algorithm trace** | Interactive step-by-step visualization of the edit graph |
| **Play / Pause / Step** | Control animation speed, go backward and forward |
| **Drag & drop** | Load any plain text file directly |
| **Stats bar** | Edit distance, similarity %, lines added/removed |

---

## Edge Cases Handled

- Empty files (both, one, or neither)
- Completely identical files (D = 0, no diff)
- Completely different files (maximum edit distance)
- Files with only whitespace differences
- Unicode characters
- Files with trailing newline differences

---

## References

1. Myers, E.W. (1986). *An O(ND) Difference Algorithm and Its Variations*. Algorithmica, 1(1-4), 251–266.
2. Fowler, J. (2014). *The Myers diff algorithm: part 1*. jfowler.com
3. GNU diffutils source: `lib/diffseq.h`
