# Papers-Notes
# Notes List
### Testing and Fuzzing
- [SyRust: Automatic Testing of Rust Libraries with Semantic-Aware Program Synthesis](#syrust-automatic-testing-of-rust-libraries-with-semantic-aware-program-synthesis)
- [RULF: Rust Library Fuzzing via API Dependency Graph Traversal](#rulf-rust-library-fuzzing-via-api-dependency-graph-traversal)
- [Search-Based Test Suite Generation for Rust](#search-based-test-suite-generation-for-rust)
- [RUSTY: A Fuzzing Tool for Rust](#rusty-a-fuzzing-tool-for-rust)
- [What is property-based testing?](#what-is-property-based-testing)
### Static Analysis
- [Understanding Memory and Thread Safety Practices and Issues in Real-World Rust Programs](#understanding-memory-and-thread-safety-practices-and-issues-in-real-world-rust-programs)
- [Detecting Unsafe Raw Pointer Dereferencing Behavior in Rust](#detecting-unsafe-raw-pointer-dereferencing-behavior-in-rust)
- [RUDRA: Finding Memory Safety Bugs in Rust at the Ecosystem Scale](#rudra-finding-memory-safety-bugs-in-rust-at-the-ecosystem-scale)
- [SafeDrop: Detecting Memory Deallocation Bugs of Rust Programs via Static Data-Flow Analysis](#safedrop-detecting-memory-deallocation-bugs-of-rust-programs-via-static-data-flow-analysis)
- [MirChecker: Detecting Bugs in Rust Programs via Static Analysis](#mirchecker-detecting-bugs-in-rust-programs-via-static-analysis)


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

# RULF: Rust Library Fuzzing via API Dependency Graph Traversal
## Overview : 
1. We compose fuzz targets based on the API dependency graph of a given library.
2. We breadthfirst search (BFS) API sequences under a length threshold on the graph.
3. For each uncovered API (deep-API) due to the length limitation, we backward search their dependent API sequences.
4. Finally, we refine our set of sequences to obtain a minimum subset that covers the same set of APIs.
## Four objectives: 
- **Validty** : The synthesized fuzz targets should be able to be successfully compiled
- **API coverage** : The set of generated fuzz targets should cover as many APIs as possible.
- **Efficiency** : The set of generated fuzz targets should be efficient for fuzzing. 
- **Effectiveness** : The generated targets should be friendly to fuzzing tools for bug hunting. In order to fuzz an API effectively, the generated program should be small.
## Approach
### API Dependency Graph
- Define an API dependency graph as a directed graph :   
$G = (FN_m, PAR_n, PE_p,CE_q)$   
where   
    - $FN_m$ is API nodes ;   
    - $PAR_n$ is parameter nodes ;
    - $PE_p$ is producer edges ⊆ $FN_m × PAR_n$, where an edge $fn_i→par_j$ implies $fn_i$ returns a value of type $par_j$ ;   
    - $CE_q$ is consumer edges ⊆ $PAR_n × FN_m$, where an edge $par_i→fn_j$ implies $par_i$ is a non-primitive parameter for $fn_j$. The weight of the edge means the number of $par_i$.   
### Validity of API Sequence
- **Reachable** : An API node is reachable if and only if it is a start node or all its required parameter nodes are reachable and the weights of consumer edges are satisfied. Similarly, a parameter node is reachable if at least one API node that can produce the parameter is reachable.
- **Validity** : An API sequence $fn_0, ..., fn_k$ is valid if $fn_0$ is a start node and each $fn_i (0 < i < k)$  is reachable given the subsequences of $fn_0, ..., fn_{i-1}$.
### API Sequence Generation
1. *BFS with Pruning*   
2. *Backward search*
3. *Merge and Refine* : prefer a candidate sequence A over B based on the following rules.   
    - A covers more new nodes than B.
    - A and B cover the same number of new nodes, but A covers more new edges than B.
    - A and B cover the same number of new nodes and edges, but A is shorter than B.
## Implementation
### Construct API Dependency Graph
1. Extract all public API signatures from the source code through rustdoc.
2. After we extract all public API signatures, we infer dependencies among APIs based on type inference to build dependency graph.
### Generate API Sequences
- If the API can mutate the value of a variable (get &mut as input), it is not appropriate that we treat this API as an end node.
- If a value is moved, we add a tag to the variable which owns the value and do not use this variable in the following statements. 
- We select some combinations of variable types and call types that we think are commonly used in API calls and mark these combinations as copying cases if they do be.
- Except for several cases marked as copying cases, we assume that all other cases are moving cases.
### Synthesize Program

---


# Search-Based Test Suite Generation for Rust
## Encoding
Each statement $s_i$ in a test case is a value $v(s_i)$, which has a type $\tau(v(s_i))\in T$, where $T$ is the finite set of types. There can be six different types of statements:
1. **Primitive statements** represents numeric variables, e.g., *let v = 42*. The primitive variable defines the value and type of the statement.
2. **Struct initializations** generate instances of a given struct, e.g., *let b = Book {name: "The Hobbit"}*. The object constructed in the statement defines the value and statement’s type. A struct instantiation can have parameters whose values are assigned out of the set {$v(s_k) | 0 \leq k < i$}.
3. **Enum initializations** generate instances of a given enum, e.g., let opt: Option<i32> = None;. The enum instance defines the value and
statement’s type. An enum instantiation can have parameters whose values are assigned out of the set {$v(s_k) | 0 \leq k < i$}.
4. **Field statements** access member variables of objects, e.g., *let b = a.x*. The member variable defines the value and the field’s statement type. The source of the member variable, i.e., *a*, must be part of the set {$v(s_k) | 0 \leq k < i$}.
5. **Associative function statements** invoke associative functions of datatypes, e.g., *let b = a.len()*. The owner of the function (if non-static) and all of the parameters must be values in {$v(s_k) | 0 \leq k < i$}. The return value determines the statement’s value and type.
6. **Function statements**  invoke loose functions, i.e., functions that are not associated with any datatype, for instance, *let a = foo()*. The parameters of the function must be values in {$v(s_k) | 0 \leq k < i$}. The return value determines the statement’s value and type.

## Implementation
### Three Intermediate Steps
1. First, the tool requires information about which data types the Crate has and which functions the data types provide. It performs analysis at the High-level Intermediate Representation (HIR) level. The HIR analysis yields a collection of Generators. These provide an overview of which data types can be instantiated in a test case.
2. To evaluate the Generated Tests in terms of their coverage, RustyUnit instruments the Mid-level Intermediate Representation (MIR) and compiles the crate yielding an Instrumented Binary. MIR is a CFG. The tool injects instructions into the MIR to trace the execution of individual basic blocks and branches. If a generated test executes a code location in the crate, the event is stored in the Execution Traces. If an executed basic block is an entry point of a branch, RustyUnit computes how the values, which the conditional depends on, need to be changed to hit other branches in that context, i.e., branch distance (BD). The BD is 0 for all basic blocks that a test case executes. 
3. The collected BDs in the execution traces are only one part. To calculate the overall fitness value with respect to each coverage target, RustyUnit must additionally determine the approach level (AL) from the corresponding Control Dependence Graphs (CDGs). AL describes how far a test case was from a target in the corresponding CDG when the test case deviated from the course, i.e., the number of missed control dependencies between the target and the point where the test case took a wrong turn.   
RustyUnit calculates the overall fitness value of a test *t* with respect to target *m* as follows and normalizes Branch distance with function $\alpha$. The goal is to push the fitness value *F* of a test case with respect to a coverage target to zero to cover the target.
$$F_m(t) = Approach Level + \alpha(Branch Distance)$$
$$\alpha(x)=\frac x{x+1}$$

### Handling Generics and Traits
1. If RustyUnit generates a statement whose return value is to be used as a parameter for another statement, the tool replaces the generic type parameters of the return type by concrete ones, while any other generic types are chosen at random; for instance, to generate an argument of type *Option\<i32>*, RustyUnit could invoke *foo* with type parameter A being *i32*.Since B is not constrained by the concrete return type and any trait bounds, the type is free to choose. In general, RUSTYUNIT mainly uses primitive data types for a SUT to keep the generated tests simple as fa as this satifies the defined trait bounds.
```
1   impl<A, B> for FoorBar<A, B> {
2       fn foo(&self, x: B, v: &Vec<A>) -> Option<A> {/*...*/} 
3   }
```
2. Otherwise, all generic type parameters of the corresponding statement are selected randomly.

---

## What is property-based testing?
Property-Based testing consists of two parts. [See more](https://forallsecure.com/blog/what-is-property-based-testing).
### Property-based test target
Properties refer to certain laws or characteristics that the output should satisfy. For example: 
```
 import (
	"strings"
	"testing"
	"testing/quick"
)

func TestCommonPrefix(t *testing.T) {
	propertyTest := func(s1, s2 string) bool {
		prefix := CommonPrefix(s1, s2)
		if !strings.HasPrefix(s1, prefix) || !strings.HasPrefix(s2, prefix) {
			return false
		}
		if len(prefix) == len(s1) || len(prefix) == len(s2) {
			return true
		}
		return s1[len(prefix)] != s2[len(prefix)]
	}
	if err := quick.Check(propertyTest, nil); err != nil {
		t.Error(err)
	}
}
```
In the example above, we check two properties of the output:
- The two input strings (s1 and s2) should start with the prefix.
- The prefix is the longest common prefix.
### Input Generators
A generator is a function that returns an instance of a given type from a source of randomness. In the previous section, we wrote a parameterized test that had two strings as arguments. The property-based testing framework automatically detected that the function expected a string. Using one of its default input generator, generated random strings automatically.


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
5  +        match result {
6               Ok(_) => {
7                   let mut inner = client.write().unwrap();
8                   inner.m = mbrs;   
9               }
10              Err(_) => {}
11          };
12      }
13  }
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


---

# Search-Based Test Suite Generation for Rust
## Encoding
Each statement $s_i$ in a test case is a value $v(s_i)$, which has a type $\tau(v(s_i))\in T$, where $T$ is the finite set of types. There can be six different types of statements:
1. **Primitive statements** represents numeric variables, e.g., *let v = 42*. The primitive variable defines the value and type of the statement.
2. **Struct initializations** generate instances of a given struct, e.g., *let b = Book {name: "The Hobbit"}*. The object constructed in the statement defines the value and statement’s type. A struct instantiation can have parameters whose values are assigned out of the set {$v(s_k) | 0 \leq k < i$}.
3. **Enum initializations** generate instances of a given enum, e.g., let opt: Option<i32> = None;. The enum instance defines the value and
statement’s type. An enum instantiation can have parameters whose values are assigned out of the set {$v(s_k) | 0 \leq k < i$}.
4. **Field statements** access member variables of objects, e.g., *let b = a.x*. The member variable defines the value and the field’s statement type. The source of the member variable, i.e., *a*, must be part of the set {$v(s_k) | 0 \leq k < i$}.
5. **Associative function statements** invoke associative functions of datatypes, e.g., *let b = a.len()*. The owner of the function (if non-static) and all of the parameters must be values in {$v(s_k) | 0 \leq k < i$}. The return value determines the statement’s value and type.
6. **Function statements**  invoke loose functions, i.e., functions that are not associated with any datatype, for instance, *let a = foo()*. The parameters of the function must be values in {$v(s_k) | 0 \leq k < i$}. The return value determines the statement’s value and type.

## Implementation
### Three Intermediate Steps
1. First, the tool requires information about which data types the Crate has and which functions the data types provide. It performs analysis at the High-level Intermediate Representation (HIR) level. The HIR analysis yields a collection of Generators. These provide an overview of which data types can be instantiated in a test case.
2. To evaluate the Generated Tests in terms of their coverage, RustyUnit instruments the Mid-level Intermediate Representation (MIR) and compiles the crate yielding an Instrumented Binary. MIR is a CFG. The tool injects instructions into the MIR to trace the execution of individual basic blocks and branches. If a generated test executes a code location in the crate, the event is stored in the Execution Traces. If an executed basic block is an entry point of a branch, RustyUnit computes how the values, which the conditional depends on, need to be changed to hit other branches in that context, i.e., branch distance (BD). The BD is 0 for all basic blocks that a test case executes. 
3. The collected BDs in the execution traces are only one part. To calculate the overall fitness value with respect to each coverage target, RustyUnit must additionally determine the approach level (AL) from the corresponding Control Dependence Graphs (CDGs). AL describes how far a test case was from a target in the corresponding CDG when the test case deviated from the course, i.e., the number of missed control dependencies between the target and the point where the test case took a wrong turn.   
RustyUnit calculates the overall fitness value of a test *t* with respect to target *m* as follows and normalizes Branch distance with function $\alpha$. The goal is to push the fitness value *F* of a test case with respect to a coverage target to zero to cover the target.
$$F_m(t) = Approach Level + \alpha(Branch Distance)$$
$$\alpha(x)=\frac x{x+1}$$

### Handling Generics and Traits
1. If RustyUnit generates a statement whose return value is to be used as a parameter for another statement, the tool replaces the generic type parameters of the return type by concrete ones, while any other generic types are chosen at random; for instance, to generate an argument of type *Option\<i32>*, RustyUnit could invoke *foo* with type parameter A being *i32*.Since B is not constrained by the concrete return type and any trait bounds, the type is free to choose. In general, RUSTYUNIT mainly uses primitive data types for a SUT to keep the generated tests simple as fa as this satifies the defined trait bounds.
```
1   impl<A, B> for FoorBar<A, B> {
2       fn foo(&self, x: B, v: &Vec<A>) -> Option<A> {/*...*/} 
3   }
```
2. Otherwise, all generic type parameters of the corresponding statement are selected randomly.

---

# RUSTY: A Fuzzing Tool for Rust
## Fuzzing Approach
1. Generating arbitrary inputs through Fuzzing tool.
2. Performing property-baes tesing, if a failure is found, RUSTY automatically finds the minimal test case to reproduce it.
3. RUSTY leverages a **concolic** execution engine to produce concrete exploit values for dicovered bugs.

---

# Detecting Unsafe Raw Pointer Dereferencing Behavior in Rust
## Threat Analysis
```
fn generate_mutable_reference<T>(v: &Vec<T>) -> &mut Vec<T> {
    unsafe {
        let p = v as *const Vec<T>;
        let q = p as *mut Vec<T>;
        &mut *q
    }
}

fn exploit_multiple() {
    let mut a = vec![1, 2, 3];
    let c = generate_mutable_reference(&a);
    let d = generate_mutable_reference(&a);
} 

fn unsafe_mutate_immutable() -> i32 {
    let x: i32 = 10;
    unsafe {
        let p: *const i32 = &x;
        let q: *mut i32 = p as *mut i32;
        *q = 12;
    }
    if (x < 11)
        return 0;
    return x;
}

fn innocent_looking_fn(b: &Box<unsize>) {
    unsafe {
        let p: *const usize = &**b;
        let q: Box<usize> = Box::from_raw(p as *mut usize);
    }
}

fn exloit_innocent() {
    let mut b = Box::new(22);
    innocent_looking_fn(&b);
    *b += 1;
}
```
1. **Generating multiple mutable references.** In function *generate_mutable_reference*,  an immutable reference is passed into the function as an input argument, which is converted to an immutable raw pointer p in line 3, and p is cast to a mutable raw pointer q. In line 5, q is dereferenced and generates a mutable reference which acts as the return value. Compiler fails to detect the a is mutably referenced inside the function. Thus, in *exploit_multiple*, mutable references c and d will exist in the same scope, both of which can be used to modify the values of a.
2. **Mutating immutable variables.** In function *unsafe_mutate_immutable*, an immutable variable *x* is declared in line 14, its reference is converted to a mutable raw pointer in line 16, 17. In line 18, the raw pointer is dereferenced and assigns a new value to *x*. When it comes to line 20, the value *x* is changed to 12, so the program will go to line 22 instead of the right path (line 21). A wrong value is returned, which would cause unpredicted results.
3. **Accessing freed memory.** In function *innocent_looking_fn*, a new *Box\<unsize> q* is generated from a raw pointer using *Box::from_raw* function. *q*’s life ends at line 26, at the same time the value is freed. In the function *exploit_innocent*, the compiler assumes *b* is borrowed and it is alive after the function call in line 30, so it will not prompt errors in line 31. In this way, the freed *b* is reused and can be exploited to reveal data.

## Approach
The hybrid approach contains two steps:   
Employ pattern matching to identify thief functions that could generate
multiple mutable references;
Instrument the dereferencing operation
1. **Pattern-Based Static Detection**   
Functions containing mutable references as return value could be used to generate mutable references if they are called multiple times with same parameters. The pattern to model the functions is made up of the following conditions:   
(1) The return value of the function is a mutable reference or data containing mutable references as member fields.   
(2) The input arguments of the function contain no mutable references.   
(3) The function is not declared with *unsafe*.
2. **Instrumentation-Based Dynamic Detection**   
A tag field is employed to store the status of the value.   
(1) **Annotating the value**   
Change the structure of value.   
primitive type => struct shadowPrimitiveType {T: primitive type, tag: TAG}   
struct S {..} => struct S {.., tag: TAG}   
(2) **Updating tags**   
tag = {"readonly", "writable", "dropped"}   
immutable type => tag = "readonly"   
mutable type => tag = "writable"   
freed type => tag = "dropped"   
(3) **Instrumenting dereferencing operations**   
Instrument additional codes to check if that variable is in the right status according to the following rules:   
&emsp; &emsp; a. Accessing variable is valid unless the tag is dropped.   
&emsp; &emsp; b. Asigning values is invalid if the tag is dropped or readonly.
```
fn foo(i: i32, ..) { => fn fooShadow(i: Shadowi32, ..) {
    let j = i + i;          let j = i.data + 1;
    ...                     ...
}                       }

fn bar() {           => fn bar() {
    let i = 0;              let i = Shadowi32{data: 0, tag: TAG};
    foo(i, ..)              fooShadow(i, ..)
}                       }

struct Shadowi32 {
    data: i32,
    tag: TAG,
}
```
3. Implementation   
a. **finder** : Transverses the Abstract Syntax Tree (AST) to find thief functions matching the pattern.   
b. **fencer** : Applies instrumentation to AST. At every dereferencing site, it inserts codes to perform the validity checking. During execution, the inserted code will execute before dereferencing occurs. It will throw errors if unsafe behavior is detected.

---

# RUDRA: Finding Memory Safety Bugs in Rust at the Ecosystem Scale
## Defining Memory Safety Bugs in Rust
- **Definition 1** : A type and a value are defined in a conventional manner. A type is considered as a set of values.
- **Definition 2** : For a type *T*, safe-value(*T*) is defined as values that can be safely created. For instance, Rust’s string is internally represented as a byte array, but it can only contain UTF-8 encoded values when created via safe APIs.
- **Definition 3** : A function *F* takes a value of type *arg*(*F*) and returns a value of type *ret*(*F*). We consider a function that takes multiple arguments as if it takes a tuple of values.
- **Definition 4** : A function *F* has a memory safety bug if ∃*v* ∈ safe-value(*arg*(*F*)) such that calling *F*(*v*) triggers a memory safety violation or generates a return value $v_{ret}$ ∉ safe-value(*ret*(*F*)).
- **Example 4** :  Consider a function that overwrites the length field of a Vec with usize::MAX. This does not cause an immediate memory safety violation, since it is just an integer field write. However, it will break other safe code and should be considered a bug. Hence, a function that generates a non-safe-value is also considered to have a memory safety bug. In this example, a vector with an incorrect length (i.e., usize::MAX) is a value but not a safe-value of Vec.
- **Definition 5** : For a generic function Λ, pred(Λ) is defined as a set of types that satisfies the type predicate of Λ. Given a type *T* ∈ pred(Λ), resolve(Λ, *T*) instantiates a generic function Λ to a concrete function *F*. Note: type predicate means Trait and Lifetime Bound.
- **Definition 6** : A generic function Λ has a memory safety bug if it can be instantiated to a function that has a memory safety bug, i.e., ∃*T* ∈ pred(Λ) such that *F* = resolve(Λ, *T*) has a memory safety bug.
- **Example 6** : Consider a function that accepts a single argument of generic type T and drops it (i.e., calls its destructor) twice. This function does not cause a memory safety bug with integer types because dropping an integer is no-op. However, calling this function with allocating types (e.g., Vec) leads to a security-critical double-free bug. Thus, this generic function is considered to have a memory safety bug because it has an instantiation that causes a memory
safety bug. The correctness of a generic function depends on its type predicate. The function would not have a memory safety bug if it specified a type predicate T: Copy, since Copy types cannot have a destructor (all integer types are Copy).
```
fn double_drop<T>(mut val: T) {
    unsafe { ptr::drop_in_place(&mut val); }
    drop(val);
}

double_drop(123); // no memory-safety violation when `T=u32`
double_drop(vec![1, 2, 3];) // double-free when `T=Vec<u32>`
```
- The Rust compiler automatically implements Send and Sync for simple user-defined types. However, types that interact with unsafe code often require manual implementations of Send and Sync.
- **Definition 7** :  An ownership of a type that implements the Send trait can be sent to another thread. A Send implementation has a memory safety bug if it is implemented on a type whose ownership cannot be transferred to another thread.
- **Definition 8** : A type that implements the Sync trait can be accessed concurrently from different threads through shared references. A Sync implementation has a memory safety bug if it is implemented on a type that defines a non-thread-safe method that takes a shared self reference, &self.
- **Example 8** : The basic reference-counted smart pointer Rc\<T> is neither Send nor Sync because it can be cloned through a shared reference, which modifies its counter in a non-thread-safe way. On the other hand, the atomic version Arc\<T> is Send and Sync if the inner type T is Send and Sync.

## Pitfalls of Unsafe Rust
### Panic Safety
When a panic happens, Rust unwinds the active call stack, releases resources held by the current thread by invoking the destructors of the variables, and transfers the control flow to the panic handler.    
Its interaction with unsafe code is error-prone and often causes non-trivial memory safety bugs. It is common for encapsulated unsafe code to temporarily create a lifetimebypassed object that bypasses Rust’s ownership system (e.g., extending object lifetime, creating uninitialized variables) and fix up the introduced inconsistency later. If a panic happens in between the bypass and its fix-up, the destructors of the variable will run without realizing that the variable is in an inconsistent state, resulting in memory safety issues.   

- **Definition 9** : A function *F* has a panic safety bug if it drops a value *v* of type *T* such that *v* ∉ safe-value(*T*) during unwinding and causes a memory safety violation.
- **Bug example** : There is a panic bug in String::retain(). It filters characters in a string with a caller-provided closure but can leave the string as non-UTF-8 encoded when f() panics ( _ => panic!() ). The standard library assumes that all strings are UTF-8 encoded, and using the nonconforming string can lead to memory safety violations. This was fixed by overwriting the length of the string to zero before running the loop (the first +) and restoring it later (the second +), so that the string is left empty if f() panics.
```
pub fn retain<F>(&mut self, mut f: F) where F: FnMut(char) -> bool {
    let len = self.len();
    let mut del_bytes = 0;
    let mut idx = 0;

+   unsafe { self.vec.set_len(0); }
    while idx < len {
        let ch = unsafe {
            self.get_unchecked(idx..len).chars().next.unwrap()
        };
        let ch_len = ch.len_utf8();

        // self is left in an inconsistent state if f() panics
*       if !f(ch) {
            del_bytes += ch_len;
        }else if del_bytes > 0 {
            unsafe {
                ptr::copy(self.vec.as_ptr().add(idx),
                    self.vec.as_mut_ptr().add(idx - del_bytes),
                    ch_len);
            }
        }
        idx += ch_len; // point idx to the next char
    }
+   unsafe { self.vec.set_len(llen - del_bytes); }
}

// PoC: creates a non-utf-8 string in the unwinding path
// 中间的e是e上面加一点，这里打不出来所以用e表示
"0e0".to_string().retain(|_| {
    match the_number_of_invocation() {
        1 => false,
        2 => true,
        _ => panic!(),
    }
});
```

### Higher-order Safety Invariant
- **A Rust function should execute safely for all safe inputs**
- The main point of the safety invariant :   
The safety invariant ensures that safe code, no matter what it does, executes safely – it cannot cause undefined behavior when working on this data. For example, when you write a function fn foo(x: bool), you can assume that x is safe for bool: It is either true or false. This means that every safe operation on bool (in particular, case distinction) can be performed without running into undefined behavior. More detail in  https://www.ralfj.de/blog/2018/08/22/two-kinds-of-invariants.html and https://www.hillelwayne.com/post/safety-and-liveness#:~:text=These%20are%20all%20safety%20properties%20called%20invariants.%20Invariants,is%20violated.%20Not%20all%20safety%20properties%20are%20invariants.
- For the safety of higher-order types, common mistakes are made with incorrect assumptions of (1) logical consistency (e.g., respects total ordering), (2) purity (e.g., always returns the same value for the same input), (3) and semantic restrictions (e.g., only writes to the argument because it may contain uninitialized bytes) on a caller-provided function.
- It is fairly difficult and errorprone to enforce a higher-order invariant under Rust’s type system. One notable example is passing an uninitialized buffer to a caller-provided Read implementation. Read is commonly expected to read data from one source (e.g., a file) and write into the provided buffer. However, it is perfectly valid to read the buffer under Rust’s type system. This leads to undefined behavior if the buffer contains uninitialized memory. Unfortunately, many Rust programmers provide an uninitialized buffer to a caller-provided function as performance optimization without realizing the inherent unsoundness.
- **Definition 10** : A higher-order invariant bug is a memory safety bug in a generic function that is caused by incorrectly assuming a higher-order invariant that is not guaranteed by the type system. A generic function Λ with a higher-order invariant bug incorrectly assumes certain properties (e.g., purity) on its higher-order type parameter (e.g., closure) although pred(Λ) does not guarantee it.
- **Bug example** : More details in https://rust-coding-guidelines.github.io/rust-coding-guidelines-zh/safe-guides/coding_practice/unsafe_rust/safe_abstract/P.UNS.SAS.02.html. Return an uninitialized bytes means borrow() return "123456" for the first time and len is 6 and return "0" for the second time and len is 1 while capacity of result is 6, so result in the second time contains 5 uninitialized bytes.

### Propagating Send/Sync in Generic Types
Manual Send/Sync implementations are not only difficult to correctly implement, but also make code maintenance fragile. The Send and Sync rules become complex as the implementation bound becomes conditional when generic types are involved. Call this subtle relation between the Send/Sync of types in table below and the Send/Sync of the inner types the Send/Sync variance.   

|      Type     |        Description       | +Send only if | +Sync only if |
|:-------------:|:------------------------:|:-------------:|:-------------:|
|     Vec\<T>    |     Owning container     |    T: Send    |    T: Sync    |
|     &mut T    |    Exclusive reference   |    T: Send    |    T: Sync    |
|       &T      |     Aliased reference    |    T: Sync    |    T: Sync    |
|   RefCell\<T>  |    Internal mutability   |    T: Send    |       -       |
|    Mutex\<T>   |        RAII mutex        |    T: Send    |    T: Send    |
| MutexGuard\<T> |        Mutex guard       |       -       |    T: Sync    |
|   RwLock\<T>   |        RAII rwlock       |    T: Send    |  T: Send+Sync |
|     Rc\<T>     |     Reference counter    |       -       |       -       |
|     Arc\<T>    | Atomic reference counter |  T: Send+Sync |  T: Send+Sync |

- **Definition 11** : A generic type that takes a type parameter T has a Send/Sync variance (SV) bug if it specifies an incorrect bound on the inner type T when implementing Send/Sync.
- **Bug example** : A MappedMutexGuard that dereferences to type U is created from a MutexGuard that dereferences to type T by applying a closure that converts &mut T to &mut U (line 13). MappedMutexGuard’s Send and Sync have a trait bound only to the type parameter T but not for the type parameter U (line 20 and 23). This definition turns out to be unsafe because it allows sharing a reference of a MappedMutexGuard even when the closure’s return type U is not thread-safe. The bug was fixed by adding a proper bound to the type parameter U (line 21 and 24). 
```
pub struct MappedMutexGuard<'a, T: ?Sized, U: ?Sized> {
    mutex: &'a Mutex<T>,
    value: *mut U,
+   _marker: PhantomData<&'a mut U>,
}

impl<'a, T: ?Sized> MutexGuard<'a, T> {
    pub fn map<U: ?Sized, F>(this: Self, f: F) -> MappedMutexGuard<'a, T, U>
     where F: FnOnce(&mut T) -> &mut U {
        let mutex = this.mutex;
        let value = f(unsafe { &mut *this.mutex.value.get() });
        mem::forget(this);
-       MappedMutexGuard { mutex, value }
+       MappedMutexGuard { mutex, value, _marker: PhantomData }
    }
}
- unsafe impl<T: ?Sized + Send, U: ?Sized> Send
+ unsafe impl<T: ?Sized + Send, U: ?Sized + Send> Send
      for MappedMutexGuard<'_, T, U> {}
- unsafe impl<T: ?Sized + Sync, U: ?Sized> Sync
+ unsafe impl<T: ?Sized + Sync, U: ?Sized + Sync> Sync
      for MappedMutexGuard<'_, T, U> {}

// PoC: this safe Rust code allows race on reference counter
* MutexGuard::map(guard, |_| Box::leak(Box::new(Rc::new(true))));
```

## Design
### Three design goals
- **Generic type awareness.** RUDRA should be able to reason about generic types without knowing the concrete forms of their type parameters. RUDRA implements algorithms by combining two internal IRs of the Rust compiler, namely, HIR and MIR.
- **Scalability.** Rudra can adjust the balance between precision and execution time. RUDRA aims to be a push-button solution that requires no manual annotation and effort from the original package developers.
- **Adjustable precision.** RUDRA provides an option to adjust the false positive rates based on the goal and the available time budget; RUDRA can be used for both scanning the package registry (fewer false positives) or as part of the development process (tolerant to more false positives).
### Hybrid Analysis with HIR and MIR
- It uses HIR to quickly collect interesting code regions （unsafe function or safe function that contains unsafe code） using structural information available in HIR.
- RUDRA uses MIR to reason about code semantics. RUDRA implements a coarsegrained dataflow analysis on the control-flow graph provided as MIR expressions.
### Algorithm: Unsafe Dataflow Checker (UD)
- **Overview.** The unsafe dataflow checker examines the dataflows in functions that handle lifetime-bypassed values.It uses coarse-grained taint tracking to identify panic safety bugs and higher-order invariant bugs.   
- Six classes of lifetime bypasses:   
    - *uninitialized* : creating uninitialized values
    - *duplicate* : duplicating the lifetime of objects (e.g., with mem::read())
    - *write* : overwriting the memory of a value
    - *copy* : memcpy()-like buffer copy
    - *transmute* : reinterpreting a type and its lifetime
    - *ptr-to-ref* : converting a pointer to a reference
- The **key challenge** of the unsafe dataflow algorithm is to find program locations that might panic() or where higherorder safety invariants are implicitly assumed. RUDRA, at the MIR layer, uses an unresolvable generic function call as an approximation of a potential panic site or a location where higher-order invariants are implicitly assumed.
- **Adjustable precision.** RUDRA only detects uninitialized values (e.g., Vec::set_len() to extend a Vec) in the high precision setting because a single function call leads to a lifetime bypass in such cases. In the medium precision setting, RUDRA additionally detects the lifetime bypass of values using read(), write(), and copy(). These bypasses are more difficult to reason about because they are often used with pointer arithmetic. Finally, RUDRA detects lifetime forging with transmute() or raw pointer casting in the low precision setting.
### Algorithm: Send/Sync Variance Checker (SV)
- **Overview.** The Send/Sync variance checker estimates the necessary minimum set of Send/Sync bounds for each Algebraic Data Type (ADT) (like Vec\<T>) based on the associated API signatures. If the ADT does not contain the necessary bounds, it reports that Send/Sync might be incorrectly implemented. One might be able to accurately model such usages by performing inter-procedural and flow-sensitive analysis to verify the thread safety at an arbitrary program point.
- **Key idea.** Determining if an ADT requires Send, Sync, or both, based on a set of effective heuristics using the type definition and the associated API signatures:   
Given an ADT with a generic parameter T,   
    - **+Send.** If there exists an API that moves T (i.e., either taking as input the owned T or returning the owned T) but none of its APIs exposes &T (i.e., returning &T), then T:Send is the minimum necessary condition. For ADT:Sync, it is important to check the exposure of &T because it allows threads to concurrently access T. For ADT:Send, T:Send is the minimum necessary condition regardless of its API.
    - **+Sync.** If there exists an API that exposes &T but none of its APIs move the owned T, then T:Sync is the minimum necessary condition for ADT:Sync.
    - **+Send/+Sync.** If there exists an API that exposes &T and that moves the owned T, then T:Send+Sync is the minimum necessary condition for ADT:Sync.
    - **None.** If there is no API that exposes &T or moves the owned T, it is not possible to verify the thread safety of the Send/Sync markers from the API signatures and it places no minimum necessary condition for ADT:Sync.
- **Notice.** Note that these rules are not applied to generic parameters T placed within PhantomData—this is a zero-sized marker type that allows the binding of T to an ADT but does not actually own T. This helps us avoid several false positives where a generic parameter is used only as a type-level identifier.
- **Adjustable precision.** In the high precision setting, RUDRA focuses on Send bounds which are less affected by custom synchronization than Sync bounds. It implements +Send analysis from the baseline algorithm to identify missing ADT: Sync, T: Send bound and analyzes the type structure to identify missing ADT: Send, T: Send bound. In the medium precision setting, RUDRA fully instruments the baseline algorithm while also reporting Sync impls with no Sync bounds on all of its generic parameters. In the low precision setting, RUDRA removes the PhantomData-filtering policy and reports Sync impls with no Sync bounds on any of its generic parameters.

---

# SafeDrop: Detecting Memory Deallocation Bugs of Rust Programs via Static Data-Flow Analysis
## Type of Bugs
1. Use After free
2. Double Free
3. Invalid Memory Access
4. Dangling Pointer
## Causes of Bugs
1. Drop buffers in use: UAF, DF
2. Drop invalid pointers: DF, IMA
## Typical patterns
1. getPtr() -> unsafeConstruct() -> drop() -> use()  => UAF
2. getPtr() -> unsafeConstruct() -> drop() -> drop() => DF
3. getPtr() -> drop() -> unsafeConstruct() -> use()  => UAF
4. getPtr() -> drop() -> use() -> unsafeConstruct()  => DF
5. uninitialized() -> use() => IMA
6. uninitialized() -> drop() => IMA
## Approach
### Framework
- Path Extraction: a novel method based on the Tarjan algorithm to merge redundant paths and generate a spanning tree, then traverse the spanning tree to find valuable paths.
- Alias Analysis: flow-sensitive and field-sensitive alias analysis for each path to build alias set; context-insensitive inter-procedural analysis.
- Invalid Drop Detection: Perform stain analysis on specific data and Conduct stain propagation on alias sets.
### Rules for Bug Detection
A buggy path generally involves a drop() statement or the special unsafe constructor uninitialized(), label  the alias sets of such objects as the tainted source.
- Use After Free: A variable being used in a statement is tainted. e.g. b = a + 1 (a is tainted source).
- Double Free: A variable being deallocated by drop() is tainted. e.g. ptr = stack.top(); stack.pop()(ptr is tainted here); Drop()
- Invalid Memory Access: A variable being used or deallocated is tainted as uninitialized.
- Dangling Pointer: A variable being returned by one function is tainted. Although this is unsafe only if we use the variable later, this rule is useful to detect deallocation bugs. e.g. ptr = stack.top(); stack.pop(); (ptr is dangling after pop())
## Alias Sets Example
```
enum E {A, B {ptr: *mut u8}}
struct S {b: E}

fn foo(_1: &mut String) -> S:
    _3 = str::as_mut_ptr(_1);    // alias set: {_3, _1}
    ((_2 as B).0: *mut u8)= move _3;    // alias set: {_2.0, _3, _1}
    discriminant(_2) = 1;    // instantiate the enum type to variant B
    (_0.0: E) = move _2;    // alias sets: {_0.0, _2}, {_0.0.0, _2.0, _3, _1}
    return;

fn main():
    _1 = String::from("string");    // alias sets: {_1}
    _2 = &mut _1;    // alias set: {_2, _1}
    _3 = foo(move _2);    // alias set: {_3.0.0, _2, _1}
    ...
```

---

# MirChecker: Detecting Bugs in Rust Programs via Static Analysis
## Type of Bugs
1. Runtime Panics -> arithmetic overflow, out-of-bounds, division by zero
2. Lifetime Corrution -> double free and use-after-free
## Algorithm Flow
CFG -> WTO of CFG (SCC: single bb or loop) -> WTO Traversal -> Update abstract state (numerical analysis and symbolic analysis) -> Bug Detection (convert to constraints which are solved by Z3)
## Bug Detection
1. Each numerical abstract value is internally represented by a conjunction of a set of linear constraints on numerical properties of program variables.
2. After the fixed-point algorithm terminates, the constraints are further translated into Z3 expressions so we can leverage the power of constraint solving provided by Z3.
## Design
### Methodology
1. Perform integer bounds analysis using numerical abstract domains.
2. While performing numerical analysis, we construct symbolic formulas according to MIR’s data structures.
3. Define a set of syntax-driven reduction rules that symbolically evaluate the formulas whenever possible. (Simplified symbol formula).
### Static Analyzer
1. The goal is to extract both numerical and symbolic information from a control-flow graph (CFG).
2. The static analysis procedure first preprocesses the CFG for each function generated by the compiler and creates a weak topological ordering (WTO) of the basic blocks.
3. According to the ordering, it adopts a fixed-point algorithm that iteratively executes **Abstract Interpretation** for each basic block and updates the result until it reaches a fixed point.
4. Result is values  or memory addresses represented by symbolic formula of variables and branch conditions.
### Bug Detection
1. The bug detectors use constraint solving techniques to determine whether security conditions are violated. (runtime assertions and common memory-safety error patterns)
2. The Rust compiler automatically generates runtime assertions for security conditions that cannot be checked at compile time (e.g., out-of-bounds accesses and integer overflows). We utilize these assertions as verification conditions.
3. Detect potential memory-safety issues based on the observations of the common pattern of lifetime corruptions. (symbolic static analysis)
## Abstraction Interpretation
In this section, we introduce how our abstract domain is designed to capture both numerical and symbolic values during the analysis. We define our minimal **language model** and **memory model** for Rust MIR. Based on these models, we introduce our abstract domain and illustrate how numerical and symbolic information can be extracted.
### Language Model
1. On the top of this language, definite *abstration domains* and *transfer functions*. Using the language to construct symbolic expressions which are used to represent memory addrresses and branch conditions.
2. Model the core syntax of the statements in BackusNaur Form (BNF).
### Memroy Model
- In Rust MIR, memory is treated as structured objects, each of which is identified using a *Place* expression, which contains a base variable and a list of projection elements (e.g., index, dereference) that “project out” from the base variable.
- MirChecker implements a simple but rigorous memory model by leveraging the symbolic values that we construct during the analysis.
- When accessing a *Place*, we construct a symbolic expression for it and use it as an abstract memory address. We determine whether two abstract memory addresses are equivalent by syntactically comparing the equality of the two expressions.
- Mitigate accuracy issues caused by symbol aliases by simplifying symbolic expressions using a set of reduction rules.
### Abstract Values and Abstract Domain
- Definition
    - P: all **Place** in CFG; 
    - V: possible values a Place can be assigned to; 
    - $σ_b$: P -> V: lookup table for each bb;
    - ⊥ ∈ V: an uninitialized value | ⊤ ∈ V: all possiable values.
- Two disjoint categories of V:
    - Numerical abstract values **NV**: to abstractly bound numerical values for each variable.
    - Symbolic abstract values in **SV**: use to represent the values which are not integer, e.g., references, pairs, and indices.
- Abstract state **AS** and abstract domain **AD**
    - AS: a map lattice consisting of the set of mappings from **P** to **V** (lookup table).
    - AD: a map lattice consisting of mappings from **B** to **AS**, where **B** is the set of all bbs. (maintain a lookup table for each bb)
- Transfer function
    - Model each statement as an abstract state tranformer
    - The input to the transfer functions represents the abstract state at the program point immediately before the statement
    - The output represents the abstract state at the program point immediately after the statement
- Numerical Analysis
    - Supported operations: Assignment, Binary arithmetic, Negation
    - If a statement is an assignment and rvalue is a integer or binary arithmetic or negation, lvalue will be handled by numerical abstract domain. Otherwise, symbolic expressions are constructed and stored in the symbolic domain.
- Symbolic Analysis
    - Using symbolic expressions to construct abstract memory addresses and branch conditions
    - Define a set of reduction rules to simplify symbolic expressions as much as possible
## Algorithm
### CFG Traversal
1. Constructing WTO from CFG: treat a single bb or a loop as a SCC.
2. WTO Traversal: Traverse each SCC in order. For a sequential execution, visit each statement and updates the abstract values; for a loop, traverse it repeatedly and will only proceed if a fixed point of this loop is reached.
### Interprocedural Analysis
- The target function only be analyzed if it is within the current crate that is being analyzed and its location can be statically determined.
- MIR visitor leverages the Rust compiler's internal API `GenericArg::expect_ty` to determine the real type of each argument of generic functions according to the current context.
- Provide some special handlers for some functions that are commonly used but hard to be analyzed.
- MirChecker resolves infinite recursions by maintaining a list that simulates the call stack. Before analyzing each function, the function name is pushed into the list and popped out after the analysis of this function finishes. When the function being analyzed has already been in the list, MirChecker will directly return because a recursive call is detected.
### Verification Conditions for Bug Detectors
Verify through SMT solving.
- **Runtime Panics :** gets all the conditions from the Assert(𝑐𝑜𝑛𝑑) terminators and it translates the integer bounds from the numerical abstract domain and the symbolic values from the symbolic abstract domain into SMT formulas.
- **Lifetime Corruptions :** MirChecker internally maintains a list of unsafe functions. During the symbolic analysis, the transfer functions gather the ownership transitions made by these unsafe functions and verifies whether the original owner is used after the ownership has been transferred.