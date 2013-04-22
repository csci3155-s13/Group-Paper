The syntax of this proposed change can be summarized by the following:  

~~~~~~~~~~~~~~  
spawning-expression:  
  cilk_spawn function-or-functor ( expression-list )  
sync-statement:  
  cilk_sync ;  
parallel-loop:  
  cilk_for ( init-clause ; condition-expr ; increment-expr ) statement  
~~~~~~~~~~~~~~  
In the above, expression-list is optional for cilk_spawn. Likewise, init-clause is optional for cilk_for.

Usage of cilk_spawn is relatively straightforward. The keyword cilk_spawn must be followed by a function, functor, or lambda expression. That function, functor, or lamda expression is referred to as the spawned function and the function that contains cilk_spawn is the spawning function. A full expression never contains more than one cilk_spawn.  
  
Assuming a given parallel program is deterministic, its parts are referred to in terms of its serial form, or serialization. To arrive at the serialization of a parallel program implemented with cilk_spawn, cilk_sync, and cilk_for, one can simply remove cilk_spawn and cilk_sync and change cilk_for to a garden-variety for loop. Strands in the program are referred to with respect to their serial order, using the terms earlier, later, earliest, and latest. This concept is called the serialization rule.  
  
In order to define the behavior of the three cilk functions, the concept of a task block must be defined. A task block is the body of a function, the body of a cilk_for loop, or a try block. A task block is considered complete when all of its children have finished executing and have synced using cilk_sync. One should not enter or exit a task block using goto, break, or return.  
  
Cilk_spawn gives permission for execution to proceed asynchronously. Control flow forks at the spawn point, which is the name given to the place where cilk_spawn is called. The spawned function’s arguments are evaluated before the spawn point. It is possible that undefined behavior may arise due to data races as a result of this parallelism.  
  
Cilk_sync demands that all children of the current task block complete their own execution before execution of the task block itself may resume.  
  
--------------
Sections 3 & 4. Edit as needed since I'm over 450 words.

The following example shows a parallel tree walk in which a computation f() is performed on the value of each node in a binary tree, yielding an integer metric. The results of the computation are summed over the entire tree:
~~~~~~~~~~~~~~
int tree_walk(node *n)
{
  int a = 0, b = 0;
  if (n->left)
    a = cilk_spawn tree_walk(n->left);
  if (n->right)
    b = cilk_spawn tree_walk(n->right);
  int c = f(n->value);
  cilk_sync;
  return a + b + c;
}
~~~~~~~~~~~~~~

Here, cilk_spawn indicates to the compiler that execution can proceed asynchronously to the next statement, without waiting for the recursive tree_walk calls to complete. A cilk_spawn defines a task – a piece of work that is permitted to execute asynchronously with the caller and with other called tasks.
When we need results from the spawned functions, cilk_sync is used to indicate a wait until all cilk_spawn subexpressions are complete. If cilk_sync is not explicitly defined, one is inserted at the end of the function. If tree_walk is deterministic, the meaning of the program is unchanged if Cilk keywords are removed (making it serialized). Easy serialization is one of the benefits of this approach since the work is easy for the programmers.

Fork-join parallelism refers to a way of specifying parallel executions of a program where the flow forks into multiple "flows" that rejoin when all parallel work is complete. When a fork is reached, the original strand ends and two new ones begin, which MAY run parallel to one another. At the join points, one or more strands terminate and the new strand continues, as can be seen in the graphs.

Adding the property of "strictness" to makes our usual fork-join parallelism model require that each function has exactly one incoming and one outgoing strand since asynchronous cilk_spawn function calls strictly nest within each other. Strict fork-join execution models have the following properties:
  - A task can fork off one or more child tasks, each of which may execute in parallel with each other and the parent task
  - A task can wait for its children to complete, but CANNOT wait for another non-child task to complete
  - A task cannot complete until all children are complete

The benefits of strictness in this model can be compared to that of using local variables and argument passing instead of global variables: decisions can be localized and individual parts of a program can be reasoned about without needing to understand the entire program at once. Thus, strict fork-join parallelism is very useful. Although it's obviously not perfect for all parallelized needs, it has the following benefits over less-organized parallel constructs:
- Serial semantics: every program can be executed serially, since a serial execution is always a legal interpretation of the parallel program.
- Composable: Parallel computations are allowed to be arbitrarily nested without taking more memory/runtime
- Modular: Parallel functions made such that the caller doesn't need to know whether a function executes in parallel. Asynchronous tasks don't "leak", preventing data races and other issues. It should be noted that task leaks are worse than memory leaks since one task leak can crash a program.
- Exceptions: Exceptions thrown from an asynchronous task will be caught at same place as if they were serialized in the program.
- Easy analysis: Strict fork-join parallelism allows various tools to have lower memory bounds to analyze.


Part 6 - 237 words

  The C++ programming language already provides many features to support multithreading. Features such as threads, mutexes, futures and async tasks allow powerful multitasking. However, even with all these features C++ still falls short in multithreading for a few reasons. The main problem is that creating and removing threads is a ‘heavy-weight’ operation. The C++ standard makes many promises for these threads that are high cost even if the code running in that thread does not require such rules. Both async and future limit their efficiency by making guarantees that slow down their potential implementations. Another problem is that threads and mutexes are not composable which adds complexities when making multithreaded programs.
	Intel argues that the fundamental problem with the existing multithreading capabilities is that they are specified in terms of threads. They argue that the programmer should be able to view the computation as a whole; not as different operations happening on different threads as it currently is. The programmer shouldn’t have to tell the computer what will be occurring on each thread, rather the computer should be able to decide what parts will be executed separately from the primary thread. Intel claims that by hiding thread interactions within the scheduler there would be a large increase in flexibility and possibly a better management of resources that could be scaled based on system resources. This form of parallel processing would also support composability without exhausting system resources.
