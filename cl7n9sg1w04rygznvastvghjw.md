# Twelve C# 11 Features

### Required Members

C# 11 introduces a new `required` modifier to properties and fields to enforce constructors and callers to initialize those values. If you initialize the object with a missing required member, you will get a compilation error.

```cs
// Initializations with required properties - valid
var p1 = new Person { Name = "Oleg", Surname = "Kyrylchuk" };
Person p2 = new("Oleg", "Kyrylchuk");

// Initializations with missing required properties - compilation error
var p3 = new Person { Name = "Oleg" };
Person p4 = new();

public class Person
{
    public Guid Id { get; set; } = Guid.NewGuid();
    public required string Name { get; set; }
    public required string Surname { get; set; }
}
```

If you have several parametrized constructors, you should add the `SetsRequiredMembers` attribute on the constructor, which initializes all required members. It informs the compiler about the proper constructor.

```cs
public class Person
{
    public Person() { }

    [SetsRequiredMembers]
    public Person(string name, string surname)
    {
        Name = name;
        Surname = surname;
    }

    public Guid Id { get; set; } = Guid.NewGuid();
    public required string Name { get; set; }
    public required string Surname { get; set; }
}
```

### Raw String Literals

C# 11 introduces raw string literals. It allows containing of arbitrary text without escaping.

The format is at least three double quotes `""".."""`. If you have text containing three double quotes, you should use four double quotes to escape them. 

Combining with string interpolation, the count of `$` denotes how many consecutive braces start and end the interpolation. In the example below, I want to use interpolation in the JSON, which already contains single braces `{}`. It'll be conflicted with the string interpolation, so I use two `$$` to denote that double braces `{{}}` start and end the interpolation.

```cs
string name = "Oleg", surname = "Kyrylchuk";

string jsonString = 
    $$"""
    {
        "Name": {{name}},
        "Surname": {{surname}}
    }
    """;

Console.WriteLine(jsonString);
```

### UTF-8 String Literals

C# 11 introduces UTF-8 string literals. You can add the `u8` suffix to a string literal to specify UTF-8 encoding. UTF-8 literals are stored as `ReadOnlySpan<byte>`. To get an array of bytes you need to use `ReadOnlySpan<T>.ToArray()` method.

```cs
// C# 10
byte[] array = Encoding.UTF8.GetBytes("Hello World");

// C# 11
ReadOnlySpan<byte> span = "Hello World"u8;
byte[] array = span.ToArray();
```

### List Patterns

C# 11 introduces list patterns.

It extends pattern matching to match sequences of elements in an array or a list. You can use list patterns with any pattern, including constant, type, property, and relational patterns.

```cs
var numbers = new[] { 1, 2, 3, 4 };

// List and constant patterns
Console.WriteLine(numbers is [1, 2, 3, 4]); // True
Console.WriteLine(numbers is [1, 2, 4]);    // False

// List and discard patterns
Console.WriteLine(numbers is [_, 2, _, 4]); // True
Console.WriteLine(numbers is [.., 3, _]);   // True

// List and logical patterns
Console.WriteLine(numbers is [_, >= 2, _, _]); // True
```

### Newlines in String Interpolation Expressions

C# 11 introduces newlines in string interpolation.

It allows any valid C# code between { }, including newlines, to improve readability.

It's helpful when using longer C# expressions in interpolation, like pattern-matching switch expressions or LINQ queries.

```cs
// switch expression in string interpolation
int month = 5;
string season = $"The season is {month switch
{
    1 or 2 or 12 => "winter",
    > 2 and < 6 => "spring",
    > 5 and < 9 => "summer",
    > 8 and < 12 => "autumn",
    _ => "unknown. Wrong month number",
}}.";

Console.WriteLine(season);
// The season is spring.

// LINQ query in string interpolation
int[] numbers = new int[] { 1, 2, 3, 4, 5, 6 };
string message = $"The reversed even values of {nameof(numbers)} are {string.Join(", ", numbers.Where(n => n % 2 == 0)
                             .Reverse())}.";

Console.WriteLine(message);
// The reversed even values of numbers are 6, 4, 2.
```

### Auto-default Structs

The C# 11 compiler automatically initializes any field or property not initialized by a constructor in the structs.

