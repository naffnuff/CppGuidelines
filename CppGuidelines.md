# C++ Guidelines

This document focuses on the many subtle pitfalls and hazards that often plague C++ code. Decisions on formatting, naming and other arbitrary matters should be kept in a different document.

Many of the points below are discussed in more detail in the [Google Style Guide](https://google.github.io/styleguide/cppguide.html) and in the [C++ Core Guidelines](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

## General
Make it easy for the reader, not the writer
* *Reason:* We usually spend much more time reading code than writing it. Taking shortcuts will surely be a waste of time.

Avoid writing surprising or confusing code
* *Reason:* For every problem, there is probably an efficient standard solution.

Prefer readable over excessively “clever” coding
* *Reason:* Readability is key to maintainability and correctnes. Esoteric often leads to hard-to-detect errors down the road.

Use the `std` library whenever possible
* *Reason:* It's state-of-the-art.

Avoid functions from the C library, especially the ones that require manual resource management
* *Reason:* It's a thing of the past. Pretty much anything they do, `std` will do better and safer.

Clean up old code according to these guidelines as far as reasonably safe and manageable

## Resource management
Don’t use naked `new` or `delete` (or their C counterparts, including functions that allocate memory that the user must free manually).
* *Reason:* It is virtually impossible to write robust code using manual resource management. Even when carefully done, manual resource management ruins exception safety (and error safety in general).

Instead, manage memory automatically or in containers using the [RAII](https://en.cppreference.com/w/cpp/language/raii) pattern (from the std library when available), that are themselves automatically managed. RAII should also be applied for other resources, such as file handlers.
* *Reason:* For automatically managed objects, the destructor is guaranteed to be run when the object goes out of scope. We exploit this by freeing resources in the destructor.

## Smart pointers
Only use smart pointers where there is an actual ownership. Arguments should usually be passed by reference (or pointer if `nullptr` is an acceptable value).
* *Reason:* Smart pointers can be a result of sloppy design, especially shared pointers. If you're using C++ like Java, then you probably made a big mistake picking C++ in the first place. They come at a cost, so avoid them when possible.

Use `std::shared_ptr` when shared ownership of an object is necessary. If it makes more sense with single ownership of an object, use `std::unique_ptr`, and transfer that ownership with `std::move()`.
* *Reason:* Shared pointers add complexity and overhead, and are often unnecessary. `std::unique_ptr` also expresses a guarantee that ownership will not be shared.

## Exceptions
In general, exceptions are a good way of handling errors.
* *Reason:* Exceptions leads to less code devoted to error handling, and mostly frees the return value from being a mere error code. Also, exceptions are automatically propagated up in the call hierarchy in a way error codes are not, and may thus more easily be handled in the right scope. Note that the exception safety is just a specific case of error safety; avoiding exceptions does not solve that problem.

All exceptions we use should implement `std::exception`. `std::runtime_exception` is the go-to implementation.
* *Reason:* In order to be sure that we catch all exceptions, they must have a common superclass. As always, we prefer `std` types when available.

Everything that can be affected by an exception must be exception-safe. Note that specifically this means that the [RAII](https://en.cppreference.com/w/cpp/language/raii) pattern is used. All std containers follow RAII and are thus exception-safe.
* *Reason:* 

Don't use `throw`. Declare `noexcept` where appropriate; all other functions should be expected to throw.
* *Reason:* It is very hard and cumbersome to cover every exception that could be thrown from a function. It is more manageable to see throwing behavior as the rule, and `nothrow` as the exception.

When changing old systems, keep exception safety in mind, and don’t assume anything.
* *Reason:* Exceptions make the issue of error safety more manageble, since there is something concrete to do about it. Code will have to be converted very carefully, small sections at a time, probably starting in the leaves of the call hierarchy, going up.

## Declarations
Declare const where appropriate
* *Reason:* `const` gives an important indication of intent and what to expect, that makes the code more maintainable. It can be very difficult to add as an afterthought.

## Arguments
Pass complex types by pointer when `nullptr` is an acceptable value, otherwise by reference. Add `const` when appropriate.
* *Reason:* There is usually no point in passing an argument of complex type by value; a `const &` will be equivalent from the caller's perspective. There is also no point in passing a smart pointer, unless there is an actual transfer or sharing of ownership involved.

## Static variables (including global)
Static variables should only be primitives or constexpr. If for some reason there is need for more, be aware of the following:
* Static variables (and their members and so on) must be trivially destructed (or dynamically allocated)
* Non-local static variables must not be dynamically initialized
* Consider the possibility that a class that now fulfills the two previous points may not do so in the future

* *Reason:* Static memory is initialized in arbitrary order at process startup, and destructed in reverse order of initialization at process shutdown. It is therefore virtually impossible to know in the constructor or destructor if other resources are initialized or destroyed, unless those resources are compile-time constants. `constrexpr` ensures that an expression is evaluated at compile time, and is therefore not affected by static initialization and destruction.

## Classes and structs
Constructors should not call virtual functions of the same class. The constructor should be kept fairly simple.
* *Reason:* The type hierarchy is constructed top-down. If a virtual function is overloaded, it will be called on an object that is not yet constructed.

## Default functions/constructors/destructor
Use = default or = delete if there is no custom definition. Especially, be explicit about movability and copyability.
* *Reason:* Types are copyable and moveable by default, but if there is no explicit declaration, there is good reason to suspect that creator of that type has not thought about what it means for an object of that type to be copied. Moving an object is usually a more efficient operation than copying it, that is used in those cases when the old object is guaranteed to be unused, such as when returned from a function, but it also makes sense to have some classes as moveable but not copyable, as for example with `std::unique-ptr`.

## Casts (not including conversion casts)
Minimize casts
* *Reason:* Casts are usually necessary due to design flaws. Casts always introduce a risk of runtime error. `dynamic_cast` is safer, but a better alternative to it is usually [dynamic dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch).

For class types, use the C++-cast that expresses your intent
* *Reason:* C casts are equal to a C++ `reinterpret_cast`. Using them is to bypass all compile-time type checking. Error of reasoning from the programmer may lead to hard-to-detect runtime errors. `const_cast` and `static_cast` have more limited purposes and are therefore safer (but see previous point).

* *Note:* C++ is full of implicit casts that are transparent and require a lot of understanding from the programmer; these are unfortunately all C casts. 

## Inheritance
Don’t use multiple *implementation* inheritance (multiple interface inheritance is fine)
* *Reason:* It is not safe to upcast to a superclass with for example an implicit cast, if that superclass is part of a multiple inheritance. There is also the risk of the [diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem).

Don’t overuse implementation inheritance.
* *Reason:* Spreading out logic over several classes makes the code harder to read. Object-orientation is not the solution to every problem. Templates are usually a more apt technique for algorithm generalization.

Always specify `override` (or `final`) when overriding. (There is no need to declare the overriding function `virtual`.)
* *Reason:* This is the only way of expressing that a function is an override, which is often very useful information.

## Operator overloading
Do not overload `&&`, `||`, `“”`, `,` or unary `&`
* *Reason:* `&&`, `||` and `,` cannot match the evaluation-order semantics of the built-in operators. `""`, declaring your own literals, may make the code confusing.

Operator semantics should follow convention
* *Reason:* The operators have a special place in the language and strong semantic connotations. Breaking convention will probably confuse and possible hide errors.

Define non-modifying binary operators as non-member functions
* *Reason:* If defined as members, the first operand will not follow the same implicit conversion rules as the second operand.

* *See:* Function overloading

## Function overloading
Overload only when it is not necessary for the reader to know which overload will be called (there are no semantic differences)
* *Reason:* The rules of overloading are tricky, especially in combination with inheritance. They depend on implicit type casts, that are hard to anticipate, that may also change as the types of arguments are modified.

## Default arguments
Do not use default arguments on virtual functions
* *Reason:* If the overrides declare different default arguments, the default used is determined by the static type of the object, subverting the whole idea of polymorphism (the feature is essentially broken).

Only use default arguments when their values are guaranteed to always be the same (beware of type conversions).
* *Reason:* Default arguments are re-evaluated at each call site, which can be extremely confusing if their value is unpredictable.

When in doubt, use overloads
* *See:*  Function overloading

## Integer types
Use only int of the built-in integer types
* *Reason:* The built-in integer types are not standardized. If there is need for a certain size, use the definitions from `stdint.h`, such as `int64_t`.

Avoid unsigned unless that is the required behavior (bitfields and modulo overflow). In particular, do not use unsigned to assert that a variable is non-negative.
* *Reason:* Underflow of unsigned integers will easily lead to errors.

Prefer iterators and range-based for loops to indexing and size comparisons.
* *Reason:* The `std` library uses the unsigned type `size_t` for all their containers (many think this is a mistake). Comparing signed and unsigned integers leads to tricky problems. Also, there is a clearer expression of intent and better encapsulation with iterators and range-based for loops.

## `auto` declaration
Good to use for iterators that are long and cumbersome and similar cases.
* *Reason:* Some types are not important enough to justify a long declaration.

OK when the type is either unimportant or clear from local context.
* *Reason:* When the type is obvious without close inspection, `auto` is OK if it helps readability.

May be a good idea for some complicated types.
* *Reason:* Some types, especially templated ones, are hard to figure out, and the programmer risks getting the type wrong. Unintended copying or type conversions may then occur. In those cases, `auto` may be justified, in combination with appropriate naming.

Discouraged in all other cases, such as data members and larger contexts
* *Reason:* In most cases, types are important information. Hiding them reduces readability.

## Lambda expressions
Use only explicit capture when lambas escape the current scope, both in regard to by-reference and by-value captures. It may be a good idea to avoid default capture alltogether except in very short lambdas.
* *Reason:* Capture by reference, and by value in the case of pointers, means there is a shallow copy of each of those captures. When the lambda has a longer lifetime than the captured variable, the capture is the equivalent of a [dangling pointer](https://en.wikipedia.org/wiki/Dangling_pointer). Especially, this can be tricky in the case of `this`, which is automatically captured by both by-reference and by-value default captures, since this is implicitly used when member functions are called.

## Namespaces
Do not use `using` directive to make a whole namespace available, except for project-local namespaces
* *Reason:* Working with `using` directives can easily cause confusion as to what code is actually used.

Use anonymous namespaces for file-private definitions
* *Reason:* Anonymous namespaces (as well as `static`) enables internal linkage, meaning they cannot be accessed or defined in another file. This makes the code safer, and also linking time is reduced.

WIP: Use namespaces (how is undecided)
* *Reason:* Namespaces prevents name collisions, both in compile time and in runtime.
