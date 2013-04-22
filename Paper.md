Paper WIP
=========
  
*Paper topic:*  
**Strict Fork-Join Parallelism in C++**  

*Team Members for this paper will include:*  
1. **Derek Baumgartner**  
2. **Connor Guerrieri**  
  
*Expected sources:*  
1. Pablo Halpern's [proposal][]  
2. Wikipedia articles on [parallelism][]  
3. C++ [Standards Committee Papers][]  
4. Internet discussions of the proposal, if possible (not yet located) 

*Code examples:*  
(All code examples are sourced from the aforementioned [proposal][].)  
Parallel tree walking:  
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

*Current status of the paper:*  
Research into the topic of adding parallelism to C++ has begun. I have located a lengthy change proposal written by Pablo Halpern of Intel Corporation.  
Code examples have been located within the proposal. Group has been assembled.

[Standards Committee Papers]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/
[proposal]: http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2012/n3409.pdf
[parallelism]: http://en.wikipedia.org/wiki/Parallel_computing
