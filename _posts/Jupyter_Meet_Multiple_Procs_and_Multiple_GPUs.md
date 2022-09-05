# The Jungle where Jupyter x Multiple Processes x Multiple GPUs

I started out exploring “how to do distributed training of FastAI/PyTorch in Jupyter?”.

I learned a few technical quirks of getting multiprocessing x Jupyter x multiple GPUs to work together for distributed training. Tumbling down this rabbit trail, I first played with ipyparallel, the clunkiness irks me.

Refining the scope and needs, on second trial I put together a generic facility to execute a function on a group of distributed processes within a Jupyter notebook, that requires almost no change to existing code, can pass objects and results without extra steps like push/pull/get/set, is multi-GPU friendly, and lets user manage the execution environment via context manager.


### What Didn’t Work

It would be nice to interleave the highly educational and interactive FastAi notebooks with the speedup of distributed training in the same Jupyter session, but it wasn’t supported. 

### ipyparallel, the Obvious Choice?

I tried ipyparallel, and packed the Torch’s DDP group admin steps into a couple of functions, and built Ddip --- an iPython extension that manages the setup of and execution on a DDP group in a ipyparallel cluster.  But there are several drawbacks with this approach:

Slow cluster startup is slow, often 7 to 10 seconds even on a single host.
Extra gymnastics to maintain cluster state (start/stop/restart via % commands).
Passing objects between the cluster and the Jupyter requires explicit push/pull, not to mention not all types of objects are serializable.
One can choose to continue to manipulate the model directly on the cluster --- but then one must either ensure an operation is multiprocess-safe to be run distributedly, or remember to only execute this cell on one particular process in the cluster.

In short: fragile, complicated, and clunky to manage, especially in a Jupyter notebook with many cells that don’t need to be run distributedly. 

Can’t we simply pass a function and parameters to Python’s multiprocessing.Process() ?


#### With the insight from Ddip/ipyparallel experiment, I refine the needs to be:

Minimizing change to existing code, in order to run a function distributedly.
Allow seamless interleaving of local and distributed calls without complicated % magics.
Waiting for the distributed function to complete (blocking call) is acceptable, notebook is typically sequential anyway.
Can the caller in Jupyter receive the result automatically when the distributed function returns?
Simply wanna to run a function on N processes, prefer not to manage a cluster.
Each process may need to access its own GPU.

#### To address the above, I settled on the following process model and usage semantics:

Allow the Jupyter process itself to be one of the workers, then from its own target function execution, it can receive the return value directly, no extra pull/get.  It can gather, manipulate, aggregate results from other workers and return that.
Each process will know the overall pool size, and its rank in the pool, to support DDP.
Use a “spawn → execute → terminate” model, no cluster maintenance, resources will be freed upon exit. The re-spawning cost is acceptable in the big picture, especially when the distributed function may take a while to run.
Create an API similar to multiprocessing.Process(), with extra parameters to allow customized behavior.

#### To implement these, I discovered a few Python and CUDA quirks:

1. A CUDA GPU context is per process, it must not be shared with another process:
Thus when calling multiprocessing.Process(),  one must set method to ‘spawn’, not ‘fork’, because the parent Jupyter session may have touched the GPU.  The ‘forkserver’ method may not be a good idea either, because it can lead to multiple contexts on a GPU card (600MB each).

2. Passing in notebook defined functions and objects, use multiprocess, not  multiprocessing , because:

“(passing) any class methods, lambdas, or functions defined in __main__ wont’ work.  “Pickling” simply can’t handle a lot of different types of Python objects.”
 -- from “https://hpc-carpentry.github.io/hpc-python/06-parallel/”

Luckily, there is a fork of multiprocessing known as multiprocess that solves it!  
With that capability, I wrote the API to accept a string of names of functions, classes, or objects defined in the notebook.  They will be serialized, and imported into the child process’ namespace as those names,the distributed function will see them without any code change.

3. On Library imports: they must go into the ‘__main__’ namespace in child process:
This is achieved using the built-in ‘__import__’ gymnastics.  Users only need to pass in a multi-line string of “import statements” to my API.

#### Putting together the pieces, the second iteration became mpify (https://github.com/philtrade/Mpify).

‘mpify.ranch()’ lets user specify a function, its arguments, any local functions/objects needed, and “import” statements. User can gather results from any one or all workers, and manage the execution environment via context manager.  Mpify includes a context manager to set-up/tear-down PyTorch’s DDP, along with a convenience wrapper that uses it.

I ported several fastai v2 notebooks to be DDP-enabled, the amount of code change is quite minimal and mechanical.  All the interactive niceties are retained.

### More than One Way, Always.

Python has many quirks when it comes to multiprocessing, interactive Jupyter, and multiple GPUs, but manageable with carefully scoped requirements and some research.  And thanks to the open source ecosystem, mpify wouldn’t be possible without such the multiprocess fork of the standard Python library multiprocessing.

The open source and python ecosystems empower pursuits along any imaginable dimension and constraints, ipyparallel for functionally, and now Ray for performance and scalability.
The only limit is the time available to navigate, choose, and learn the myriads of offerings.

