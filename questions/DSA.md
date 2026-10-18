## DSA

𝗗𝗦𝗔
1. Implement LRU/LFU Cache.
2. Find Median from Data Stream.
3. Word Ladder problem (shortest transformation).
4. Merge K Sorted Lists.
5. Detect Cycle in a Directed Graph.
6. Maximum Subarray Sum (Kadane + variations).
7. Kth Largest Element in a Stream.
8. Design a Scheduler with Task Priorities.
9. Two Sum variations (hashmap, sorted array, k-sum).
10. Binary Search on Answer (capacity, days, weights).
11. Longest Increasing Subsequence (DP + patience sorting).
12. Clone Graph with random pointers.
13. Serialize and Deserialize a Binary Tree.
14. Trapping Rain Water (stack + two pointer).
15. Top K Frequent Elements (heap + bucket sort).
16. Word Break Problem (DP + trie).

## SCENARIOS BASED QUESTION

1. "A gaming platform has 1 million users. During registration, check if a username is taken instantly."
- HashSet. O(1) lookup. Follow-up: "Memory concern?" → Bloom Filter. Probabilistic but uses 10x less memory.

2. "Build a cache with max 10,000 items. Evict the least recently used. All operations must be O(1)."
- HashMap + Doubly Linked List. HashMap for O(1) lookup. Linked list for O(1) recency tracking. Every access moves the node to the head. Eviction removes the tail.

3. "A stock trading app receives thousands of price updates per second. Show the top 10 most expensive stocks at any moment."
- Min-Heap of size 10. Why min and not max? Because you need to quickly remove the SMALLEST of the top 10 when a bigger price arrives. New price > heap min? Remove min, insert new. O(log 10) ≈ O(1).

4. "Implement browser history with back and forward navigation."
- Two Stacks. Stack 1 is for the back. Stack 2 for forward. Navigate to the new page? Push current to Stack 1 and clear Stack 2.

5. "Implement a queue using stacks."
- Two Stacks. Push to stack 1. For dequeueing—if stack 2 is empty, pour stack 1 into stack 2. Pop from stack 2. Amortized O(1).
