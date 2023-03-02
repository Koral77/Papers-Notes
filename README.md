# Papers-Notes
# Notes List
- [SyRust: Automatic Testing of Rust Libraries with Semantic-Aware Program Synthesis](#syrust-automatic-testing-of-rust-libraries-with-semantic-aware-program-synthesis)
- [Understanding Memory and Thread Safety Practices and Issues in Real-World Rust Programs](#understanding-memory-and-thread-safety-practices-and-issues-in-real-world-rust-programs)


--- 

# SyRust: Automatic Testing of Rust Libraries with Semantic-Aware Program Synthesis
## The issues to be solved
1. **Basic Typing and Subtyping Constraints** Rust is statically typed: all variables are assigned a type at compile time and all uses of variables are consistent with their types. An type should be matched with its reference and an immutable reference should be matched with mutable reference, but not the other way around.
2. **Polymorphism and API Specification Refinement** To generates test cases for APIs with polymorphic types, the algorithm needs to know how the type variable T is instantiated and handle subsequent typing constraints(traits). Defining a notion of API refinement: a type signature that permits fewer inputs is considered more refined. For example, Vec\<String> is more refined than Vec\<T>, as the former permits only vectors of strings and the latter permits more types.
3. **Variable Ownership and Lifetime** The uniqueness of the ownership of a vaule must hold across the program.  A lifetime of a variable is a region in the code in which this variable can be used. If a variable is used, then the compiler ensures that this usage occurs within the lifetime of the variable.
4. **Relationship Between Lifetimes** If a variable b is a reference to another varabile a, then b's lifetime depends on a. That is, the lifetime of b must be strictly contained in the lifetime of a.
5. **Borrowing and Refenrences** To prevent data races, Rust enforces the rule that only one mutable reference can be active for any location in memory. The compiler will reject any attempts to borrow a mutable reference to a memory location while another mutable reference is active. Furthemore, mutable and immutable references for one memory location cannot co-exist, but a memory location can have many immutable only references.

## Methodology

### **Rules for Constraints**

1. **Basic Typing constraints**
> - Rule 1 states that the available variables and types at the start of the synthesis are the ones in the template T.
> - Rule 2 describes if x is defined in line i, then the type of x in line i + 1 is same as in line i and the types of variables belonging to the intersection of the domains of context(i) and context(i + 1) remain unchanged.
> - Rule 3 defines type checking for a multi-line program. 
>   - 1. Single-Line Type Check Definition: Let f: t1 × ... × tk → t' and line i contain a call to f(v1, ..., vk), line i containing f types checks if it satisfies the conditions that v1, ..., vk ∈ dom context(i) and types of all variables in dom context(i) ⊑ all types from the template and API signature and a polymorphic match is compatible for every pair of polymorphic variables. 
>   - 2. Multi-Line program Type Check Definition:  Every single line of code in the program satifies the conditions in Single-Line Type Check Definition.
2. **Ownership and Variable Lifetime**
> - Rule 4 makes sure that the types must be a primitive type or static referece (&) if the variables are the same.
> - Rule 5  used to terminate non-primitive variable lifetimes upon use.
> - Rule 6 states that the lifetime of reference &v is included in the variable v.
> - Rule 7  keeps the lifetime relation even when the types change and content is moved out.
> - Rule 8 promises while &mut is active, prevent another &mut and &. 
> - Rule 9 promises while & is active, prevent another &mut.

### **Hybrid API Refinement**

Dealing with the three most common types of polymorphic Rust APIs
1. No Input Polymorphism: \
*Example*: Vec::new() → Vec\<T> \
*Concretize*: \
The hybrid API refinement module mines concrete types from the API set and template.For example, if algorithm collect i32 and u32 from the API and template, it add 2 variants of Vec::new where the output type is changed to Vec\<i32> and Vec\<u32> respectively. This combinatorial enumeration generates a large number of APIs, so it should be only sparingly used. It is only used for No Input Polymorphism.
2. Polymorphic Input, Concrete Output: \
*Example*: Vec::push(&mut Vec\<T>, T) → () \
*Concretize*:\
For a majority of the cases, the subtyping strategy works fine. However, this strategy fails when type variables in the input are annotated with traits that any matching type must support. \
Instead of dealing with complex trait requirements, the algorithm uses the compiler errors as feedback to refine specifications of polymorphic variables. When a type match fails because of mismatching traits, it refines the specification of the type variable to no longer match that particular type. The refinement is usually complete after a few rounds.
3. Polymorphic Input, Polymorphic Output: \
*Example*: Vec :: pop(&mut Vec\<T>) → Option\<T> \
*Concretize*: \
For example, the synthesis algorithm matches &mut Vec\<T> with a concrete type such as &mut Vec\<i32>. The synthesizer generates a test case where Vec::pop is used with the input of type &mut Vec\<i32> and predicts the output type as Option\<i32>. This test case is then passed to the test executioner, which attempts to compile and run the test case. \
If the test case compiles successfully, we duplicate the
function and fully concretize the duplicated API’s inputs with the current input types. Since the program type-checked, the predicted output type is correct. We replace the output of the duplicated API with the prediction.
If the test case fails to compile, what happens next depends on what error we get. If the error is originating from the inputs to the API, then we perform the same refinement as in the previous section. Other errors are fjxed directly. For example, if we get the message "expected String, got Vec\<i32>", we duplicate the API, concretize the inputs, and set the output type to Vec\<i32>.\
After
duplicating the API, we add a constraint to the original API to prevent it from being used with the same combination of input types.

### **API Selection Strategy**

We allow few APIs to be manually selected to simulate the scenarios where the programmers want to test specific APIs. Then the rest of the APIs are chosen through weighted random selection. The weights depend on whether the API contains unsafe code or not. APIs that contain unsafe code are given 50% more weight than APIs that are completely free of unsafe code. To this set, we add default APIs that represent operations built into Rust: for assignment to mutable (let mut x = y) and two kinds of borrowing (&, &mut).


---

## Understanding Memory and Thread Safety Practices and Issues in Real-World Rust Programs

## Insights

1. (Reasons of Using unsafe) Most unsafe usages are for good or unavoidable reasons, indicating that Rust’s rule checks are sometimes too strict and that it is useful to provide an alternative way to escape these checks.
2. (Unsafe Removal) Interior unsafe is a good way to encapsulate unsafe code.
3. (Encapsulating Interior Unsafe) Some safety conditions of unsafe code are diffjcult to check. Interior unsafe functions often rely on the preparation of correct inputs and/or execution environments for their internal unsafe code to be safe.
4. (Memory Safety Issues) Rust’s safety mechanisms (in Rust’s stable versions) are very effective in preventing memory bugs. All memory-safety issues involve unsafe code (although many of them also involve safe code).
5. (Memory Safety Issues Fixing Strategies) More than half of memory-safety bugs were fixed by changing or conditionally skipping unsafe code, but only a few were fixed by completely removing unsafe code, suggesting that unsafe code is unavoidable in many cases.
6. (Thread Safety Issues: Blocking Bugs) Lacking good understanding in Rust’s lifetime rules is a common cause for many blocking bugs.
7. (Thread Safety Issues: Non-Blocking Bugs) There are patterns of how data is (improperly) shared and these patterns are useful when designing bug detection tools.
8. (Thread Safety Issues: Non-Blocking Bugs) How data is shared is not necessarily associated with how non-blocking bugs happen, and the former can be in unsafe code and the latter can be in safe code.
9. (Thread Safety Issues: Non-Blocking Bugs) Misusing Rust’s unique libraries is one major root cause of non-blocking bugs, and all these bugs are captured by runtime checks inside the libraries, demonstrating the effectiveness of Rust’s runtime checks.
10. (Thread Safety Issues: Non-Blocking Bugs) The design of APIs can heavily impact the Rust compiler's capability of identifying bugs. (&mut self vs. &self)
11. (Fixes of Non-Blocking Bugs) Fixing strategies of Rust non-blocking (and blocking) bugs are similar to traditional languages. Existing automated bug fixing techniques are likely to work on Rust too.

## Suggestions

1. (Reasons of Using unsafe) Programmers should try to find the source of unsafety and only export that piece of code as an unsafe interface to minimize unsafe interfaces and to reduce code inspection efforts.
2. (Unsafe Removal) Rust developers should first try to properly encapsulate unsafe code in interior unsafe functions before exposing them as unsafe.
3. (Encapsulating Interior Unsafe) If a function’s safety depends on how it is used, then it is better marked as unsafe not interior unsafe.
4. (Encapsulating Interior Unsafe)  Interior mutability can potentially violate Rust’s ownership borrowing safety rules, and Rust developers should restrict its usages and check all possible safety violations, especially when an interior mutability function returns a reference. We also suggest Rust designers difgerentiate interior mutability from real immutable functions.
5. (Memory Safety Issues) Future memory bug detectors can ignore safe code that is unrelated to unsafe code to reduce false positives and to improve execution efficiency.
6. (Thread Safety Issues: Blocking Bugs) Future IDEs should add plug-ins to highlight the location of Rust’s implicit unlock, which could help Rust developers avoid many blocking bugs.
7. (Fixing Blocking Bugs) Rust should add an explicit unlock API of Mutex, since programmers may not save the return value of lock() in a variable and explicitly dropping the return value is sometimes inconvenient.
8. (Thread Safety Issues: Non-Blocking Bugs) Internal mutual exclusion must be carefully reviewed for interior mutability functions in structs implementing the Sync trait.


## Reasons of Usage

1. Most of them (66%) are for (unsafe) **memory operations**, such as *raw pointer manipulation* and *type casting*. In Rust std, the heaviest unsafe usages appear in the sys module, likely because it interacts more with low-level systems.
2. Further analyze the purposes of our studied 600 unsafe usages. The most common purpose of the unsafe usages is to **reuse existing code** (42%), for example, to convert a C-style array to Rust’s variable-size array (called slice), to call functions from external libraries like glibc. Another common purpose of using unsafe code is to **improve performance** (22%). The remaining unsafe usages include bypassing Rust’s safety rules to **share data across threads** (14%) and other types of Rust compiler check bypassing.
3. Sometimes removing unsafe will not cause any compile errors (32 or 5% of the studied unsafe usages). For 21 of them, programmers mark a function as unsafe for **code consistency** (e.g., the same function for a different platform is unsafe). For the rest, programmers use unsafe to give a **warning of possible dangers** in using this function.
4. Five unsafe usages among the above no-compile-error cases are for labeling struct constructors (there are also 50 such usages in the Rust std library). These constructors only contain safe operations (e.g., initializing struct fields using input parameters), but other functions in the struct can perform unsafe operations and their safety depends on safe initialization of the struct. For example, in Rust *std*, the String struct has a constructor function String::from_utf8_unchecked() which creates a String using the input array of characters. This constructor is marked as an unsafe function although the operations in it are all safe. Other member functions of String that use the array content could potentially have safety issues due to invalid UTF-8 characters. Marking constructor unsafe is more efficient and reliable than marking all these functions which make programmer to check inputs of each of them.
5. Rust *std* does not perform any explicit condition checking in most of its interior unsafe function. Instead, it ensures that **the input or the environment** that the interior unsafe code executes with is safe. For example, the unsafe function Arc::from_raw() always takes input from the return of Arc::into_raw() in all sampled interior unsafe functions. Rust std performs explicit checking for the rest of the interior unsafe functions.
6. The cases where interior unsafe code is **improperly encapsulated**: do not perform any checking of return value from external library function calls, directly derefence input parameters or use them directly as indices to access memory without any boundary checking, not checking the validity of function pointers, using type casting to change objects’ lifetime to static, and potentially accessing uninitialized memory.
7. Of particular interest are two bad practices that lead to potential problems. The potential error below is caused by holding an immutable reference while changing the underlying object. It can be fixed by passing &mut self to pop() rather than &self.
```
1   impl<T, ...> Queue<T, ...> {
2       pub fn pop(&self) -> Option<T> {unsafe {...}}
3       pub fn peek(&self) -> Option<&mut T> {unsafe {...}}
4   }
5   // let e = Q.peek().unwrap();
6   // {Q.pop()}
7   // println!("{}", *e); < - use after free
```

## Memory Safety Issues

### **Bug categories**
1. *Buffer overflow* :   
An error happens when computing buffer size or index in safe code and an out-of-boundary memory access happens later in unsafe code; \
Checks do not work due to wrong checking logic, inconsistent struct data, or integer overflow; \
Input parameters are used directly or indirectly as an index to access a buffer, without any boundary checks.
2. *Null pointer dereferencing* :   
All bugs in this category are caused by dereferencing a null pointer in unsafe code.
3. *Reading uninitialized memory* :   
All bugs detected are defined in unsafe and used in safe.
4. *Invalid free* :   
Out of the ten invalid-free bugs, five share the same (unsafe) code pattern. f points to an uninitialized memory buffer (line 6). Assigning a new FILE struct to *f at line 7 ends the lifetime of the previous one, causing the previous one to be dropped by Rust. However, the previous one contains an uninitialized memory buffer, freeing its heap memory is invalid.   
``` 
1   pub struct FILE {
2       buf: Vec<u8>,
3   }
4  
5   pub unsafe fn _fdopen(...) {
6       let f = alloc(size_of::<FILE>()) as * mut FILE;
7  -    *f = FILE{buf: vec![0u8; 100]};
8  +    ptr::write(f, FILE{buf: vec![0u8; 100]});  
9   }
```
5. *Use after free*:   
Caused by an object is dropped implicitly by Rust in safe code (when its lifetime ends), but a pointer to the object or to a field of the object still exists and is later derenferenced in unsafe code.   
In the code below, p in line 2 gets the address of a BioSlice object and it is used to call an unsafe function CMS_sign() which dereferences p. However, the object p points to is deallocated after line 5. So p in line 12 is empty.
```
1   pub fn sign(data: Option<&[u8]>) {
2  -    let p = match data {
3  -        Some(data) => BioSlice::new(data).as_ptr(),
4  -        None => ptr::null_mut(),
5  -    };
6  +    let bio = match data {
7  +        Some(data) => Some(BioSlice::new(data));
8  +        None => None,
9  +    };
10      let p = bio.map_or(ptr::null_mut(), |p| p.as_ptr());
11      unsafe {
12          let cms = cvt_p(CMS_sign(p));
13      }
14  }
```
6. *Double free*:   
2 out of 6 bugs are safe -> unsafe, the rest are unsafe -> safe and unique to Rust. These buggy programs first conduct some unsafe memory operations to create two owners of a value. When these owners’ lifetime ends, their values will be dropped (twice), causing double free. One such bug is caused by the code below, which reads the content of t1 and puts it into t2 without moving t1. If type T contains a pointer field that points to some object, the object will have two owners, t1 and t2. When t1 and t2 are dropped by Rust implicitly when their lifetime ends, double free of the object happens. A safer way is to moce the ownership from t1 to t2.   
```
t2 = ptr::read::<T>(&t1)
```

### **Fixing Strategies**

1. *Conditionally skip code* :   
Some bugs were fixed by capturing the conditions that lead to dangerous operations and skipping the dangerous operations under these conditions. For example, when the offset into a buffer is outside its boundary, buffer accesses are skipped.
2. *Adjust lifetime* :   
Some bugs were fixed by changing the lifetime of an object to avoid it being dropped improperly. These include extending the object’s lifetime to fix use-after-free (example in category 5), changing the object’s lifetime to be bounded to a single owner to fix double-free, and avoiding the lifetime termination of an object when it contains uninitialized memory to fix invalid free (example in category 4).
3. *Changing unsafe operands* :   
Some bugs were fixed by modifying operands of unsafe operations, such as providing the right input when using memcpy to initialize a buffer and changing the length and capacity into a correct order when calling Vec::from_raw_parts().
4. *Other* 

## Thread Safety Issues

### **Blocking Bugs (dead lock)**
55 out of 59 blocking bugs are caused by operations of synchronization primitives, like *Mutex* and *Condvar*. All these synchronization operations have safe APIs, but their implementation heavily uses interior-unsafe code, since they are primarily implemented by reusing existing libraries like pthread. The other four bugs are not caused by primitives’ operations.
1. *Mutex and RwLock* :   
Failing to acquire lock (for Mutex) or read/write (for RwLock) results in thread blocking for 38 bugs, with 30 of them caused by double locking, seven caused by acquiring locks in confmicting orders, and one caused by forgetting to unlock when using a self-implemented mutex.   
Here is an example. *client* is protected by an *RWLock*. At line 3, its *read* lock is acquired and its m field is used as input to call function connect(). If connect() returns Ok, the *write* lock is acquired at line 7. The *write* lock will cause a double lock, since the lifetime of the temporary reference-holding object returned by client.read() spans the whole match code block and the read lock is held until line 11. The patch is to save to the return of connect() to a local variable to release the read lock at line 4, instead of using the return directly as the condition of the match code block.   
Rust developers need to have a good understanding of the lifetime of a variable returned by lock(), read(), or write() to know when unlock() will implicitly be called.
```
1   fn do_request() {
2       // client: Arc<RwLock<Inner>>
3  -    match connect(client.read().unwrap().m) {
4  +    let result = connect(client.read().unwrap().m);
5  +    match result {
6           Ok(_) => {
7               let mut inner = client.write().unwrap();
8               inner.m = mbrs;   
9           }
10          Err(_) => {}
11      };    
12  }
```
2. *Condvar* :   
In eight of the ten bugs related to Condvar, one thread is blocked at wait() of a Condvar, while no other threads invoke notify_one() or notify_all() of the same Condvar. In the other two bugs, one thread is waiting for a second thread to release a lock, while the second thread is waiting for the first to invoke notify_all().
3. *Channel* : 
In Rust, a channel has unlimited buffer size by default, and pulling data from an empty channel blocks a thread until another thread sends data to the channel. Rust also supports channel with a bounded bufger size. When the buffer of a channel is full, sending data to the channel will block a thread.
4. *Once* :   
*Once* is designed to ensure that a global variable is only initialized once. The initialization code can be put into a closure and used as the input parameter of the call_once() method of a Once object. Even when multiple threads call call_once() multiple times, only the first invocation is executed. However, when the input closure of call_once() recursively calls call_once() of the same Once object, a deadlock will be triggered.

### **Fixing Blocking Bugs**
1. Adjusting synchronization operations, including adding new operations, removing unnecessary operations, and moving or changing existing operations. One fixing strategy unique to Rust is adjusting the lifetime of the returned variable of lock() (or read(), write()) to change the location of the implicit unlock().
2. One strategy to avoid blocking bugs is to explicitly define the boundary of a critical section. Rust allows explicit drop of the return value of lock() (by calling mem::drop()). Although efgective, this method is not always convenient, since programmers may want to use lock() functions directly without saving their return values.
### **Non-Blocking Bugs**
Non-blocking bugs are concurrency bugs where all threads can finish their execution, but with undesired results. Non-blocking bugs include data race, atomicity violation and order violation.
1. *Sharing with unsafe code* :   
The most common way to share data is by passing a raw pointer to a memory space. A thread can store the pointer in a local variable and later dereference it or cast it to a reference. All raw pointer operations are unsafe, although after (unsafe) casting, accesses to the casted reference can be in safe code.   
The second most common type of data sharing to be accessing OS system calls and hardware resources (through unsafe code). For example, in one bug, multiple threads share the return value of system call getmntent(), which is a pointer to a structure describing a file system. 
The other two unsafe data-sharing methods are accessing static mutable variables which is only allowed in unsafe code, and implementing the unsafe Sync trait for a struct.
2. *Sharing with safe code* :   
Even though the sharing of any single value in safe code follows Rust’s safety rules (i.e., no combination of aliasing and mutability), bugs still happen because of violations to programs’ semantics. 5 out of 15 bugs use atomic variables as shared variables, and the other ten bugs wrap shared data using Mutex (or RwLock). To ensure lifetime covers all usages, nine bugs use Arc to wrap shared data and the other six bugs use global variables as shared variables.

### **Fixes of Non-Blocking Bugs**
1. Enforcing atomic accesses to shared memroy.
2. Enforcing ordering between two shared-memroy access from different threads.
3. Avoiding (problematic) shared memory accesses.
4. Changing application-specific logic.
5. Making a local copy of some shared memory.

## Bug Detection
### **Detecting Memory Bugs**
Many memroy-related issues are caused by misuse of ownership and lifetime. Thus, an efficient way to avoid or detect memory bugs in Rust is to analyze object ownership and lifetime.
1. **IDE tools**   
Being able to visualize objects’ lifetime and owner(s) during programming time could largely help Rust programmers avoid memory bugs.
2. **Static detectors**   
Ownership/lifetime information can also be used to statically detect memory bugs. It is feasible to build static checkers to detect invalid-free, use-after-free, double-free memory bugs by analyzing object lifetime and ownership relationships.    
In this paper, authors have built a new static checker based on lifetime/ownership analysis of Rust's midlevel intermediate representation (MIR) to detect use-afterfree bugs.   
MIR provides explicit ownership/lifetime information and rich type information. The detector maintains the state of each variable (alive or dead) by monitoring when MIR calls StorageLive or StorageDead on the variable. For each pointer/reference, they conduct a "points-to" analysis to maintain which variable it points to/references. This points-to analysis also includes cases where the ownership of a variable is moved. When a pointer/reference is dereferenced, the tool checks if the object it points to is dead and reports a bug if so.
### **Detecting Concurrency Bugs**
1. **IDE tools**
Rust’s implicit lock release is the cause of several types of blocking bugs. An effective way to avoid these bugs is to visualize critical sections. The boundary of a critical section can be determined by analyzing the lifetime of the return of function lock(). Highlighting blocking operations such as lock() and channel-receive inside a critical section is also a good way to help programmers avoid blocking bugs.   
2. **Static detectors**
Lifetime and ownership information can be used to statically detect blocking bugs. By analyzing lock lifetime, a double lock detector was built. It first identifies all call sites of lock() and extracts two pieces of information from each call site: the lock being acquired and the variable being used to save the return value. As Rust implicitly releases the lock when the lifetime of this variable ends, our tool will record this release time. We then check whether or not the same lock is acquired before this time, and report a double-lock bug if so.
