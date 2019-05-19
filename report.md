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


### algorithm


### synchronization

### rationale

