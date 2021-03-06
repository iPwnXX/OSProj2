## task 1 ARGUMENT PASSING

### Data structure and functions

#### in thread.c  

add function:  
```
+ /*get a thread according to the given id*/  
struct thread * get_thread_by_id (tid_t){}  
```
#### in thread.h  
```
 /*represent four state of child thread*/  
struct child_state {  
    tid_t child_id;  
    bool called_exit;  
    bool has_been_waited;  
    int child_exit_state;  
    struct list_elem elem_child_state;    
};  

```
#### in process.c  
```
/*file name split from argument*/  
char cp_name;  

/*state of current child*/  
struct child_state s_child  

/*current thread*/  
struct thread *t_cur;  
```
##### in start_process()  
```
/*current thread*/  
struct thread *t_cur;  

/*parent thread*/  
struct thread *t_parent;  

/*flag of load status*/  
bool load_flag;  

##### in process_wait()  
/*current thread*/  
struct thread *t_cur;  

/*child status*/  
struct child_state* s_child;  

/*list of element*/   
struct list_elem *e;  

/*current status*/  
int cur_state;   
```
##### in process_exit()  
```
/*add list element for finding child process during iteraction*/  
struct list_elem *iter, *next;  
```
### algorithm
+ `setup_stack()`   
   + the key alogirhtm in argument passing is to parse and seperate the command and parameters passing in by deliminating white space. and set up a stack in memory to save argument in right order. in process_execute() we split the string of command and argument, which is used for thread name and the input of start_process() & setup_stack(). the commond name can be obtained from thread->name. And then in setup_stack(), the argument is written in reverse order for each string into stack and append  \0 for each argument. \*esp is the stack pointer originally initialled with PHASE_BASE and moved top down. each time get the strlength and move \*esp - strlengh and use memcpy to save the splitted string pattern. the argc is number of argument in 4 bytes. each string token is pushed downward in the page during scan of *file_name* while decreasing the \*esp.   

### synchronization

in *process_start()* after invoke *load()* for loading the executable, if return success and freed the allocated page, we need to inform the parent process that the child process has ended. The code here is as follows:   
 lock_acquire(&parent->child_lock);   
parent->child_load_state = load_state;   
cond_signal(&parent->cond_child, &parent->child_lock);   
lock_release(&parent->child_lock);   
acquire lock in case of concurrent invoking,and wake up the waiting parent thread. then release the lock.  

### rationale
this approach of setting up stack can handle overflowing of stack page. During setup_stack(), we scan through the input *file_name* to add each argument, without counting the space of memory requaired. and after going through, the overflow can be handled by page fault exeption if the allocated address is invalid. it will return -1 in exit().  


## TASK2 PROCESS CONTROL SYSCALLS

### Data structure and functions:
##### In thread.h
```
/* The status of child thread. */
struct child_status {
  tid_t child_id                        
  bool is_exit_called;                  
  bool has_been_waited;                
  int child_exit_status;                
  struct list_elem elem_child_status;   
};
```
##### In thread struct
```
#ifdef USERPROG
     tid_t parent_id;                    /* parent thread id */

    uint32_t *pagedir;                  /* Page directory. */
    /* signal to indicate the child's executable-loading status:
     *  - 0: has not been loaded
     *  - -1: load failed
     *  - 1: load success*/
    int child_load_status;

    /* monitor used to wait the child, owned by wait-syscall and waiting
       for child to load executable */
    struct lock lock_child;
    struct condition cond_child;

    /* list of children, which should be a list of struct child_status */
    struct list children;

    /* file struct represents the execuatable of the current thread */
    struct file *exec_file;
 #endif
```
### Algorithms

+ `void halt(void)`  
Call `shutdown_power()` directly.

+ `void exit(int status)`
    + Iterate current thread's children list, if find the child thread, which thread id is equal to current thread's thread id, update child thread's attributes.
    + Finally call `thread_exit()`, which executes `process_exit()` to free the current process's resources in process.c.

+ `pid_t exec(const char *cmd_line)`  
    + First, using `is_valid_ptr()`, which is implemented in syscall.c to check if the user pointer is valid.
    + Then, call `process_execute()` to start a new thread running a user program loaded from cmd_line.
    + Next, child thread is waited by a while loop until it is loaded successfully or failed.
    + Finally, return the new thread's thread id. Notice that in our implementation, when there is a user process within a kernal thread, we use one-to-one mapping from tid to pid, which means pid=tid.

