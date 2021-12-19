## LINQ improvements in .NET 6

### The Default Value for *OrDefault Methods

The *Enumerable.FirstOrDefault* method returns the first element of a sequence, or a default value if no element is found. In .NET 6, you can override the default value. You can override the default value also for *SingleOrDefault *and *LastOrDefault *methods.

```cs
List<int> list1 = new() { 1, 2, 3 };
int item1 = list1.FirstOrDefault(i => i == 4, -1);
Console.WriteLine(item1); // -1

List<string> list2 = new() { "Item1" };
string item2 = list2.SingleOrDefault(i => i == "Item2", "Not found");
Console.WriteLine(item2); // Not found
``` 

### New *By Methods

.NET 6 introduces the new *Enumerable.*By* methods. A 'keySelector' is provided to compare elements by. 

New methods:
- MinBy
- MaxBy
- DistinctBy
- ExceptBy
- IntersectBy
- UnionBy


```cs
List<Product> products = new()
{
    new() { Name = "Product1", Price = 100 },
    new() { Name = "Product2", Price = 5 },
    new() { Name = "Product3", Price = 50 },
};

Product theCheapestProduct = products.MinBy(x => x.Price);
Product theMostExpensiveProduct = products.MaxBy(x => x.Price);
Console.WriteLine(theCheapestProduct);
// Output: Product { Name = Product2, Price = 5 }
Console.WriteLine(theMostExpensiveProduct);
// Output: Product { Name = Product1, Price = 100 }

record Product
{
    public string Name { get; set; }
    public decimal Price { get; set; }
}
``` 

### A new Chunk Method

If you need to split elements of a sequence into chunks, you don't have to implement it on your own anymore in .NET 6. It introduces a new *Enumerable.Chunk* extension method.

```cs
IEnumerable<int> numbers = Enumerable.Range(1, 505);
IEnumerable<int[]> chunks = numbers.Chunk(100);

foreach (int[] chunk in chunks)
{
    Console.WriteLine($"{chunk.First()}...{chunk.Last()}");
}

//  Output:
//  1...100
//  101...200
//  201...300
//  301...400
//  401...500
//  501...505
``` 

### Three-way Zip Method
The *Enumerable.Zip* extension method produces a sequence of tuples with elements from two specified sequences. In .NET 6, it can combine tuples from three sequences.

It cannot combine tuples from four and more sequences.


```cs
int[] numbers = { 1, 2, 3, 4, };
string[] months = { "Jan", "Feb", "Mar" };
string[] seasons = { "Winter", "Winter", "Spring" };

var test = numbers.Zip(months).Zip(seasons);

foreach ((int, string, string) zipped in numbers.Zip(months, seasons))
{
    Console.WriteLine($"{zipped.Item1} {zipped.Item2} {zipped.Item3}");
}
// Output:
// 1 Jan Winter
// 2 Feb Winter
// 3 Mar Spring
``` 

### Index Support in the ElementAt Method

.NET Core 3.0 has introduced the *Index* struct, which is used by the C# compiler to support a new unary prefix "hat" operator (^). It means index "from the end" of collection. In .NET 6, *Enumerable.ElementAt* method supports the *Index*.

```cs
IEnumerable<int> numbers = new int[] { 1, 2, 3, 4, 5 };
int last = numbers.ElementAt(^0);
Console.WriteLine(last); // 5
``` 

### Range Support in the Take Method
The *Range* struct has been introduced in the .NET Core 3.0. It is used by the C# compiler to support a range operator '..'

In .NET 6, the *Enumerable.Take* method supports the *Range*.

```cs
IEnumerable<int> numbers = new int[] { 1, 2, 3, 4, 5 };

IEnumerable<int> taken1 = numbers.Take(2..4);
foreach (int i in taken1)
    Console.WriteLine(i);
// Output:
// 3
// 4

IEnumerable<int> taken2 = numbers.Take(..3);
foreach (int i in taken2)
    Console.WriteLine(i);
// Output:
// 1
// 2
// 3

IEnumerable<int> taken3 = numbers.Take(3..);
foreach (int i in taken3)
    Console.WriteLine(i);
// Output:
// 4
// 5
``` 

### Avoiding Enumeration with TryGetNonEnumeratedCount

.NET 6 introduces a new *Enumerable.TryGetNonEnumerated* method.
It attempts to determine the number of elements in a sequence without forcing an enumeration. It's useful for *IQueryable*, when calling *Enumerable.Count* you don't want to evaluate the entire query.

```cs
IEnumerable<int> numbers = GetNumbers();
TryGetNonEnumeratedCount(numbers);
// Output: Could not get a count of numbers without enumerating the sequence 

IEnumerable<int> enumeratedNumbers = numbers.ToList();

var test = enumeratedNumbers.ElementAt(-1);
    
TryGetNonEnumeratedCount(enumeratedNumbers);
// Output: Count: 5

void TryGetNonEnumeratedCount(IEnumerable<int> numbers)
{
    if (numbers.TryGetNonEnumeratedCount(out int count))
        Console.WriteLine($"Count: {count}");
    else
        Console.WriteLine("Could not get a count of numbers without enumerating the sequence");
}

IEnumerable<int> GetNumbers()
{
    yield return 1;
    yield return 2;
    yield return 3;
    yield return 4;
    yield return 5;
}
```

### Wrapping Up

All code samples you can find on my  [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/LINQ%20imrpovements).




