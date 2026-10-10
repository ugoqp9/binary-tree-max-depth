# maximum depth of binary tree: the 5-line recursion, the BFS version, and why your answer keeps coming back off by one

LeetCode 104 is rated Easy, and it is. Which is exactly why people fumble it in interviews. Not because the recursion is hard, but because there are two things you can get wrong without noticing: the definition of depth, and the difference between the recursive answer and the one you'd actually want if the tree were a degenerate 10,000-node chain.

Here's the problem in full, the three solutions worth knowing, the complexity claims that are technically true and practically misleading, and where this problem sits inside the bigger pattern you're really being tested on.

## What LeetCode 104 is asking

Given the `root` of a binary tree, return its maximum depth. A binary tree's maximum depth is the number of nodes along the longest path from the root node down to the farthest leaf node.

- `root = [3,9,20,null,null,15,7]` → `3` (paths `3 → 20 → 15` and `3 → 20 → 7`)
- `root = [1,null,2]` → `2`
- `root = []` → `0`

Constraints: the number of nodes is in the range `[0, 10^4]`, and `-100 <= Node.val <= 100`. Nothing about the values matters here. They exist so you can't cheat by inspecting them, and they're a hint that the answer only depends on structure.

A leaf is a node with no children. That sentence is the whole base case in disguise: a leaf has depth 1, not 0.

## The definition trap: nodes or edges

This is the part that quietly costs points.

LeetCode counts **nodes** on the root-to-leaf path. The tree `[3,9,20,null,null,15,7]` has three nodes on its longest path, so the answer is 3.

Some sources count **edges** instead, which for that same tree gives 2. AlgoMonster's own lesson page for this problem describes depth as "the number of edges along the longest path from the root down to the farthest leaf node," while the worked example on that same page returns 3 for `[3,9,20,null,null,15,7]` — the node-count answer. A third-party write-up of the AlgoMonster version flags precisely this: their DFS lesson counts edges, so if you're solving the LeetCode version you'd need to return `dfs(root) - 1`.

Practically: read the definition on the page you're solving. If the expected output for a three-node path is 3, count nodes. If it's 2, count edges. Then write the base case once and never think about it again. The classic symptom of mixing the two up is a solution that's correct on `[1,null,2]` and wrong on everything else.

## Approach 1: recursive DFS (what most interviewers expect)

The tree is a recursive structure, and so is the problem. The depth of a node is 1 plus the deeper of its two subtrees. An empty subtree has depth 0.

python
class Solution:
    def maxDepth(self, root: Optional[TreeNode]) -> int:
        if not root:
            return 0
        return 1 + max(self.maxDepth(root.left), self.maxDepth(root.right))


Java, if that's your interview language:

java
class Solution {
    public int maxDepth(TreeNode root) {
        if (root == null) return 0;
        return 1 + Math.max(maxDepth(root.left), maxDepth(root.right));
    }
}


Note the traversal order. The recursive calls on the children have to finish before the parent can compute anything, so this is post-order traversal. That ordering question — do I need my children's results before I can answer, or do I pass something down? — is the actual decision you make in every tree problem, and this is the cleanest place to learn it.

Dry run on `[3,9,20,null,null,15,7]`:

- `maxDepth(9)` → both children null → `1 + max(0, 0)` = `1`
- `maxDepth(15)` → `1`, `maxDepth(7)` → `1`, so `maxDepth(20)` → `1 + max(1, 1)` = `2`
- `maxDepth(3)` → `1 + max(1, 2)` = `3`

Every node is visited exactly once, so time is O(n). Space is the call stack, which holds one frame per level: O(h), where h is the tree height. For a balanced tree that's O(log n). For a completely skewed tree it's O(n), and that's not a theoretical footnote — see the edge cases below.

## Approach 2: iterative DFS with an explicit stack

Same traversal, manual stack. Each entry carries the node and its depth, so the depth travels with the node instead of living in a call frame.

