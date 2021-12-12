## 20 New APIs in .NET 6

### DateOnly and TimeOnly

.NET 6 introduces two long-awaited types - *DateOnly* and *TimeOnly*. 
They represent the date or time portion of a *DateTime*.


```cs
// public DateOnly(int year, int month, int day)
// public DateOnly(int year, int month, int day, Calendar calendar)
DateOnly dateOnly = new(2021, 9, 25);
Console.WriteLine(dateOnly);
// Output: 25-Sep-21

// public TimeOnly(int hour, int minute)
// public TimeOnly(int hour, int minute, int second)
// public TimeOnly(int hour, int minute, int second, int millisecond)
// public TimeOnly(long ticks)
TimeOnly timeOnly = new(19, 0, 0);
Console.WriteLine(timeOnly);
// Output: 19:00 PM

DateOnly dateOnlyFromDate = DateOnly.FromDateTime(DateTime.Now);
Console.WriteLine(dateOnlyFromDate);
// Output: 23-Sep-21

TimeOnly timeOnlyFromDate = TimeOnly.FromDateTime(DateTime.Now);
Console.WriteLine(timeOnlyFromDate);
// Output: 21:03 PM
``` 
### Parallel.ForEachAsync

It allows you to control the degree of parallelism for scheduled asynchronous work.


```cs
var userHandlers = new[]
{
    "users/okyrylchuk",
    "users/jaredpar",
    "users/davidfowl"
};

using HttpClient client = new()
{
    BaseAddress = new Uri("https://api.github.com"),
};
client.DefaultRequestHeaders.UserAgent.Add(new ProductInfoHeaderValue("DotNet", "6"));

ParallelOptions options = new()
{
    MaxDegreeOfParallelism = 3
};
await Parallel.ForEachAsync(userHandlers, options, async (uri, token) =>
{
    var user = await client.GetFromJsonAsync<GitHubUser>(uri, token);
    Console.WriteLine($"Name: {user.Name}\nBio: {user.Bio}\n");
});

public class GitHubUser
{
    public string Name { get; set; }
    public string Bio { get; set; }
}

// Output:
// Name: David Fowler
// Bio: Partner Software Architect at Microsoft on the ASP.NET team, Creator of SignalR
// 
// Name: Oleg Kyrylchuk
// Bio: Software developer | Dotnet | C# | Azure
// 
// Name: Jared Parsons
// Bio: Developer on the C# compiler
```

### ArgumentNullException.ThrowIfNull()

Nice small improvement for ArgumentNullException.
No need to check for null in every method before throwing an exception.
Now it is a one-liner.


```cs
ExampleMethod(null);

void ExampleMethod(object param)
{
    ArgumentNullException.ThrowIfNull(param);
    // Do something
}
``` 
### PriorityQueue

Meet a new data structure in .NET 6.
PriorityQueue represents a min priority queue.
Each element is enqueued with an associated priority that determines the dequeue order.
The elements with the lowest number get dequeued first.


```cs
PriorityQueue<string, int> priorityQueue = new();

priorityQueue.Enqueue("Second", 2);
priorityQueue.Enqueue("Fourth", 4);
priorityQueue.Enqueue("Third 1", 3);
priorityQueue.Enqueue("Third 2", 3);
priorityQueue.Enqueue("First", 1);

while (priorityQueue.Count > 0)
{
    string item = priorityQueue.Dequeue();
    Console.WriteLine(item);
}

// Output:
// First
// Second
// Third 2
// Third 1
// Fourth
``` 
### Reading and Writing Files

.NET 6 introduces a new low-level API for reading and writing files without a FileStream.
A new type, RandomAccess, provides offset-based APIs for reading and writing files in a thread-safe manner.


```cs
using SafeFileHandle handle = File.OpenHandle("file.txt", access: FileAccess.ReadWrite);

// Write to file
byte[] strBytes = Encoding.UTF8.GetBytes("Hello world");
ReadOnlyMemory<byte> buffer1 = new(strBytes);
await RandomAccess.WriteAsync(handle, buffer1, 0);

// Get file length
long length = RandomAccess.GetLength(handle);

// Read from file
Memory<byte> buffer2 = new(new byte[length]);
await RandomAccess.ReadAsync(handle, buffer2, 0);
string content = Encoding.UTF8.GetString(buffer2.ToArray());
Console.WriteLine(content); // Hello world
``` 
### A New PeriodicTimer

