**A trivially copyable type**
is a type whose storage is contiguous (and thus its copy implies a trivial memory block copy, as if performed with memcpy), either cv-qualified or not. This is true for scalar types, trivially copyable classes and arrays of any such types.

**A trivially copyable class**
is a class (defined with class, struct or union) that:

  - uses the implicitly defined copy and move constructors, copy and move assignments, and destructor.
  - has no virtual members.
  - its base class and non-static data members (if any) are themselves also trivially copyable types.
  - This class inherits from integral_constant as being either true_type or false_type, depending on whether T is a trivially copyable type.


**A trivially default constructible type** 
is a type which can be trivially constructed without arguments or initialization values, either cv-qualified or not. This includes scalar types, trivially default constructible classes and arrays of such types.

**A trivially default constructible class**
is a class (defined with class, struct or union) that:

  - uses the implicitly defined default constructor.
  - has no virtual members.
  - has no non-static data members with brace- or equal- initializers.
  - its base class and non-static data members (if any) are themselves also trivially default constructible types.

**A trivial type**
is a type whose storage is contiguous (trivially copyable) and which only supports static default initialization (trivially default constructible), either cv-qualified or not. It includes scalar types, trivial classes and arrays of any such types.

**A trivial class** 
is a class (defined with class, struct or union) that is both trivially default constructible and trivially copyable, which implies that:

  - uses the implicitly defined default, copy and move constructors, copy and move assignments, and destructor.
  - has no virtual members.
  - has no non-static data members with brace- or equal- initializers.
  - its base class and non-static data members (if any) are themselves also trivial types.
