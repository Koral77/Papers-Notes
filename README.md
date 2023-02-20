# Papers-Notes
## Notes List
- [SyRust: Automatic Testing of Rust Libraries with Semantic-Aware Program Synthesis](#syrust-automatic-testing-of-rust-libraries-with-semantic-aware-program-synthesis)


--- 

## SyRust: Automatic Testing of Rust Libraries with Semantic-Aware Program Synthesis
**The issues to be solved**
1. **Basic Typing and Subtyping Constraints** Rust is statically typed: all variables are assigned a type at compile time and all uses of variables are consistent with their types. An type should be matched with its reference and an immutable reference should be matched with mutable reference, but not the other way around.
2. **Polymorphism and API Specification Refinement** To generates test cases for APIs with polymorphic types, the algorithm needs to know how the type variable T is instantiated and handle subsequent typing constraints(traits). Defining a notion of API refinement: a type signature that permits fewer inputs is considered more refined. For example, Vec\<String> is more refined than Vec\<T>, as the former permits only vectors of strings and the latter permits more types.
3. **Variable Ownership and Lifetime** The uniqueness of the ownership of a vaule must hold across the program.  A lifetime of a variable is a region in the code in which this variable can be used. If a variable is used, then the compiler ensures that this usage occurs within the lifetime of the variable.
4. **Relationship Between Lifetimes** If a variable b is a reference to another varabile a, then b's lifetime depends on a. That is, the lifetime of b must be strictly contained in the lifetime of a.
5. **Borrowing and Refenrences** To prevent data races, Rust enforces the rule that only one mutable reference can be active for any location in memory. The compiler will reject any attempts to borrow a mutable reference to a memory location while another mutable reference is active. Furthemore, mutable and immutable references for one memory location cannot co-exist, but a memory location can have many immutable only references.

**Methodology**

Nine rules are used to solve the issues above.
1. **Basic Typing constraints**
> - Rule 1 states that the available variables and types at the start of the synthesis are the ones in the template T.
> - Rule 2 describes if x is defined in line i, then the type of x in line i + 1 is same as in line i and the types of variables belonging to the intersection of the domains of context(i) and context(i + 1) remain unchanged.
> - Rule 3 defines type checking for a multi-line program. 
>> 1. Single-Line Type Check Definition: Let f: t1 × ... × tk → t' and line i contain a call to f(v1, ..., vk), line i containing f types checks if it satisfies the conditions that v1, ..., vk ∈ dom context(i) and types of all variables in dom context(i) ⊑ all types from the template and API signature and a polymorphic match is compatible for every pair of polymorphic variables. 
>> 2. Multi-Line program Type Check Definition:  Every single line of code in the program satifies the conditions in Single-Line Type Check Definition.
2. **Ownership and Variable Lifetime**
> - Rule 4 makes sure that the types must be a primitive type or static referece (&) if the variables are the same.
> - Rule 5  used to terminate non-primitive variable lifetimes upon use.
> - Rule 6 states that the lifetime of reference &v is included in the variable v.
> - Rule 7  keeps the lifetime relation even when the types change and content is moved out.
> - Rule 8 promises while &mut is active, prevent another &mut and &. 
> - Rule 9 promises while & is active, prevent another &mut.

**Hybrid API Refinement**

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