Meet a fully async 'PeriodicTimer'. 
It enables waiting asynchronously for timer ticks.
It has one method, 'WaitForNextTickAsync', which waits for the next tick of the timer or for the timer to be stopped.

```cs
// One constructor: public PeriodicTimer(TimeSpan period)
using PeriodicTimer timer = new(TimeSpan.FromSeconds(1));

while (await timer.WaitForNextTickAsync())
{
    Console.WriteLine(DateTime.UtcNow);
}

// Output:
// 13 - Oct - 21 19:58:05 PM
// 13 - Oct - 21 19:58:06 PM
// 13 - Oct - 21 19:58:07 PM
// 13 - Oct - 21 19:58:08 PM
// 13 - Oct - 21 19:58:09 PM
// 13 - Oct - 21 19:58:10 PM
// 13 - Oct - 21 19:58:11 PM
// 13 - Oct - 21 19:58:12 PM
// ...

``` 
### Metrics API

.NET 6 has an implementation of the OpenTelemetry Metrics API specification.
A 'Meter' class creates instrument objects.
There are instrument classes:
- Counter
- Histogram
- ObservableCounter
- ObservableGauge

You can even listen to the meter.

```cs
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Create Meter
var meter = new Meter("MetricsApp", "v1.0");
// Create counter
Counter<int> counter = meter.CreateCounter<int>("Requests");

app.Use((context, next) =>
{
    // Record the value of measurement
    counter.Add(1);
    return next(context);
});

app.MapGet("/", () => "Hello World");
StartMeterListener();
app.Run();

// Create and start Meter Listener
void StartMeterListener()
{
    var listener = new MeterListener();
    listener.InstrumentPublished = (instrument, meterListener) =>
    {
        if (instrument.Name == "Requests" && instrument.Meter.Name == "MetricsApp")
        {
            // Start listening to a specific measurement recording
            meterListener.EnableMeasurementEvents(instrument, null);
        }
    };

    listener.SetMeasurementEventCallback<int>((instrument, measurement, tags, state) =>
    {
        Console.WriteLine($"Instrument {instrument.Name} has recorded the measurement: {measurement}");
    });

    listener.Start();
}
```

### Reflection API for Nullability Information

It provides nullability information and context from reflection members:
- ParameterInfo
- FieldInfo
- PropertyInfo
- EventInfo


```cs
var example = new Example();
var nullabilityInfoContext = new NullabilityInfoContext();
foreach (var propertyInfo in example.GetType().GetProperties())
{
    var nullabilityInfo = nullabilityInfoContext.Create(propertyInfo);
    Console.WriteLine($"{propertyInfo.Name} property is {nullabilityInfo.WriteState}");
}

// Output:
// Name property is Nullable
// Value property is NotNull

class Example
{
    public string? Name { get; set; }
    public string Value { get; set; }
}
```

### Reflection API for Nested Nullability Information

It allows you to get nested nullability information. 
You can specify that an array property must be non-null, but the elements can be null or vice versa. 
You can get the nullability information for the array elements.


```cs
Type exampleType = typeof(Example);
PropertyInfo notNullableArrayPI = exampleType.GetProperty(nameof(Example.NotNullableArray));
PropertyInfo nullableArrayPI = exampleType.GetProperty(nameof(Example.NullableArray));

NullabilityInfoContext nullabilityInfoContext = new();

NullabilityInfo notNullableArrayNI = nullabilityInfoContext.Create(notNullableArrayPI);
Console.WriteLine(notNullableArrayNI.ReadState);              // NotNull
Console.WriteLine(notNullableArrayNI.ElementType.ReadState);  // Nullable

NullabilityInfo nullableArrayNI = nullabilityInfoContext.Create(nullableArrayPI);
Console.WriteLine(nullableArrayNI.ReadState);                // Nullable
Console.WriteLine(nullableArrayNI.ElementType.ReadState);    // Nullable

class Example
{
    public string?[] NotNullableArray { get; set; }
    public string?[]? NullableArray { get; set; }
}
```

