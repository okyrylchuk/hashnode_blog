## Thirteen C# 10 Features

### Constant Interpolated Strings

C# 10 allows initializing *const* strings using string interpolation, but the placeholder must also be a *const* string.

The placeholder can't be a numeric constant cause it's converted to string at runtime.


```cs
const string name = "Oleg";
const string greeting = $"Hello, {name}.";

Console.WriteLine(greeting);
// Output: Hello, Oleg.
``` 
### Extended Property Patterns
Starting from C# 10, you can reference nested properties or fields within a proper pattern. The property pattern becomes more readable and requires fewer curly brackets.


```cs
Person person = new()
{
    Name = "Oleg",
    Location = new() { Country = "PL" }
};

if (person is { Name: "Oleg", Location.Country: "PL" })
{
    Console.WriteLine("It's me!");
}

class Person
{
    public string Name { get; set; }
    public Location Location { get; set; }
}

class Location
{
    public string Country { get; set; }
}
```
If *Location* is *null*, the pattern will not be matched and returns *false*.

### File Scoped Namespaces

C# 10 introduces a new way of namespace declarations - file scoped namespaces.  
However, you cannot declare a nested namespace or a second file-scoped namespace in the same file. 

```cs
namespace FileScopedNamespace;
    
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello World!");
    }
}
```

### Global Usings
C# 10 adds a new modifier to a *using* directive - *global*. It means that *using* is applied to all files in the compilation. 

All *global using* directives must be before non-global *using* directives. *Global usings* can be combined with a *static* modifier.


```cs
global using System;
global using System.Collections.Generic;
global using System.Linq;
global using System.Threading.Tasks;

List<int> list = new() { 1, 2, 3, 4 };
int sum = list.Sum();
Console.WriteLine(sum);

await Task.Delay(1000);
```

### Assignment and Declaration in The Same Deconstruction

In previous versions of C#, a deconstruction could initialize newly declared variables or assign all values to existing variables.

C# 10 can do both assignment and declaration in the same deconstruction.


```cs
var rgb = (255, 100, 30);

// Initialization & assignment
int r;
(r, int g, int b) = rgb;

Console.WriteLine($"RGB: {r}, {g}, {b}");
// Output: RGB: 255, 100, 30
```

###  Sealed When Overriding ToString for Record Type

You couldn't add a *sealed* modifier when you override *ToString* method in the record type in C# 9.

In C# 10, you can do that. As in classes, it forbids to override *ToString* method in the derived record types.

```cs
Product product = new() { Name = "Bread" };
Console.WriteLine(product.ToString());
// Output: Bread

public record Product
{
    public string Name { get; init; }

    public sealed override string ToString()
    {
        return Name;
    }
}
``` 

### Record Struct and Class Declarations

C# 10 introduces a record struct. You can declare it with *record struct* keywords.
It also adds new syntax for a record class with *record class* keywords. The single *record* keyword from C# 9 means the declaration of record class. It was left for backward compatibility.

```cs
Person me = new() { FirstName = "Oleg", LastName = "Kyrylchuk" };

// 1. Built-in formatting for display
Console.WriteLine(me);
// Output: Person { FirstName = Oleg, LastName = Kyrylchuk }

// 2. Create a new record struct using with-expression
Person otherPerson = me with { FirstName = "John" };
Console.WriteLine(otherPerson);
// Output: Person { FirstName = John, LastName = Kyrylchuk }

// 3. Value equality
Person anotherMe = new() { FirstName = "Oleg", LastName = "Kyrylchuk" };
Console.WriteLine(me == anotherMe);
// Output: True

record struct Person
{
    public string FirstName { get; init; }
    public string LastName { get; init; }
}

// 4. You can create a positional record struct
record struct Product(string Name, decimal Price);
``` 

### Parameterless Struct Constructors

Previous C# versions don't support field initializers in the structs. C# 10 fixes it and closes the gap between *struct* and *class* declarations. If the *struct* has field initializers, the compiler will synthesize a public parameterless constructor.

```cs
using System;

Person person = new() { Name = "Oleg" };

Console.WriteLine(person.Id + " " + person.Name);
// Output: 0cc6caac-d061-4f46-9301-c7cc2a012e47 Oleg

struct Person
{
    public Guid Id { get; init; } = Guid.NewGuid();
    public string Name { get; set; }
}
```

### Lambdas with Attributes

C# 9 has allowed attributes on local functions. C# 10 allows attributes on *lambda* expressions and *lambda* parameters. To distinguish expression attribute from parameter attribute, you must use a parenthesized parameter list for *lambda* expression.

```cs
Action a =[MyAttribute] () => { };               // [MyAttribute] lambda
Action<int> b =[return: MyAttribute] (x) => { };  // [MyAttribute] lambda
Action<int> c = ([MyAttribute] x) => { };         // [MyAttribute] x
var _ = string () => { return string.Empty; };

int one = (x => x)(1);

class MyAttribute : Attribute
{ }
```

### Explicit Return Type in Lambdas

In C# 10, you can specify an explicit return type for *lambdas*. Explicit return types are not supported for anonymous methods declared with *delegate { }* syntax.

```cs
Test<int>();

var l1 = string () => string.Empty;
var l2 = int () => 0;
var l3 = static void () => { };

void Test<T>()
{
    var l4 = T () => default;
}
```

### AsyncMethodBuilder Attribute Applying to the Method

Since C# 7, you can apply the *AsyncMethodBuilder* attribute to a type only. In C# 10, you can also apply the attribute to a single method.

The attribute is read by the compiler and specifies that a type can be an async return type. Applying the attribute to the method you can specify a different async method builder for it. You can read more about it  [here](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/attributes/general#asyncmethodbuilder-attribute) and  [here](https://devblogs.microsoft.com/premier-developer/extending-the-async-methods-in-c/).

```cs
using System.Runtime.CompilerServices;

class Example
{
    [AsyncMethodBuilder(typeof(AsyncVoidMethodBuilder))]
    public void ExampleMethod()
    {
    }
}
```

### Expression With in Structs

C# 9 has introduced *with* expression for record types. It produces a copy of its operand with the specified properties and fields modified. In C# 10 you can also use *with* expression with struct types.

```cs
Product potato = new() { Name = "Potato", Category = "Vegetable" };
Console.WriteLine($"{potato.Name} {potato.Category}");
// Output: Potato Vegetable

Product tomato = potato with { Name = "Tomato" };
Console.WriteLine($"{tomato.Name} {tomato.Category}");
// Output: Tomato Vegetable

struct Product
{
    public string Name { get; set; }
    public string Category { get; set; }
}
```

### Expression With in Anonymous Types

You can use *with* expression also with the anonymous types in C# 10.

```cs
var potato = new { Name = "Potato", Category = "Vegetable" };
Console.WriteLine($"{potato.Name} {potato.Category}");
// Output: Potato Vegetable

var onion = potato with { Name = "Onion" };
Console.WriteLine($"{onion.Name} {onion.Category}");
// Output: Onion Vegetable
```

### Wrapping Up

All code samples you can find on my  [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/C%23%2010%20features).













