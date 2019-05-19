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
the key alogirhtm in argument passing is to parse and seperate the command and parameters passing in by deliminating white space. and set up a stack in memory to save argument in right order. in process_execute() we split the string of command and argument, which is used for thread name and the input of start_process() & setup_stack(). the commond name can be obtained from thread->name. And then in setup_stack(), the argument is written in reverse order for each string into stack and append  \0 for each argument. \*esp is the stack pointer originally initialled with PHASE_BASE and moved top down. each time get the strlength and move \*esp - strlengh and use memcpy to save the splitted string pattern. the argc is number of argument in 4 bytes. each string token is pushed downward in the page during scan of *file_name* while decreasing the \*esp.   

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
### Synchronization
### Rationale:
