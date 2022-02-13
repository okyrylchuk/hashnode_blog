## A comprehensive overview of C# 9 features

### 1. Target-typed New Expression

Previously in C#, a new expression has always required type to be specified (except for implicitly typed array expressions). Starting with C# 9, you can remove specifying type if you explicitly define the type you assign to.

```cs
Person person = new();
```

### 2. Target-typed Conditional ?:

The compiler couldn't figure out shared type between branches in the conditional operator `?:`. C# 9 allows it if there is a target type.

```cs
class Program
{
     class Person { }
     class Student : Person { }
     class Customer : Person { }

     static void Main(string[] args)
     {
         Student student = new ();
         Customer customer = new ();
         bool isStudent = true;

         // Target-typed conditional ?:. Didn't work in previous C# versions.
         Person person = isStudent ? student : customer;
     }
}
```

### 3. Init Only Setters

C# 9 introduces a new `init` accessor to make properties immutable and you can use them with object initializers.

```cs
class Program
{
    class Person
    {
        public string FirstName { get; init; }
        public string LastName { get; init; }
    }
    static void Main(string[] args)
    {
        var person = new Person()
        {
            FirstName = "Oleg",
            LastName = "Kyrylchuk"
        };
        person.FirstName = "New first name"; // immutable
        person.LastName = "New last name";   // immutable
    }
}
```

### 4. Covariant Returns

It was impossible to set a more specific return type in method override in a derived class in C#. C# 9 makes it possible with covariant returns.

```cs
class Program
{
    abstract class Cloneable
    {
        public abstract Cloneable Clone();
    }
    class Person : Cloneable
    {
        public string Name { get; set; }
        public override Person Clone()
        {
            return new Person { Name = Name };
        }
        static void Main(string[] args)
        {
            Person person = new Person { Name = "Oleg" };
            Person clonedPerson = person.Clone();
        }
    }
}
```

### 5. Top-level Programs

C# 9 introduces top-level programs friendly for beginners with no boilerplate code:

- any statement is allowed
- can await things
- magic args parameter
- can return status

```cs
using System;
using System.Threading.Tasks;

Console.WriteLine(args);
await Task.Delay(1000);
return 0;
```

### 6. Type Pattern Matching

Pattern matching has been improved in C# 9.

A type as a pattern is allowed.

```cs
static void MatchPatternByType(object o1, object o2)
{
    var tuple = (o1, o2);
    if (tuple is (int, string))
    {
        Console.WriteLine("Pattern matched in if!");
    }
    // OR
    switch (tuple)
    {
        case (int, string):
            Console.WriteLine("Pattern matched in switch!");
            break;
    }
}
```

Relational patterns permit checking relational constraints when compared to a constant value.

```cs
static string GetCalendarSeason(int month) => month switch
{
    >= 3 and < 6 => "spring",
    >= 6 and < 9 => "summer",
    >= 9 and < 12 => "autumn",
    12 or (>= 1 and < 3) => "winter",
    _ => throw new ArgumentOutOfRangeException(nameof(month),
                    $"Unexpected month: {month}."),
};
```

C# 9 introduces pattern combinators to create logical patterns: `and`, `or`, `not`.

```cs
class Person { 
static void Main()
{
    MatchPatternByType(1, "test")
    // Example with 'not'
    Person person = null;
    if (person is not null) { }

    // Example with 'or' and 'and'
    bool IsLetter(char c) =>
        c is >= 'a' and <= 'z' or >= 'A' and <= 'Z';
}
```

### 7. Static Anonymous Functions

Anonymous functions are not cheap. Lambda can unintentionally capture local variable and it can result in unexpected additional allocations. The `static` modifier on lambdas in C# 9 helps to avoid it.

```cs
[MemoryDiagnoser]
public class Benchmark
{
    [Benchmark]
    public void AnounymousFunction()
    {
        var list = new List<int> { 1, 2, 3 };
        int y = 2;
        list.ForEach(x => x *= y);
    
    [Benchmark]
    public void StaticAnounymousFunction()
    {
        var list = new List<int> { 1, 2, 3 };
        const int y = 2;
        list.ForEach(static x => x *= y);
    }
}
```

Benchmarks.
![benchmark result.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1644768216979/rQNusUejo.png)

### 8. Attributes on Local Functions

Local functions are permitted to have attributes in C# 9. Parameters on local functions are also allowed to have attributes.

```cs
class Program
{
    static void Main()
    {
        [Obsolete("Attribute on local function")]
        void LocalFunction()
        {
            Console.WriteLine("Hello World!");
        }
        LocalFunction();
    }
}
```

### 9. Lambda Discard Parameters

Minor improvements for lambdas and anonymous functions in C# 9 allow discarding their parameters. So intent is clear - parameters are unused. The feature is useful for WinForms coding.

```cs
class Button
{
    public event EventHandler Click;
}
static void Main()
{
    void PrintHelloWorld()
    {
        Console.WriteLine("Hello World!");
    }
    Button button = new();
    button.Click += (_, _) => PrintHelloWorld();
}
```

### 10. Unconstrained Type Parameter Annotations

In C# 8, `?` annotation could only be applied to type parameters that were explicitly constrained to value or reference types. In C# 9 it can be applied to any type parameter, without constraints.

```cs
#nullable enable
class Example
{
    // handle both reference and value types
    public void DoSomething<T>(T? param)
    { }
}

static void Main()
{
    var example = new Example()
    // reference type, not null
    example.DoSomething(new object());
    // reference type, null
    example.DoSomething<object>(null);
    // value type
    example.DoSomething(DateTime.Today);
}
```

### 11. Default Constraint

