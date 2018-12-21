#rust-cs

This is a learning project written in rust, to explore how to implement something, outside a larger project I have.

I thought it might be easier to develop it without getting distracted by the larger problem and amount of code in my "flow" project.
I also thought it might be useful for others to look at, improve upon or learn from.

## The problem
I want to have a client-server system (hence "-cs" in the name) for executing sets of jobs concurrently.

Tasks are structures with a reference to the code that implements them.
Tasks may accept inputs and produce outputs to be returned.
IOTasks may make use of STDIO, including blocking the running thread waiting for user input.

SW Components foreseen:
- standard library with implementations for tasks
- client library with the code required to submit tasks
- execution library to receive tasks for execution, execute them and return results to the server. Linked with the standard library.
- server library with code to handle reception of sets of tasks, executor registration, and the dispatching of tasks for execution

To make things more manageable, I have divided what I'd ideally like to do (if it's possible) into a number of phases:

### Phase 1 - single machine / single process / single thread
- Single binary is built linking: client library, execution library, server library and a combined "main"
- The app starts up. It starts a server thread and gets channels to be used to submit tasks sets to
- The client code (main thread) connects to the server thread as a submitter of sets of tasks for execution
- The client registers with the server thread as an executor of tasks, receiving a channel to receive tasks on and one to return results on
- The client submits a set of tasks to be run, to the server
- The client code starts the task execution loop, receiving tasks from the server over a channel, executes them, returning results to server thread on a channel
- The server distributes tasks to executors that it knows about (only one at this point) over channels
- The server distributes them until there are no more tasks to execute and there are none still running whereupon it ends the process, ending the client and server threads

### Phase 2 - single machine / single process / multiple threads
As Phase #1 except:
- The server thread creates additional executor threads (using the execution library) that receive tasks and executes them.
- The client code (main thread) connects to the server thread as a submitter of sets of tasks for execution, indicating it is a foreground executor with STDIO
- The server distinguishes between tasks requiring STDIO and those not. All tasks requiring STDIO are dispatched to the client executor thread that is running in foreground

### Phase 3 - single machine / multiple processes 
- Server binary is built linking: execution library, server library and a server "main"
- Client binary is built linking: execution library, client library and a client "main"
- Server app is built linking the server library, execution library
- A server maybe running on the local machine already
- A client app starts up. It if discovers a server already running on the local machine, then it connects to it.
- If no local server is found running, one is started by the client, and then it connects to it
- There is a common "run-time" library that knows how to discover servers, connect to a server, submit tasks and receive tasks for execution
- The client registers itself as an executor of tasks (in particular ones that use STDIO)
- The client submits a set of tasks to be run, to the server
- The server can optionally create additional executor threads (using the same run-time library as the client)
- The server distributes tasks to executors that it knows about
- The server distributes them until there are no more tasks to execute and none still running
- When all tasks have been executed, the client quits, the server continue to run
- If the server has no additional sets of tasks it is distributing, then it can end executor threads and exit itself

### Phase 4 - multiple machines
A further extension to the problem which I won't try and cover just yet (unless things go very well) is:

The server being able to distribute tasks for execution to other connected servers, over the network. 

This would require previous 
distribution of the code for the tasks, binary compatibility of some task-executable between server platforms, 
or the serialization/deserialization and remote compilation (since I'm focussing on rust) and execution of the tasks

## The Tasks
I will define some very simple task, such as printing some text or a number, to be used in the sets of tasks as the tasks themselves are not important in this project.

In "flow" there is a standard library used by all executors to execute library functions, as well as a specific set of semantics about the tasks that determines when the execution should end. 
Here that is not important, the server logic to determine when the set of tasks has run to completion here can be simple.