python
class Solution:
    def maxDepth(self, root: Optional[TreeNode]) -> int:
        if not root:
            return 0
        stack = [(root, 1)]
        best = 0
        while stack:
            node, depth = stack.pop()
            best = max(best, depth)
            if node.left:
                stack.append((node.left, depth + 1))
            if node.right:
                stack.append((node.right, depth + 1))
        return best


Time is still O(n). Space is O(n) in the worst case for the stack, but it comes out of the heap rather than the call stack, which matters in languages with a hard recursion limit.

## Approach 3: BFS, counting levels

If you're computing a level-based property, use the traversal that thinks in levels. Each pass over the queue drains exactly one level, so the number of passes is the depth.

python
from collections import deque

class Solution:
    def maxDepth(self, root: Optional[TreeNode]) -> int:
        if not root:
            return 0
        q = deque([root])
        depth = 0
        while q:
            for _ in range(len(q)):   # exactly one level
                node = q.popleft()
                if node.left:
                    q.append(node.left)
                if node.right:
                    q.append(node.right)
            depth += 1
        return depth


One Python-specific trap: use `deque`, not a list with `pop(0)`. Removing from the front of a Python list is O(n), which turns this O(n) solution into O(n²) and can time out on large inputs. The `for _ in range(len(q))` snapshot is also load-bearing — if you loop `while q` directly and check `len(q)` each iteration, you blend levels together and count nothing.

## Which one is actually better

| Approach | Time | Space | Use when |
| --- | --- | --- | --- |
| Recursive DFS | O(n) | O(h) — O(log n) balanced, O(n) skewed | Default answer. Short, readable, what interviewers expect |
| Iterative DFS (stack) | O(n) | O(n) worst case, on the heap | Tree can be very deep, or the language has a low recursion limit |
| BFS (queue) | O(n) | O(w), where w is the widest level — O(n) for a balanced tree, O(1) for a skewed one | You also need per-level information (right side view, zigzag, level averages) |

Two things worth saying out loud in an interview: BFS uses the most memory on the shape of tree where DFS is cheapest, and DFS uses the most memory on the shape where BFS is cheapest. For a balanced tree, the last level holds roughly n/2 nodes, so the BFS queue is O(n) while the recursion stack is O(log n). Flip it for a linked-list-shaped tree.

## Edge cases that actually cost people points

**Empty tree.** `root = []` returns 0. If your base case is inside the loop rather than at the top, this crashes.

**Single node.** `[1]` returns 1. The root is both root and leaf, and this is where edge-counting implementations tend to return 0.

**Skewed trees and Python's recursion limit.** The node count goes up to 10⁴. A left- or right-skewed tree of 10,000 nodes produces a recursion depth of 10,000, and CPython's default recursion limit is 1000. Your clean recursive solution raises `RecursionError` on a legal input. `sys.setrecursionlimit` fixes it locally, but the safer interview answer is to mention the tradeoff and offer the iterative version — that's usually the follow-up question anyway.

**Don't use depth as a sentinel.** Some solutions return `-1` for an empty subtree and add 1 on the way up. It produces the same number for this problem, but the moment you extend the code to a sibling problem like minimum depth, the sentinel breaks the logic.

## The pattern behind the problem

Maximum depth is the entry point to tree DFS, and the template generalizes with one change: what you return, and how you combine the two child results.

python
def dfs(node):
    if not node:
        return BASE_VALUE          # 0 for depth, True for existence, -inf for max path
    left = dfs(node.left)
    right = dfs(node.right)
    return COMBINE(left, right, node)


The same shape solves:

- **Minimum Depth of Binary Tree** — combine with `min`, and handle the single-child case explicitly, because a node with one child is not a leaf
- **Diameter of Binary Tree** — return height upward, but track `left + right` as a global answer
- **Balanced Binary Tree** — return height, propagate a failure flag
- **Path Sum** — pass the remaining target down instead of returning a depth up

If you're grinding randomly, these four feel like four unrelated problems. They're one template with different combine steps. Recognizing that is the skill interviewers are actually measuring, and it's the reason a 231-problem curriculum can beat 3,000 problems done in no particular order.