### Process Path and ID
You can access a process path and ID without allocation a new Process instance.


```cs
int processId = Environment.ProcessId
string path = Environment.ProcessPath;

Console.WriteLine(processId);
Console.WriteLine(path);
```

###  A New Configuration Helper

A new configuration helper 'GetRequiredSection` has been added in .NET 6.
It throws an exception if a required configuration section is missing.


```cs
WebApplicationBuilder builder = WebApplication.CreateBuilder(args);
WebApplication app = builder.Build();

MySettings mySettings = new();

// Throws InvalidOperationException if a required section of configuration is missing
app.Configuration.GetRequiredSection("MySettings").Bind(mySettings);
app.Run();

class MySettings
{
    public string? SettingValue { get; set; }
}
```

### CSPNG

You can easily generate a random sequence of values from a Cryptographically Secure Pseudorandom Number Generator (CSPNG).

It's useful for cryptographic applications for:
- key generation
- nonces
- salts in certain signature schemes


```cs
// Fills an array of 300 bytes with a cryptographically strong random sequence of values.
// GetBytes(byte[] data);
// GetBytes(byte[] data, int offset, int count)
// GetBytes(int count)
// GetBytes(Span<byte> data)
byte[] bytes = RandomNumberGenerator.GetBytes(300);
```

### Native Memory API

.NET 6 introduces a new API to allocate native memory. 
A new NativeMemory type has methods for allocation and freeing memory.

```cs
unsafe
{
    byte* buffer = (byte*)NativeMemory.Alloc(100);

    NativeMemory.Free(buffer);

    /* This class contains methods that are mainly used to manage native memory.
    public static class NativeMemory
    {
        public unsafe static void* AlignedAlloc(nuint byteCount, nuint alignment);
        public unsafe static void AlignedFree(void* ptr);
        public unsafe static void* AlignedRealloc(void* ptr, nuint byteCount, nuint alignment);
        public unsafe static void* Alloc(nuint byteCount);
        public unsafe static void* Alloc(nuint elementCount, nuint elementSize);
        public unsafe static void* AllocZeroed(nuint byteCount);
        public unsafe static void* AllocZeroed(nuint elementCount, nuint elementSize);
        public unsafe static void Free(void* ptr);
        public unsafe static void* Realloc(void* ptr, nuint byteCount);
    }*/
}
```

### Power of 2
.NET 6 introduces new helpers for working with powers of 2.
- 'IsPow2' evaluates whether the specified value is a power of two.
- 'RoundUpToPowerOf2' rounds the specified value up to a power of two.


```cs
// IsPow2 evaluates whether the specified Int32 value is a power of two.
Console.WriteLine(BitOperations.IsPow2(128));            // True

// RoundUpToPowerOf2 rounds the specified T:System.UInt32 value up to a power of two.
Console.WriteLine(BitOperations.RoundUpToPowerOf2(200)); // 256
``` 

### WaitAsync on Task

You can easier wait for the task to complete execution asynchronously.
The 'TimeoutException' is thrown when the timeout of operation expires.

⚠️ This is for un-cancellable operations!


```cs
Task operationTask = DoSomethingLongAsync();

await operationTask.WaitAsync(TimeSpan.FromSeconds(5));

async Task DoSomethingLongAsync()
{
    Console.WriteLine("DoSomethingLongAsync started.");
    await Task.Delay(TimeSpan.FromSeconds(10));
    Console.WriteLine("DoSomethingLongAsync ended.");
}

// Output:
// DoSomethingLongAsync started.
// Unhandled exception.System.TimeoutException: The operation has timed out.
``` 

### New Math APIs
New methods:
- SinCos
- ReciprocalEstimate
- ReciprocalSqrtEstimate 

New overloads:
- Min, Max, Abs, Sign, Clamp supports *nint *and *nuint*
- DivRem variants return a tuples

