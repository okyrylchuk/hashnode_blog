## Seven System.Text.Json features in the .NET 6

### Ignore Circular References

In .NET 5, you can preserve references for circular references using System.Text.Json.
But you couldn't ignore them. The *JsonException * is thrown if circular references have been detected. In .NET 6, you can ignore them.

```cs
Category dotnet = new()
{
    Name = ".NET 6",
};
Category systemTextJson = new()
{
    Name = "System.Text.Json",
    Parent = dotnet
};
dotnet.Children.Add(systemTextJson);

JsonSerializerOptions options = new()
{
    ReferenceHandler = ReferenceHandler.IgnoreCycles,
    WriteIndented = true
};

string dotnetJson = JsonSerializer.Serialize(dotnet, options);
Console.WriteLine($"{dotnetJson}");

public class Category
{
    public string Name { get; set; }
    public Category Parent { get; set; }
    public List<Category> Children { get; set; } = new();
}

// Output:
// {
//   "Name": ".NET 6",
//   "Parent": null,
//   "Children": [
//     {
//       "Name": "System.Text.Json",
//       "Parent": null,
//       "Children": []
//     }
//   ]
// }
```

### Notifications for (De)Serialization

In .NET 6, System.Text.Json exposes notifications for (de)serialization.

There are four new interfaces to implement according to your needs:
* IJsonOnDeserialized
* IJsonOnDeserializing
* IJsonOnSerialized
* IJsonOnSerializing

```cs
Product invalidProduct = new() { Name = "Name", Test = "Test" };
JsonSerializer.Serialize(invalidProduct);
// The InvalidOperationException is thrown

string invalidJson = "{}";
JsonSerializer.Deserialize<Product>(invalidJson);
// The InvalidOperationException is thrown

class Product : IJsonOnDeserialized, IJsonOnSerializing, IJsonOnSerialized
{
    public string Name { get; set; }

    public string Test { get; set; }

    public void OnSerialized()
    {
        throw new NotImplementedException();
    }

    void IJsonOnDeserialized.OnDeserialized() => Validate(); // Call after deserialization
    void IJsonOnSerializing.OnSerializing() => Validate();   // Call before serialization

    private void Validate()
    {
        if (Name is null)
        {
            throw new InvalidOperationException("The 'Name' property cannot be 'null'.");
        }
    }
}
```

###  Serialization Order of Properties

In .NET 6, the *JsonPropertyOrderAttribute* has been added to System.Text.Json. It allows controlling the serialization order of properties. Previously, the serialization order was determined by reflection order.

```cs
Product product = new()
{
    Id = 1,
    Name = "Surface Pro 7",
    Price = 550,
    Category = "Laptops"
};

JsonSerializerOptions options = new() { WriteIndented = true };
string json = JsonSerializer.Serialize(product, options);
Console.WriteLine(json);

class Product : A
{
    [JsonPropertyOrder(2)] // Serialize after Price
    public string Category { get; set; }

    [JsonPropertyOrder(1)] // Serialize after other properties that have default ordering
    public decimal Price { get; set; }

    public string Name { get; set; } // Has default ordering value of 0

    [JsonPropertyOrder(-1)] // Serialize before other properties that have default ordering
    public int Id { get; set; }
}

class A
{
    public int Test { get; set; }
}

// Output:
// {
//   "Id": 1,
//   "Name": "Surface Pro 7",
//   "Price": 550,
//   "Category": "Laptops"
// }
```

### Write Raw JSON with Utf8JsonWriter

.NET 6 introduces the possibility to write raw JSON with System.Text.Json.Utf8JsonWriter.

Helpful when you want:
* to enclose existing JSON in new JSON
* to format values differently from the default formatting

```cs
JsonWriterOptions writerOptions = new() { Indented = true, };

using MemoryStream stream = new();
using Utf8JsonWriter writer = new(stream, writerOptions);

writer.WriteStartObject();
writer.WriteStartArray("customJsonFormatting");
foreach (double result in new double[] { 10.2, 10 })
{
    writer.WriteStartObject();
    writer.WritePropertyName("value");
    writer.WriteRawValue(FormatNumberValue(result), skipInputValidation: true);
    writer.WriteEndObject();
}
writer.WriteEndArray();
writer.WriteEndObject();
writer.Flush();

string json = Encoding.UTF8.GetString(stream.ToArray());
Console.WriteLine(json);

static string FormatNumberValue(double numberValue)
{
    return numberValue == Convert.ToInt32(numberValue)
        ? numberValue.ToString() + ".0"
        : numberValue.ToString();
}

// Output:
// {
//    "customJsonFormatting": [
//      {
//        "value": 10.2
//      },
//      {
//        "value": 10.0
//      }
//  ]
// }
```