The code below doesn't compile in the previous versions of C#. The compiler sets the default values.

```cs
struct Person
{
    public Person(string name)
    {
        Name = name;
    }

    public string Name { get; set; }
    public int Age { get; set; }
}
```

### Pattern Match `Span<char>` on a Constant String

Using pattern matching, you can test if the string has a specific constant value in C#.

C# 11 allows pattern matching a `Span<char>` and `ReadOnlySpan<char>` on a constant string.

```cs
ReadOnlySpan<char> str = "Oleg".AsSpan();

if (str is "Oleg")
{
    Console.WriteLine("Hey, Oleg");
}
```

### Generic Attributes

In C#, if you want to pass the type to an attribute, you can use the `typeof` expression.

However, there is no way to constrain what types are allowed to be passed. C# 11 allows generic attributes.

```cs
class MyType { }

class GenericAttribute<T> : Attribute
    where T: MyType 
{
    private T _type;
}

[Generic<MyType>]
class MyClass { }
```

### Extended `nameof` Scope

C# 11 extends the scope of `nameof` expressions.

You can specify the name of a method parameter in an attribute on the method or parameter declaration.

This feature can be used in adding attributes for code analysis.

```cs
public class MyAttr : Attribute
{
    private readonly string _paramName;
    public MyAttr(string paramName)
    {
        _paramName = paramName;
    }
}
public class MyClass
{
    [MyAttr(nameof(param))]
    public void Method(int param, [MyAttr(nameof(param))] int anotherParam)
    { }
}
```

### An Unsigned Right-shift Operator

C# 11 introduces an unsigned right-shift operator `>>>`.

It shifts bits right without replicating the high-order bit on each shift.

```cs
int n = -32;
Console.WriteLine($"Before shift: bin = {Convert.ToString(n, 2),32}, dec = {n}");

int a = n >> 2;
Console.WriteLine($"After     >>: bin = {Convert.ToString(a, 2),32}, dec = {a}");

int b = n >>> 2;
Console.WriteLine($"After    >>>: bin = {Convert.ToString(b, 2),32}, dec = {b}");

// Output:
// Before shift: bin = 11111111111111111111111111100000, dec = -32
// After     >>: bin = 11111111111111111111111111111000, dec = -8
// After    >>>: bin =   111111111111111111111111111000, dec = 1073741816
```

### Static Abstract Members in Interfaces

C# 11 introduces static abstract members in interfaces.

You can add static abstract members in interfaces to define interfaces that include overloadable operators, other static members, and static properties.

```cs
public interface IAdditionOperator<TSelf, TOther, TResult>
    where TSelf : IAdditionOperator<TSelf, TOther, TResult>
{
    static abstract TResult operator +(TSelf left, TOther right);
}
```

### Generic Math

Static abstract members feature has been added to enable generic math support. More about it you can read in this [blog post](https://devblogs.microsoft.com/dotnet/preview-features-in-net-6-generic-math/).

```cs
Point p1 = new() { X = 10, Y = 5 };
Point p2 = new() { X = 5, Y = 7 };

Point p3 = p1 + p2;
Point p4 = p1 - p2;
Console.WriteLine(p3);
Console.WriteLine(p4);

public record Point : 
    IAdditionOperators<Point, Point, Point>, 
    ISubtractionOperators<Point, Point, Point>
{
    public int X { get; set; }
    public int Y { get; set; }

    public static Point operator +(Point left, Point right)
    {
        return left with { X = left.X + right.X, Y = left.Y + right.Y };
    }

    public static Point operator -(Point left, Point right) 
    {
        return left with { X = left.X - right.X, Y = left.Y - right.Y };
    }

    public override string ToString() => $"X: {X}; Y: {Y}";
}
```

### File scoped types

*I know my post's title is "Twelve C# 11 Features". However, I want to present you the thirteenth feature. I've added it to my post after .NET 7 release.*

C# 11 introduces a new access modifier `file`.

The visibility of created type is scoped to the source file in which it is declared.

This feature helps source generator authors avoid naming collisions.

```cs
file class Person
{
    public string FirstName { get; set; }
    public string LastName { get; set; }
}
```

### Wrapping Up

All code samples you can find on my [GitHub](https://github.com/okyrylchuk/dotnet7_features/tree/main/C%23%2011%20features).







