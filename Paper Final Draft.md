Strict Fork-Join Parallelism
============================
By Dylan Miller, Derek Baumgartner, and Connor Guerrieri

Fork-join parallelism refers to a way of specifying parallel executions of a program where the flow forks into multiple "flows" that rejoin when all parallel work is complete. When a fork is reached, the original strand ends and two new ones begin, which *may* run parallel to one another. At the join points, one or more strands terminate and the new strand continues.

Adding the property of "strictness" makes our usual fork-join parallelism model require that each function has exactly one incoming and one outgoing strand since asynchronous cilk_spawn function calls strictly nest within each other. Strict fork-join execution models have the following properties. One or more child tasks can be forked off, each executing in parallel with each other and the parent task. A task may wait for its children to complete, but not for another non-child task to complete. A task cannot complete until all children are complete

The benefits of strictness in this model can be compared to that of using local variables and argument passing instead of global variables: decisions can be localized and individual parts of a program can be reasoned about without needing to understand the entire program at once. Thus, strict fork-join parallelism is very useful. Although it is obviously not perfect for all parallelized needs, it has the following benefits over less-organized parallel constructs. Every program can be executed serially, since a serial execution is always a legal interpretation of the parallel program. Parallel computations are allowed to be arbitrarily nested without taking more memory or runtime. Parallel functions made such that the caller does not need to know whether a function executes in parallel. Asynchronous tasks do not "leak", preventing data races and other issues. It should be noted that task leaks are worse than memory leaks since one task leak can crash a program. Exceptions thrown from an asynchronous task will be caught at same place as if they were serialized in the program. Finally, strict fork-join parallelism allows various tools to have lower memory bounds to analyze.

Intel's Pablo Halpern, in his proposal to the C++ Standards Committee, has outlined a C++ language extension that implements strict fork-join parallelism. The syntax of this proposed change can be summarized by the following:  

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
  
Assuming a given parallel program is deterministic, its parts are referred to in terms of its serial form, or serialization. To arrive at the serialization of a parallel program implemented with cilk_spawn, cilk_sync, and cilk_for, one can simply remove cilk_spawn and cilk_sync and change cilk_for to a garden-variety for loop. Strands in the program are referred to with respect to their serial order, using the terms earlier, later, earliest, and latest. This concept is called the serialization rule because all programs implemented with this form of parallelism must be serializable.
  
In order to define the behavior of the three cilk functions, the concept of a task block must be defined. A task block is the body of a function, the body of a cilk_for loop, or a try block. A task block is considered complete when all of its children have finished executing and have synced using cilk_sync. One should not enter or exit a task block using goto, break, or return.  
  
cilk_spawn gives permission for execution to proceed asynchronously. Control flow forks at the spawn point, which is the name given to the place where cilk_spawn is called. The spawned function’s arguments are evaluated before the spawn point. It is possible that undefined behavior may arise due to data races as a result of this parallelism. cilk_sync demands that all children of the current task block complete their own execution before execution of the task block itself may resume. cilk_for allows loop iterations to run in parallel. The catch is that the loop body cannot modify the control variable or a nonlocal variable, as this would cause data race issues.

The below example shows a tree walk, implemented in parallel fashion, in which a computation is performed on the value of each node in a binary tree, returning an integer. The results of this computation are summed over the entire tree:
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
In this example, cilk_spawn permits execution to continue asynchronously without waiting for all the recursive calls to tree_walk to complete. cilk_spawn defines a share of work, called a task, that can execute in parallel with the caller and with other called tasks. cilk_sync then forces execution to wait for the cilk_spawn subexpressions to complete so that the results can be worked on. If cilk_sync is not explicitly defined, one is inserted automatically at the end of the function. tree_walk is deterministic, so the meaning of the program is unchanged if our cilk keywords are removed (termed serialization). Easy serialization is one of the benefits of this approach since the concept is easy for programmers to grasp.

The C++ programming language already provides many features to support multithreading. Features such as threads, mutexes, futures and async tasks allow powerful multitasking. However, even with all these features C++ still falls short in multithreading for a few reasons. The main problem is that creating and removing threads is a ‘heavy-weight’ operation. The C++ standard makes many promises for these threads that are high cost even if the code running in that thread does not require such rules. Both async and future limit their efficiency by making guarantees that slow down their potential implementations. Another problem is that threads and mutexes are not composable which adds complexities when making multithreaded programs.
	
Intel argues that the fundamental problem with the existing multithreading capabilities is that they are specified in terms of threads. They argue that the programmer should be able to view the computation as a whole; not as different operations happening on different threads as it currently is. The programmer shouldn’t have to tell the computer what will be occurring on each thread, rather the computer should be able to decide what parts will be executed separately from the primary thread. Intel claims that by hiding thread interactions within the scheduler there would be a large increase in flexibility and possibly a better management of resources that could be scaled based on system resources. This form of parallel processing would also support composability without exhausting system resources.

Community response to this proposal was noted as quite positive, apparently because strict fork-join parallelism is viewed by many as a desirable addition to the language. However, it was proposed that these additions take place within a pure library interface. Pablo Halpern has written another paper since the publication of the proposal described above in which he explores the possibility of a parallelism library. His conclusion was that due to myriad small technical issues, this would not be an appropriate solution. He writes:
> Engineers trying to write parallel software should be able to focus their attention on distributing work efficiently in parallel and avoiding races, not on circumventing dark corners of the language or library.

Even though the potential issues with the library implementation, such as exception scope problems and object lifetime troubles, could probably be fixed via band-aid workarounds, these fixes would violate this principle outlined by Halpern by creating such "dark corners" in the language. Therefore, he continues to support strict fork-join parallelism as a language extension, rather than a library.

Works Cited:
* [Pablo Halpern's Initial Proposal]
* [Considering a Fork-Join Parallelism Library]
* [Wikipedia]
[Pablo Halpern's Initial Proposal]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2013/n3557.pdf
[Considering a Fork-Join Parallelism Library]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3409.pdf
[Wikipedia]: http://en.wikipedia.org/wiki/Parallel_computing
