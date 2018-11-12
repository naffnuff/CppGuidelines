# Utopia C++ Guidelines

This document focuses on the many subtle pitfalls and hazards that often plague C++ code. Decisions on formatting, naming and other more or less arbitrary choices should be kept in a different document.

Many of the points below are discussed in more detail in the [Google Style Guide](https://google.github.io/styleguide/cppguide.html) and in the [C++ Core Guidelines](http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines).

## General
Make it easy for the reader, not the writer.
* *Reason:* We usually spend much more time reading code than writing it. Taking shortcuts will surely be a waste of time.

Avoid writing surprising or confusing code.
* *Reason:* For every problem, there is probably an efficient standard solution.

Prefer readable over excessively “clever” coding.
* *Reason:* Readability is key to maintainability and correctnes. Esoteric code often leads to hard-to-detect errors down the road.

Use the `std` library whenever possible.
* *Reason:* It's the result of a peer-reviewed process involving some of the sharpest minds in academic computer science. And it is specifically designed to be useful for pretty much any code you will write.

Avoid functions from the C library, especially the ones that force us into manual resource management.
* *Reason:* They make the code base more error-prone and less robust. Pretty much anything they do, `std` does better and safer.

Clean up old code according to these guidelines as far as reasonably safe and manageable.
* *Reason:* It's an investment that will save us time and misery in the long run.

## Resource management
Prefer automatic memory management. Only allocate on the heap when absolutely necessary.
* *Reason:* Scopes in C++ are a very powerful tool for sound structure in the code. The lifetime of objects usually have a natural place in that scoped structure. Following this structure not only creates order, but also serves us error-, hassle-, and waste-free memory management. We should use it as much as we can.

Don’t use naked `new` or `delete`. The same principle holds for other resources, such as file handlers, sockets and locks.
* *Reason:* It is virtually impossible to write structured and robust code using manual resource management. Even when carefully done, manual resource management ruins exception safety (and error safety in general).

Instead, manage resources automatically or in containers using the [RAII](https://en.cppreference.com/w/cpp/language/raii) pattern (from the `std` library when available), that are themselves automatically managed. RAII should also be applied for other resources.
* *Reason:* Dynamically allocated memory can be automatically managed by extension. For automatically managed objects, the destructor is guaranteed to be run when the program leaves the scope of the object. We exploit this by freeing resources in the destructor.

## Raw pointers and references
Raw pointers (`A* a`) and references (`A& a`) express non-ownership (except in RAII containers).
* *See:* Resource management and Smart pointers.

Let raw pointers only point to single objects, as opposed to arrays.
* *Reason:* There is really no need for built-in arrays anymore. Use `std::vector`, `std::string` or, for a thin abstraction of the built-in array, `std::array`.

Polymorphic objects must be held by (smart) pointer or reference, not by value.
* *Reason:* For polymorphic objects, value semantics causes [object slicing](https://en.wikipedia.org/wiki/Object_slicing). That is, copying is done using only static type information, and dynamic data is thereby corrupted and/or leaked.

## Smart pointers
Only use smart pointers where there is an actual ownership. Arguments should usually be passed by reference (or pointer if `nullptr` is an acceptable value).
* *Reason:* Smart pointers can be a result of sloppy design, especially shared pointers. If you're using C++ like Java, then you probably made a big mistake picking C++ in the first place. They come at a cost, so use them when necessary.

Prefer `std::unique_ptr`. Use `std::shared_ptr` only when shared ownership of an object is necessary. If it makes more sense with single ownership of an object (as is often the case), use `std::unique_ptr`, and transfer that ownership with `std::move()`.
* *Reason:* Shared pointers add complexity and overhead, and are often unnecessary. `std::unique_ptr` expresses a guarantee that there is a single ownership of an object, making a the code more efficient and easier to reason about.

## Exceptions
In general, exceptions are a good way of handling errors.
* *Reason:* Exceptions lead to less code devoted to error handling, and typically frees the return value from acting as an error code. Also, exceptions are propagated up in the call hierarchy in a way that enables error handling in the right scope. Note that exception safety is just a specific case of error safety; avoiding exceptions does not make code error-safe.

All exceptions we use should implement `std::exception`. `std::runtime_exception` is the go-to implementation.
* *Reason:* In order to catch all exceptions, they must have a common superclass. As always, we prefer `std` types when available.

Always catch exceptions by reference. Do not create exceptions with `new`.
* *Reason:* Adding dynamic memory management to exception handling is pointless and creates memory leaks. Catching by value would disable the polymorphism we require by using std::exception. Catching by pointer and by reference cannot be done by the same catch clause.

Everything that can be affected by an exception must be exception-safe. This means that the [RAII](https://en.cppreference.com/w/cpp/language/raii) pattern is used. All std containers follow RAII and are thus exception-safe.
* *Reason:* New standards make the issue of error safety manageble. There is no point in introducing exceptions if the code does not use the other modern constructs that go along with them. Also, many `std` functions may throw exceptions.

Don't use `throw`. Declare `noexcept` where appropriate; all other functions should be expected to throw.
* *Reason:* It is very hard and cumbersome to cover every exception that could be thrown from a function. It is more manageable to see throwing behavior as the rule, and `nothrow` as the exception. Also, `throw` is deprecated.

When changing old systems, keep exception safety in mind for all affected code.
* *Reason:* Code will have to be converted very carefully, small sections at a time, probably starting in the leaves of the call hierarchy, going up.

## Declarations
Declare `const` where appropriate.
* *Reason:* `const` gives an important indication of intent and what to expect, that makes the code more maintainable. It can be difficult to add as an afterthought.

## Arguments
Receive complex types by pointer when `nullptr` is an acceptable value, otherwise by reference; `const` when appropriate.
* *Reason:* There is usually no point in passing an argument of complex type by value; a `const &` will be equivalent from the caller's perspective. There is also no point in passing a smart pointer, unless there is an actual transfer or sharing of ownership involved.

## Static variables
* *Note:* Static class variables are essentially global variables, so the two will be referred to as "non-local static variables" below.

Non-local static variables should always be `const`.
* *Reason:* Modification of a non-local static variable is a completely unstructured way of passing information. As such, it is a common and hard-to-trace source of error.

Static variables should only be primitives or `constexpr`. If for some reason there is need for more, be aware of the following:
* Static variables (and their members and so on) must be trivially destructed (if not dynamically allocated).
* Non-local static variables must not be dynamically initialized.
* Consider the possibility that a class that now fulfills the two previous conditions may not do so in the future.

* *Reason:* Static memory is initialized in arbitrary order at process startup, and destructed in reverse order of initialization at process shutdown. It is therefore virtually impossible to know in the constructor or destructor if other resources are initialized or destroyed, unless those resources are compile-time constants. `constrexpr` ensures that an expression is evaluated at compile time, and is therefore not affected by static initialization and destruction.

## Classes and structs
Constructors should not call virtual functions of the same class. Constructors should be kept fairly simple.
* *Reason:* The type hierarchy is constructed top-down. If a virtual function is overloaded, it will be called on an object that is not yet constructed. Unfortunately, there is no static check for this. If the logic is complicated, it may be hard to assert that a virtual function is not called indirectly.

## Default functions/constructors/destructor
Use `= default` or `= delete` if there is no custom definition. Especially, be explicit about movability and copyability.
* *Reason:* Types are copyable and moveable by default, but if there is no explicit declaration, there no way of knowing if the designer of that type has thought about what it means for an object of that type to be copied, or moved. Move is often essentially a more efficient version of copy, and often it makes no sense to define it, but it also makes sense to have some types moveable but not copyable, such as `std::unique_ptr`. Being explicit conveys important information.

## Casts of complex types
Minimize casts
* *Reason:* Casts are usually necessary due to design flaws. Casts always introduce a risk of runtime error. `dynamic_cast` is safer, but a better alternative to it is usually [dynamic dispatch](https://en.wikipedia.org/wiki/Dynamic_dispatch).

Use the C++-cast that best expresses your intent.
* *Reason:* C casts are equal to a C++ `reinterpret_cast`. Using them is to bypass all compile-time type checking. Error of reasoning from the programmer may lead to hard-to-detect runtime errors. `const_cast` and `static_cast` have more limited purposes and are therefore safer (but see previous point).

Declare constructors that take one argument (copy and move constructors excepted) `explicit`, unless they are intensional implicit conversion constructors.
* *Reason:* A non-`explicit` constructor that takes one argument will enable implicit type conversions from the argument type to the constructed type.

* *Note:* C++ is full of implicit casts and conversions that are transparent and require a lot of understanding for the programmer even to be aware of them. These are all C casts. This is something that every C++ programmer will just have to live with and be especially aware of.

## Inheritance
Don’t use multiple *implementation* inheritance.
* *Reason:* It is not safe to upcast to a superclass with for example an implicit cast, if that superclass is part of a multiple inheritance. There is also the risk of the [diamond problem](https://en.wikipedia.org/wiki/Multiple_inheritance#The_diamond_problem).

Don’t overuse (single) implementation inheritance.
* *Reason:* Spreading out logic over several classes makes the code harder to read. Object-orientation is not the solution to every problem. Templates are often a more apt technique for algorithm generalization.

Always specify `override` (or `final`) when overriding.
* *Reason:* This is the only way of expressing that a function is an override, which is often very useful information.
* *Note:* Old code does use this keyword simply because it was not available until fairly recently. Adding it to old code is simple and safe.
* *Note:* It is entirely redundant to declare an overriding function `virtual` once it is specified `override`. Some old code does this to be clear.

## Operator overloading
Do not overload `&&`, `||`, `“”`, `,` or unary `&`.
* *Reason:* `&&`, `||` and `,` are not evaluated in the same order as their built-in versions. `""`, declaring your own literals, and overloading `&` will probably make the code confusing.

Operator semantics should follow convention.
* *Reason:* The operators have a special place in the language and strong semantic connotations. Breaking convention will probably confuse and possibly hide errors.

Define non-modifying binary operators as non-member functions.
* *Reason:* If defined as members, the first operand will not follow the same implicit conversion rules as the second operand.

* *See:* Function overloading.

## Function overloading
Overload only when it is not necessary for the reader to know which overload will be called (there are no semantic differences).
* *Reason:* The rules of overloading are tricky, especially in combination with inheritance. They depend on implicit type casts, that are hard to identify, that may also change as the types of the arguments are modified.

## Default arguments
Do not use default arguments on virtual functions.
* *Reason:* If the overrides declare different default arguments, the default used is determined by the static type of the object, subverting the whole idea of polymorphism (the feature is essentially broken).

Only use default arguments when their values are guaranteed to always be the same. Beware of type conversions.
* *Reason:* Default arguments are re-evaluated at each call site, which can be extremely confusing if their value is unpredictable.

When in doubt, use overloads
* *See:*  Function overloading

## Integer types
Use only int of the built-in integer types.
* *Reason:* The built-in integer types are not standardized. If there is need for a certain size, use the definitions from `stdint.h`, such as `int64_t`.

Avoid unsigned unless that is the required behavior (bitfields and modulo overflow). Especially, do not use unsigned to assert that a variable is non-negative.
* *Reason:* Underflow of unsigned integers will easily confuse and cause errors.

Prefer iterators and range-based for loops to indexing and size comparisons.
* *Reason:* The `std` library uses the unsigned type `size_t` for all their containers (many think this is a mistake). Comparing signed and unsigned integers leads to tricky problems. Also, there is a clearer expression of intent and better encapsulation with iterators and range-based for loops.

## `auto` declaration
Good to use for iterators that are long and cumbersome and similar cases.
* *Reason:* Some types are not important enough to justify a long declaration.

OK when the type is either unimportant or clear from local context.
* *Reason:* When the type is obvious without close inspection, `auto` is OK if it helps readability.

May be a good idea for some complicated types.
* *Reason:* Some types, especially templated ones, are hard to figure out, and the programmer risks getting the type wrong. Unintended copying or type conversions may then occur. In those cases, `auto` may be justified, in combination with appropriate naming.

Discouraged in all other cases, such as data members and larger contexts.
* *Reason:* In most cases, types are important information. Hiding them reduces readability.

## Lambda expressions
Avoid default capture, both for by-reference (`[&]`) and by-value (`[=]`) captures. Especially, always use explicit capture when the lamba has a longer lifetime than the current scope. In the case of very short lambdas default capture may be OK.
* *Reason:* Capture by reference, and by value in the case of pointers, makes a shallow copy of each of those captures. When the lambda has a longer lifetime than the captured variable, the capture is in effect a [dangling pointer](https://en.wikipedia.org/wiki/Dangling_pointer). Especially, this can be tricky in the case of `this`, which is automatically captured by both by-reference and by-value default captures, since `this` is implicitly used when members are called.

## Namespaces
Do not use `using` directive to make a whole namespace available, except for project-local namespaces.
* *Reason:* Using `using` directives can easily cause confusion as to what code is actually used.

Use anonymous namespaces for file-private declarations and definitions.
* *Reason:* Anonymous namespaces (as well as `static`) enables internal linkage, meaning they cannot be accessed or defined in another file. This makes the code safer, and also linking time is reduced.

WIP: Use namespaces (how is undecided).
* *Reason:* Namespaces prevent name collisions, both at compile time and at runtime.