+ `int wait(pid_t pid)`  
Call `process_wait()` in process.c  

    + Iterate current thread's children list to find the child thread.  
    + Parent thread will wait until child thread is exit by checking `thread_get_by_id()` in thread.c.
    + When parent thread is signaled the child thread is exist, it will start to check the status. If child thread `is_existed_called` and `has_been_waited` are both true, then update status to child's exit status and update `has_been_waited` to be true.

### Synchronization

We used lock to ensure that only the running thread can get into the criticle section for thread safe in syscall functions.

### Rationale:

In our design, the shortcoming is that each time when executing a syscall function, it will iterate all of the children list, which the time cost is O(n).  

Coding part is mainly on userprog/syscall.c and userprgs/process.c and thread/thread.h. We encapulate most funtional parts in syscall.c and mainly hard parts are in process.c.

In the future, our design is easy to extend, because we could add new structs in thread.h and process.c, and extend new features according to new structs.   

## task 3 FILE OPERATION SYSCALLS
### Data structure and functions
##### in syscall.c
```
struct file_descriptor
{
  int fd_num;
  tid_t owner;
  struct file *file_struct;
  struct list_elem elem;
};

/* a list record the file that is opened by syscall in user process. */
struct list list_open_file; 

/* lock that ensure only one thread can enter file system at the same time */
struct lock file_sys_lock;

```
### Algorithms
#### in syscall.c

+ `bool create()`   
/*the function create a new file with specific file size bytes and does not   
   + open it and return true if successfull*/
first check if the file name is valid, then acquire the lock in case of synchronization error.   
if all above is okay, invoke filesys_create() and return the status, then release the lock.

+ `bool remove()`      
   + all pocedures are the same with create() except invoking filesys_remove()

+ `int read()`   
/*read the file with given size of bytes into buffer and return how many bytes has been    
actually read or -1 for read fail */   
   + First check the validation of the pointer: buffer and buffer + size, exit -1 if not okay.  
then acquire the file system lock to become the lock holder. next check the status, if it is STDOUT_FILENO   
get input from standard input. else, invoke *get_open_file() to find the file, and use file_read() to read file   
and return the status.

+ `int write()`   
/*writes bytes of given size from buffer to the open file and return how many bytes has been    
actually write or -1 for read fail */   
   + check the buffer pointer is valid, then acquire the file system lock, after that judge the status.   
   if it is STDOUT_FILENO, put buffer to console.  else, invoke *get_open_file() to find the file, and use   
   file_write() to write file into buffer.

### Synchronization

For each file operation, we append file_system_lock to make sure there is only one thread opearting the file system.   
function such as open(), read() or write() all require acquiring the lock first in function body before modify file struct.

### Rationale:
in every of the new function, each time when we access the memory, we first check the validation the memory pointer, if it is a valid memory location, then we can excecute the following code, otherwise return -1 to thread.

## Q&A

+ A reflection on the project – what exactly did each member do? What went well, and
what could be improved?   

    11612210： task1 & task3 deal with argument passing in process.c and file system call in syscall.c    
    with corresponding report part

    11612023: task2 deal with process control system call in syscall.c with corresponding report part

    the struct definition in .h is doing well. while the function definition style can be improved to be uniform.
    
+ Does your code exhibit any major memory safety problems (especially regarding C strings), memory leaks, poor error handling, or race conditions?

        in argument passing setup_stack() function, we dealed with overflow by marking the exit state    
        (mentioned in task 1 algorithm). In file system call, each time invoking function such as   
        read/write/open, we first check the validation of the given memory pointer to gurantee   
        safe acccess.

+ Did you use consistent code style? Your code should blend in with the existing Pintos
code. Check your use of indentation, your spacing, and your naming conventions.   

       the naming of variables and functions are kept the same style as the orignal pintos,   
       where token is split with '-', for example get_thread_by_id

+ Is your code simple and easy to understand?   

      yes, we keep our added code similar to the original code to make it understandable, and   
      the naming style clear and explicit.

+ If you have very complex sections of code in your solution, did you add enough com-
ments to explain them?

      yes, in processs.c setup_stack(), since the procedure is considered to be complex, we leaved    
      enough comment besides to explain the stack process.

+  Did you leave commented-out code in your final submission?

       no, we delete those dead code.
       
+  Did you copy-paste code instead of creating reusable functions?
       for those function that will be repeatly used, we extract them out and put it into new functions. such    
       as get_thread_by_id
       
+  Are your lines of source code excessively long? (more than 100 characters)
     
       most of the lines are kept within lengh less than 100 characters. longer code will start   
       a new line.

+ Did you re-implement linked list algorithms instead of using the provided list manipulation
     
       we choose to use the given list and list elem stead of creating our own data container.
