# ChessEngineParallelization

Chess programming, a classic challenge in artificial intelligence, poses immense complexity due to the sheer number of possible moves. With an astronomical 10^120 potential configurations, surpassing atoms in the observable universe, the field has garnered significant attention. The Minimax algorithm facilitates chess gameplay by evaluating moves within a search tree. However, due to computational constraints, search depth is limited.

This notebook delves into enhancing an existing C-based chess engine through parallelization, enabling simultaneous computation of independent branches. The goal is to accelerate the decision-making process by exploring additional layers of the search tree. Rather than building a chess engine from scratch, our focus is on optimizing an already functional one. Through parallelization, we aim to contribute insights into improving computational performance in chess engines.

# Parallel Search Algorithms
## Root Splitting
The main idea of this technique is to ensure that each node except for the root, is visited by only one processor. To keep the effect of the alpha beta pruning, we split the children nodes of the root into clusters, each cluster being served by only one thread. Each child is then processed as the root of a sub search tree that will be visited in a sequential fashion, thus respecting the alpha beta constraints. When a thread finishes computing it’s subtree(s) and returns its evaluation to the root node, that evaluation is compared to the current best evaluation (that’s how minimax works), and that best evaluation may be updated. So to ensure coherent results, given that multiple threads may be returning to the root node at the same time, threads must do this part of the work in mutual exclusion: meaning that comparing to and updating the best evaluation of the root node must be inside a critical section so that it’s execution remains sequential. And that’s about everything about this algorithm!

We'll just illustrate the key (changed) parts of the original C code here.

We said that we parallelize at the root level, so for that we parallelize the for loop that iterates over the children of the root node for calling the minimax function on them. for that, we use the following two openmp directives:

#pragma omp parallel private (/*variables private to thread here */) #pragma omp for schedule (dynamic)

The two above directives need to be put right before the for loop to parallelize. In the first one we declared the variables that must be private to each thread. In the second directive we specified a dynamic scheduling between threads, meaning that if a thread finishes his assigned iterations before others do, he'll get assigned some iterations from another thread, that way making better use of the available threads.

Then we must ensure inside our for loop that after each minimax call returns, the thread enters in a critical section in mutual exclusion to compare (and modify) the best evaluation of the root node. We do that using the following directive:

#pragma omp critical { // Code here is executed in mutual exclusion }

All the code written inside this directive will be run by at most one thread at a time, ensuring thus the coherence of the value of our best evaluation variable.

## Beam Search
The beam search technique uses the move reordering to only explore the most promising nodes of the search tree; that is, after we reorder the children of a given node, we only explore a subset of those child nodes, by ignoring the least promising ones (according to the estimation function!), and thus making the search faster. Of course the trade-off of doing so is to risk not exploring parts of the search tree that might lead to better moves.

To implement this feature, we only need to make the slight change of iterating through n/2 child nodes instead of n in the minimax function. We chose to only consider the most promising half of the children nodes.
