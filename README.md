# dijkstra algorithm: what it actually does, when it breaks, and how to remember the template under interview pressure

You type "dijkstra algorithm" into a search box for one of two reasons. Either you're in a data structures course and the pseudocode in your notes refuses to make sense, or you have an interview in three weeks and you keep writing the version that's wrong in a subtle way.

Both problems have the same root. Dijkstra's algorithm is short enough to memorize in five minutes, and almost everything interesting about it lives in the details people skip: why the greedy step is safe, what happens to stale priority queue entries, and which graphs break the whole thing. This is a walkthrough of those details, plus notes on the failure modes that actually show up in interviews and on practice platforms.

## What problem Dijkstra is solving

Given a graph with weighted edges and one starting vertex, find the shortest path from that source to every other reachable vertex. Not just to one destination — to all of them, at the same cost, which is the slightly surprising part. Edsger W. Dijkstra worked it out in 1956 and published it in 1959, and the single-source framing is why it survives in routing software, network protocols and game pathfinding decades later.

Two conditions have to hold:

- Every edge weight is non-negative. Zero is fine. Negative breaks it.
- You want the shortest path by accumulated weight, not by number of hops. For hop counts on unweighted graphs, plain BFS is simpler and faster.

Dijkstra keeps a running estimate of the distance to each vertex, starting at 0 for the source and infinity for everything else. On each iteration it pulls the unvisited vertex with the smallest known distance, marks it done, and relaxes its outgoing edges — meaning it checks whether reaching a neighbor through the vertex just finalized is cheaper than the neighbor's current estimate. If it is, the estimate gets updated and the neighbor goes back into the queue.

## Why the greedy choice is allowed

This is the part most tutorials state and don't justify, and it's the part interviewers poke at.

The loop always commits to the closest unsettled vertex. That looks reckless. It's safe because of non-negative weights: any alternative route to that vertex would have to pass through some other unsettled vertex, and every unsettled vertex is at least as far away. Adding non-negative edges to a path that's already longer can't produce something shorter. So the moment a vertex comes out of the queue as the minimum, its distance is final.

Take the guarantee away and the argument collapses. Add a single negative edge and a vertex you already finalized may later have a shorter path through it — the algorithm never looks back, and you get a wrong answer with no error message. That's the practical value of the "non-negative weights" line in every textbook.

## One run, step by step

Small graph, source A:

- A→B costs 1, A→C costs 4, B→C costs 2, B→D costs 5, C→D costs 1.

Distances start as A=0, B=∞, C=∞, D=∞.

1. Pop A (0). Relax: B becomes 1, C becomes 4. Queue holds (1,B) and (4,C).
2. Pop B (1). Relax: C through B is 1+2=3, better than 4, so C becomes 3. D becomes 1+5=6.
3. Pop C (3). Relax: D through C is 3+1=4, better than 6, so D becomes 4.
4. Pop D (4). Nothing left to improve.

Final: A=0, B=1, C=3, D=4. Every relaxation that improved a value also recorded where it came from, which is what lets you rebuild the actual path at the end instead of just the cost.

Here's the template, in the lazy-deletion form that most competitive programmers actually write:

python
import heapq

def dijkstra(graph, source):
    dist = {v: float("inf") for v in graph}
    dist[source] = 0
    prev = {}
    pq = [(0, source)]

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue  # stale entry, skip it
        for v, w in graph[u]:
            if d + w < dist[v]:
                dist[v] = d + w
                prev[v] = u
                heapq.heappush(pq, (dist[v], v))
    return dist, prev


Two lines carry most of the weight. The `if d > dist[u]: continue` check is the difference between a correct implementation and one that reprocesses vertices with outdated distances. And the fact that you push a new entry instead of updating the old one is deliberate: standard heap libraries don't support a decrease-key operation, so you let duplicates accumulate and ignore them on the way out. The textbook version with `incr_priority` and a Fibonacci heap is faster in theory and rarely worth writing by hand.

## The complexity numbers, and why the queue choice decides them