To allow `?` annotation for type parameters on the overridden methods, C# 8 allowed explicit constraints to value and reference types. C# 9 allows a new `where T: default` constraint on overridden methods that are not constrained to reference or value types.

```cs
class Base
{
    // handle both reference and value types
    public virtual void DoSomething<T>(T? param) { }
}

class Overridden : Base
{
    // override with default constraint
    // handle both reference and value types
    public override void DoSomething<T>(T? param)
        where T : default
    { }
}

static void Main()
{
    Base @base = new();
    @base.DoSomething(1); // value type
    @base.DoSomething(new object()); // ref type, not null
    @base.DoSomething<object>(null); // ref type, nul

    Overridden overriden = new();
    overriden.DoSomething(1); // value type
    overriden.DoSomething(new object()); // ref type, not null
    overriden.DoSomething<object>(null); // ref type, nul
}
```

### 12. Extended Partial Methods

C# 8 had several restrictions on partial methods:

- Must have a `void` return type.
- Cannot have `out` parameters.
- Cannot have any accessibility (implicitly `private`).

C# 9 removes those restrictions.

```cs
class Program
{
    partial class Example
    {
        // Other than void return type is allowed
        public partial int A();
        // Access modifiers are allowed
        public partial void B();
        // Out params are allowed
        public partial void C(out int param);
    }
    partial class Example
    {
        public partial int A() => 0;
        public partial void B() { }
        public partial void C(out int param)
        {
            param = 0;
        }
    }
    static void Main(string[] args)
    {
    }
}
```

### 13. Extension 'GetEnumerator' Support for 'foreach' Loops

In C# `IEnumerator<T>` doesn't have 'GetEnumerator()' method required by `foreach` loop. C# 9 allows to add it as an extension method and `foreach` loop recognizes it.

```cs
public static class Extensions
{
    public static IEnumerator<T> GetEnumerator<T>
        (this IEnumerator<T> enumerator) => enumerator;
}
class Program
{
    static void Main()
    {
        var list = new List<int> { 1, 2, 3 };
        IEnumerator<int> enumerator = list.GetEnumerator();
        foreach (int i in enumerator)
        {
            Console.WriteLine(i);
        }
        // Output:
        // 1
        // 2
        // 3
    }
}
```

### 14. Native-sized Integers

C# 9 introduces new keywords `nint` and `nuint` to define native-sized integers. 32-bit integer when running in 32-bit process. 64-bit integer when running in 64-bit process. They can be used for interop scenarios and low-level libraries. 

```cs
// The compiler provides operations and conversions for
// 'nint' and 'nuint' that are appropriate for integer types. 
nint a = 10;
nint b = 7;

nuint c = 5;
nuint d = 3;
```

### 15. Function Pointers

C# 9 introduces the real function pointers using `delegate*` syntax. Only valid in an `unsafe` context.

```cs
unsafe class Example
{
    // This method has a managed calling convention.
    // This is the same as leaving the 'managed' keyword off.
    delegate*<int, void> functionPointer1
    // The same as functionPointer1, but with explicit 'managed' keyword
    delegate* managed<int, void> functionPointer2
    // This method will be invoked using whatever the default unmanaged calling
    // convention on the runtime platform is. This is platform and architecture
    // dependent and is determined by the CLR at runtime.
    public delegate* unmanaged<int, void> functionPointer3;
}
```

### 16. Records

C# 9 introduces a new data type - `record`. It's a lightweight immutable (if it has `init` props!) version of the class. It's a reference type, but with value-based comparison.

```cs
record Person
{
    // No constructor required to initialize properties
    public string FirstName { get; init; } // immutable
    public string LastName { get; init; } // immutable

static void Main()
{
    var person = new Person
    {
        FirstName = "Oleg",
        LastName = "Kyrylchuk"
    }

    // Use with-expression to create new record from existing
    // specifying the changes in the values of properties
    var bondPerson = person with { LastName = "Bond" }

    // Value base comparison
    var duplicatedPerson = new Person
    {
        FirstName = "Oleg",
        LastName = "Kyrylchuk"
    };

    person.Equals(duplicatedPerson); // true
    _ = person == duplicatedPerson;  // true
}
``` 

### 17. Positional Records

Records in C# 9 allows constructors and deconstructors. However, it requires a lot of code. You can omit most of the code using Positional records.

```cs
// Positional record
// Immutable auto properties are created by position
// Constructor and deconstructor (by position) are here
record Person(string FirstName, string LastName);

static void Main()
{
    // Construct record
    var person = new Person("Oleg", "Kyrylchuk");
    Console.WriteLine(person.FirstName);
    Console.WriteLine(person.LastName)

    // Deconstruct record
    var (firstName, lastName) = person;
    Console.WriteLine(firstName);
    Console.WriteLine(lastName);
}
```

### 18. Records and Inheritance

Records in C# 9 can inherit records (can't classes). Records have a hidden virtual method that clones the whole record. Every derived record overrides it to call the copy constructor and chains it with the copy constructor of the base record.

```cs
record Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }

record Employee : Person
{
    public string Position { get; init; }

static void Main()
{
    Person person = new Employee
    {
        FirstName = "Oleg",
        LastName = "Kyrylchuk",
        Position = ".NET Developer"
    }
    // Hidden virtual method copies the whole record (as Employee)
    Person newPerson = person with { LastName = "Bond" }

    Employee employee = newPerson as Employee;
    Console.WriteLine(employee.FirstName); // Oleg
    Console.WriteLine(employee.LastName);  // Bond
    Console.WriteLine(employee.Position);  // .NET Developer
}
```

### Wrapping Up

All code samples (with a comparison with C# 8) you can find on my [GitHub](https://github.com/okyrylchuk/csharp9_features).