```cs
// New methods SinCos, ReciprocalEstimate and ReciprocalSqrtEstimate
// Simultaneously computes Sin and Cos
(double sin, double cos) = Math.SinCos(1.57);
Console.WriteLine($"Sin = {sin}\nCos = {cos}");

// Computes an approximate of 1 / x
double recEst = Math.ReciprocalEstimate(5);
Console.WriteLine($"Reciprocal estimate = {recEst}");

// Computes an approximate of 1 / Sqrt(x)
double recSqrtEst = Math.ReciprocalSqrtEstimate(5);
Console.WriteLine($"Reciprocal sqrt estimate = {recSqrtEst}");

// New overloads
// Min, Max, Abs, Clamp and Sign supports nint and nuint
(nint a, nint b) = (5, 10);
nint min = Math.Min(a, b);
nint max = Math.Max(a, b);
nint abs = Math.Abs(a);
nint clamp = Math.Clamp(abs, min, max);
nint sign = Math.Sign(a);
Console.WriteLine($"Min = {min}\nMax = {max}\nAbs = {abs}");
Console.WriteLine($"Clamp = {clamp}\nSign = {sign}");

// DivRem variants return a tuple
(int quotient, int remainder) = Math.DivRem(2, 7);
Console.WriteLine($"Quotient = {quotient}\nRemainder = {remainder}");

// Output:
// Sin = 0.9999996829318346
// Cos = 0.0007963267107331026
// Reciprocal estimate = 0.2
// Reciprocal sqrt estimate = 0.4472135954999579
// Min = 5
// Max = 10
// Abs = 5
// Clamp = 5
// Sign = 1
// Quotient = 0
// Remainder = 2
``` 

### CollectionsMarshal.GetValueRefOrNullRef

It returns a ref to the struct value which can be updated in place.
It's not for general purposes, but for high-performance scenarios.

```cs
Dictionary<int, MyStruct> dictionary = new()
{
    { 1, new MyStruct { Count = 100 } }
};

int key = 1;
ref MyStruct value = ref CollectionsMarshal.GetValueRefOrNullRef(dictionary, key);
// Returns Unsafe.NullRef<TValue>() if it doesn't exist; check using Unsafe.IsNullRef(ref value)
if (!Unsafe.IsNullRef(ref value))
{
    Console.WriteLine(value.Count); // Output: 100

    // Mutate in-place
    value.Count++;
    Console.WriteLine(value.Count); // Output: 101
}

struct MyStruct
{
    public int Count { get; set; }
}
``` 

### ConfigureHostOptions

A new ConfigureHostOptions API on IHostBuilder. It makes the application setup simpler.

```cs
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureHostOptions(o =>
            {
                o.ShutdownTimeout = TimeSpan.FromMinutes(10);
            });
}
``` 

### Create Async Scope

.NET 6 introduces a new *CreateAsyncScope* method for creating *AsyncServiceScope*.
The existing *CreateScope* method throws an exception when you dispose of an *IAsyncDisposable* service. The *CreateAsyncScope* provides a straightforward solution.

```cs
await using var provider = new ServiceCollection()
        .AddScoped<Example>()
        .BuildServiceProvider();

await using (var scope = provider.CreateAsyncScope())
{
    var example = scope.ServiceProvider.GetRequiredService<Example>();
}

class Example : IAsyncDisposable
{
    public ValueTask DisposeAsync() => default;
}
``` 

### Simplified Call Patterns for Cryptographic Operations

New methods on *SymmetricAlgorithm* to avoid streams if the payload is already in memory:
- DecryptCbc
- DecryptCfb
- DecryptEcb
- EncryptCbc
- EncryptCfb
- EncryptEcb

They offer a simple approach to using cryptographic APIs.


```cs
static byte[] Decrypt(byte[] key, byte[] iv, byte[] ciphertext)
{
    using (Aes aes = Aes.Create())
    {
        aes.Key = key;
        return aes.DecryptCbc(ciphertext, iv, PaddingMode.PKCS7);
    }
}
``` 

### Conclusion

As you can see, .NET 6 has a lot of new APIs. I don't claim that the list is complete. I could miss something. 

All code samples you can find on my  [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/New%20APIs%20in%20.NET%206).























