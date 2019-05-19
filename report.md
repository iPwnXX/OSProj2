## task 1

### data structure

#### in thread.c  

add function:  
/*get a thread according to the given id*/  
struct thread * get_thread_by_id (tid_t){}  

#### in thread.h  

/*represent four state of child thread*/  
struct child_state {  
    tid_t child_id;  
    bool called_exit;  
    bool has_been_waited;  
    int child_exit_state;  
    struct list_elem elem_child_state;    
};  


#### in process.c  
/*file name split from argument*/  
char cp_name  

/*state of current child*/  
struct child_state s_child  

/*current thread*/  
struct thread *t_cur;  

##### in start_process()  
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
struct list_elem *e  

/*current status*/  
int cur_state   

##### in process_exit()  
/*add list element for finding child process during iteraction*/  
struct list_elem *iter, *next;  

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