The runtime is entirely a story about which priority queue you used.

| Queue implementation | Extract-min cost | Total runtime |
| --- | --- | --- |
| Unsorted array / linear scan | O(V) | O(V²) |
| Binary heap | O(log V) | O((V + E) log V) |
| Fibonacci heap | O(log V) amortized, O(1) decrease-key | O(E + V log V) |

O(V²) is genuinely competitive on dense graphs where E is close to V², which is why the adjacency-matrix version still shows up in older textbooks. The binary heap version is the default answer. The Fibonacci heap bound is the one people quote because it looks better, and it's largely theoretical — the constant factors are bad enough that it's rarely used in practice.

Every vertex is extracted once, and every edge can trigger at most one insertion, which is where the O((V + E) log V) comes from. If you can explain that sentence out loud, you've covered the complexity question interviewers usually ask.

## Where Dijkstra breaks, and what to use instead

- **Negative edge weights.** The greedy commitment is invalid. Use Bellman-Ford, which handles negative edges and can detect negative cycles; it runs in O(V·E).
- **Unweighted graphs.** Skip the priority queue. BFS gives you shortest paths in O(V + E) with a plain queue.
- **Negative cycles.** No shortest path exists if you can loop forever and keep decreasing cost. Bellman-Ford tells you this happened; Dijkstra will just hand you confident nonsense.
- **Huge graphs, one known destination.** Plain Dijkstra explores outward in all directions. If you have a decent estimate of distance to the goal, A* adds that as a heuristic to the queue priority and explores far fewer vertices — same code, one extra term.

## Variants worth knowing about

A* is the one you should be able to describe even if you've never implemented it: instead of ordering the queue by distance from the source, you order by distance from the source plus an estimate of remaining distance to the target. With an admissible heuristic you still get an optimal path, and you usually visit a fraction of the vertices.

Bidirectional search runs two Dijkstras, one from the source and one from the destination, and meets in the middle. It's a real speedup in practice and a much rarer interview question.

Grid pathfinding is where most people meet the algorithm outside a classroom. Roads, game maps and robot navigation are all weighted graphs, and traffic-aware routing is often just Dijkstra on a graph where the edge costs change — one patent filing in this space describes running Dijkstra over a street network built from intersections and turn rules, exactly that pattern. Internet routing protocols like OSPF use the same core idea, which is why the 1959 paper still gets cited in network engineering.

## How this shows up in interviews

Dijkstra almost never arrives as "implement Dijkstra." It arrives disguised:

- Cheapest flights within K stops — Dijkstra with a state dimension, or Bellman-Ford if K is small.
- Network delay time — textbook single-source shortest path.
- Path with minimum effort or minimum probability — the same loop with max or product instead of sum.
- Swim in rising water, cheapest path in a grid — grid graph, priority queue, same skeleton.

The recognition step is the hard part, not the code. AlgoMonster's public decision flowchart makes this explicit: it asks whether you have a shortest-path problem, then whether the graph is weighted, and if both answers are yes it points at Dijkstra's algorithm. That's the whole identification heuristic in two questions.

Worth noting what AlgoMonster's own ROI analysis says about priorities: DFS, BFS and two pointers dominate interview problem sets, basic data structures have high return on study time, and priority queues and heaps — the data structure Dijkstra depends on — land in the middle tier. Greedy algorithms, which Dijkstra technically is, get rated low ROI because each one is different and correctness often needs a proof you can't improvise. Read that as a scheduling hint: Dijkstra is worth knowing cold, but it isn't the thing to spend a month on if your interviews are close.

## Learning it as a pattern instead of as a proof

The gap for most people isn't understanding Dijkstra. It's reproducing it correctly at 11pm in a timed assessment after a long day. That's a pattern-recognition problem, and it's the specific thing AlgoMonster builds its curriculum around — it's a structured course from ex-Google engineers that teaches reusable templates rather than a bank of problems to grind.

Their decision flowcharts cover the "which pattern is this" step, the explanation format is text and diagrams rather than video, and the practice problems are attached to the pattern that solves them rather than sitting in one undifferentiated list. For Dijkstra specifically, that means you meet it as "weighted graph, shortest path, distinct edge costs" instead of as problem #743.

Two caveats reviewers raise consistently, and they're fair. The problem volume is small compared to LeetCode — reviewers put it around 231 problems, versus thousands on LeetCode — so it teaches frameworks rather than fluency, and many candidates pair it with a problem bank for volume. And there are no video walkthroughs at all; if you learn by watching someone narrate a screen, the text-and-diagram format will feel dry.

## AlgoMonster plans and pricing

There's no monthly option and no traditional free trial. What's listed is a limited free tier plus two paid tiers, currently discounted from the list prices:

| Plan | What you get | Price | Billing | Get access |
| --- | --- | --- | --- | --- |
| Free | Limited access to a subset of patterns and lessons, enough to judge whether the teaching style works for you | $0 | — | [Browse AlgoMonster's free content](https://algo.monster?fpr=li69) |
| Pro — Yearly | All patterns, lessons and problems; unlimited AI assistant; unlimited Speedrun; real questions from top companies updated monthly; 500+ lessons across 6 courses, 1,000+ illustrations, 3,000+ editorials; bonus Foundation and Build an LLM courses; member discount on live bootcamps | $99/year (shown as $8.25/mo billed annually, down from a $198 list price) | 12 months, cancel or upgrade anytime | [Get AlgoMonster Pro yearly](https://algo.monster?fpr=li69) |
| Pro — Lifetime | Everything in the annual plan plus lifetime updates to included Pro content | $189 one-time (down from a $378 list price) | Billed once | [Get AlgoMonster Pro lifetime](https://algo.monster?fpr=li69) |

Three things to know before you decide:

1. Prices on this product move with promotions, and some blog posts still quote higher historical figures for the lifetime tier. Trust the checkout page, not a review from six months ago.
2. Some partner-developed courses, such as Quant Interview Prep, are sold separately and aren't part of either plan.
3. There's no team or enterprise plan — it's built for individual candidates.

The annual plan makes sense if you have interviews in the next couple of months and want the material done before then. Lifetime makes sense if you expect to change jobs every two or three years and don't want to subscribe again each time; at $189 it costs less than two years of the annual price, and it's the tier most reviewers point to as the better long-run value. If you're not sure yet, the free tier exists precisely so you can find out whether diagram-based explanations click for you before paying anything.

## Mistakes that show up again and again

- **Forgetting the stale-entry check.** Duplicates in the queue with outdated distances get processed as if they were current, which corrupts results on graphs with frequent updates. The `if d > dist[u]` line is not optional in the lazy version.
- **Running it on negative weights.** Sometimes this produces a correct answer anyway, which is worse than failing, because it teaches the wrong habit.
- **Skipping path reconstruction.** Many problems want the path, not the distance. Record predecessors during relaxation and walk them backward — the edges you saved along the way form a tree rooted at the source.
- **Reimplmenting the heap by hand** when the language library has one. In an interview, hand-rolling a binary heap costs you twenty minutes you probably needed for the actual problem logic.
- **Reaching for Dijkstra when BFS is enough.** If all edges cost the same, the priority queue buys you nothing but slower code and more places to make mistakes.

## The short version

Dijkstra answers one question: cheapest route from a single source to everything else, assuming no negative edges. It works because non-negative weights make the greedy commitment safe. Its runtime is whatever your priority queue's operations cost — O((V + E) log V) with a binary heap. It fails on negative weights, gets replaced by BFS on unweighted graphs and by A* when you know roughly where you're heading, and it shows up in interviews as a grid problem or a cheapest-path problem long before it shows up under its own name.

Learn the template until you can write it without the stale-entry bug, learn to recognize the two-question test — shortest path, weighted edges — and treat everything past that as the details it is.

👉 [Start with AlgoMonster's free tier and see the Dijkstra pattern mapped out](https://algo.monster?fpr=li69)