AlgoMonster is built around exactly this framing: each problem sits inside an explicit pattern with a template and a set of variations, and this one lives under the DFS-on-tree pattern. Their walkthrough for this problem breaks it into staged steps (read the problem, identify the pattern, then implement) rather than dropping a finished solution on you, and the platform keeps an AI assistant on hand that will point at the specific line you got wrong instead of only telling you the test failed. 👉 [See AlgoMonster's tree DFS pattern walkthrough and current plans](https://algo.monster?fpr=li69)

Worth knowing before you pay: the format is text and interactive diagrams, with no video walkthroughs, and the problem bank is in the low hundreds. If you learn from watching someone code, this will feel dry. If you learn by reading and then typing the template yourself, it's efficient. Third-party reviews also flag that system design coverage is introductory and behavioral prep is surface-level, so senior candidates will still need something else for those rounds.

## AlgoMonster plans and what they cost

AlgoMonster doesn't sell a monthly plan. The official subscribe page shows a free tier and two paid options, with the following details as listed there:

| Plan | What you get | Price | Billing cycle | Link |
| --- | --- | --- | --- | --- |
| Free | Limited access to sample patterns and problems, enough to test the teaching style | $0 | — | [Start on the free tier](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; company questions updated monthly; bonus Foundation and Build-an-LLM courses; 12 months of full access | $8.25/month, billed as $99/year | Annual, cancel or upgrade anytime | [Get Pro Yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime content updates, one payment | $189 | One-time | [Get Pro Lifetime](https://algo.monster?fpr=li69) |

A few notes on those numbers, because you'll see different ones elsewhere. Several review sites quote a lifetime price of $459 against a $918 list price, and one quotes a yearly range of $119–$299. Those pages are describing promotional states and list prices that may or may not be live at any given moment. The number that matters is the one on the checkout page when you buy, and the official subscribe page has been showing $99/year and $189 as discounted rates. Some coupon pages advertise an extra percentage-off code stacked on top of an active sale; I couldn't confirm any specific code against the official page, so treat those as worth trying at checkout rather than as facts.

The straightforward comparison: at $99/year, lifetime pays for itself in under two years. If you're prepping for one interview cycle and expect to finish in six to eight weeks, yearly is the cheaper commitment. If you're early in your career and expect to interview again for promotions or job changes, the lifetime price is roughly two annual renewals.

Also useful to know what a subscription does *not* include: the platform has no live mock interviews, no video, and no timer pressure. It builds pattern recognition; it doesn't simulate the 45 minutes with a senior engineer watching your screen. Reading about the tree DFS pattern and then defending a `1 + max(left, right)` return value under follow-up questions are different skills.

## Follow-up questions you can expect on this exact problem

**"Can you do it without recursion?"** Yes, either of the two versions above. Name the stack or queue explicitly.

**"What's the space complexity?"** Be precise: O(h) for the recursive version with h equal to tree height, not O(log n) universally. Intervviews often ask this because candidates say "O(log n)" by default and skip the skewed case.

**"What if the tree has a million nodes?"** Recursion depth is the constraint, not time. Iterative DFS or BFS removes the call-stack ceiling.

**"How would you return the actual path, not just the depth?"** Carry the path in the stack entries or return `(depth, node)` up the recursion. The complexity is unchanged; the bookkeeping is not.

**"Why is this rated Easy?"** Because the code is five lines and the idea is one sentence. The failure modes — node vs edge counting, the recursion limit on skewed trees, O(n²) from `pop(0)` — are the real content. That's true of a lot of Easy-rated tree problems, which is why they're worth solving properly instead of pattern-matching your way to a passing submission.

If you want to practice this one properly rather than just clear it, do it in this order: write the recursive version from scratch without looking, dry-run it on `[3,9,20,null,null,15,7]` on paper, then rewrite it iteratively, then solve Minimum Depth and Diameter with the same template. That sequence takes about an hour and covers more interview surface than twenty random problems. 👉 [Compare AlgoMonster's Yearly and Lifetime access](https://algo.monster?fpr=li69)