###  IAsyncEnumerable Support

In .NET 6, System.Text.Json supports *IAsyncEnumerable*. The serialization of *IAsyncEnumerable* transforms it into an array. For deserialization of root level JSON Arrays, the *DeserializeAsyncEnumerable* method has been added.

```cs
static async IAsyncEnumerable<int> GetNumbersAsync(int n)
{
    for (int i = 0; i < n; i++)
    {
        await Task.Delay(1000);
        yield return i;
    }
}
// Serialization using IAsyncEnumerable
JsonSerializerOptions options = new() { WriteIndented = true };
using Stream outputStream = Console.OpenStandardOutput();
var data = new { Data = GetNumbersAsync(5) };
await JsonSerializer.SerializeAsync(outputStream, data, options);
// Output:
// {
//    "Data": [
//      0,
//      1,
//      2,
//      3,
//      4
//  ]
// }

// Deserialization using IAsyncEnumerable
using MemoryStream memoryStream = new(Encoding.UTF8.GetBytes("[0,1,2,3,4]"));
// Wraps the UTF-8 encoded text into an IAsyncEnumerable<T> that can be used to deserialize root-level JSON arrays in a streaming manner.
await foreach (int item in JsonSerializer.DeserializeAsyncEnumerable<int>(memoryStream))
{
    Console.WriteLine(item);
}
// Output:
// 0
// 1
// 2
// 3
// 4
```
 A GIF of how IAsyncEnumerable is being serialized.

![Serialize IEnumerableAsync.gif](https://cdn.hashnode.com/res/hashnode/image/upload/v1641126935090/B65KrqyPv.gif)

### (De)Serialize JSON Data To/From a Stream

In .NET 6, overloads for streams for synchronous methods Serialize/Deserialize have been added.

```cs
string json = "{\"Value\":\"Deserialized from stream\"}";
byte[] bytes = Encoding.UTF8.GetBytes(json);

// Deserialize from stream
using MemoryStream ms = new MemoryStream(bytes);
Example desializedExample = JsonSerializer.Deserialize<Example>(ms);
Console.WriteLine(desializedExample.Value);
// Output: Deserialized from stream

// ==================================================================

// Serialize to stream
JsonSerializerOptions options = new() { WriteIndented = true };
using Stream outputStream = Console.OpenStandardOutput();
Example exampleToSerialize = new() { Value = "Serialized from stream" };
JsonSerializer.Serialize<Example>(outputStream, exampleToSerialize, options);
// Output:
// {
//    "Value": "Serialized from stream"
// }

class Example
{
    public string Value { get; set; }
}
```

###  Work With JSON Like a DOM

.NET 6 provides types for handling an in-memory writeable document object model (DOM) for random access of the JSON elements within a structured view of data.

New types:
* JsonArray
* JsonNode
* JsonObject
* JsonValue

```cs
// Parse a JSON object
JsonNode jNode = JsonNode.Parse("{\"Value\":\"Text\",\"Array\":[1,5,13,17,2]}");
string value = (string)jNode["Value"];
Console.WriteLine(value); // Text
                          // or
value = jNode["Value"].GetValue<string>();
Console.WriteLine(value); // Text

int arrayItem = jNode["Array"][1].GetValue<int>();
Console.WriteLine(arrayItem); // 5
                              // or
arrayItem = jNode["Array"][1].GetValue<int>();
Console.WriteLine(arrayItem); // 5

// Create a new JsonObject
var jObject = new JsonObject
{
    ["Value"] = "Text",
    ["Array"] = new JsonArray(1, 5, 13, 17, 2)
};
Console.WriteLine(jObject["Value"].GetValue<string>());  // Text
Console.WriteLine(jObject["Array"][1].GetValue<int>());  // 5

// Converts the current instance to string in JSON format
string json = jObject.ToJsonString();
Console.WriteLine(json); // {"Value":"Text","Array":[1,5,13,17,2]}
``` 

### Wrapping Up

All code samples you can find on my  [GitHub](https://github.com/okyrylchuk/dotnet6_features/tree/main/System.Text.Json%20features).








