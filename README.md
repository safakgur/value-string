## ValueString

[![NuGet Status](https://img.shields.io/nuget/v/Dawn.ValueString.svg?style=flat)](https://www.nuget.org/packages/Dawn.ValueString/)

### Introduction

ValueString allows you to serialize an object as a culture-invariant string
and parse it to any type that implements the Parse/TryParse pattern.

It is intended to be used for convenience when there is a need to initialize typed
instances from culture-neutral (invariant) strings - often read from a simple
configuration file or a database table that contains configuration data as strings.

```c#
// ValueString uses the invariant culture when converting
// objects to string if they implement IFormattable.
// So the following value contains "1.5" (dot-separated) even if (1.5).ToString()
// returns "1,5" (comma-separated) due to the current culture.
var value = ValueString.Of(1.5);

// "As" method uses the invariant culture by default.
// It also has an overload accepting an IFormatProvider.
var number = value.As<double>(); // Calls double.Parse.

// Nullable values are supported.
value = ValueString.Of(default(double?));
number = value.As<double>(); // Throws an InvalidCastException.
var nullable = value.As<double?>(); // null.

// Enums are supported.
value = ValueString.Of("Red"); // Enum field name.
var red = value.As<ConsoleColor>();

value = ValueString.Of("12"); // Enum field value.
var green = value.As<ConsoleColor>();

// You can also cast a "dynamic" ValueString to the target type directly.
dynamic d = value;
number = (double)d;

// "Is" is just like "As", but calls the type's TryParse method instead of Parse.
value = ValueString.Of("1.1.1.1");
if (value.Is(out IPAddress address)) // Calls IPAddress.TryParse.
    Console.WriteLine("The IP address is: {0}", address);

// An implicit operator exists converting strings to ValueString instances.
value = "1.5";
```

There are extension methods for parsing key/ValueString dictionaries.  
Consider you inject the configuration parameters to your service like below:

```c#
private readonly Uri uri;
private readonly int? timeout;

public SomeService(IReadOnlyDictionary<string, ValueString> config)
{
    // TryGetValue extension finds the value and converts it to the desired type.
    if (!config.TryGetValue("Uri", out this.uri) || // UriKind.Absolute.
        !config.TryGetValue("Timeout", out this.timeout)) // NumberStyles.Number
        {
            // fast fail/throw exception
        }
}
```

This is equal to the following code without ValueString:


```c#
private readonly Uri uri;
private readonly int? timeout;

public SomeService(IReadOnlyDictionary<string, string> config)
{
    var invariant = CultureInfo.InvariantCulture;

    string temp;
    if (!config.TryGetValue("Uri", out temp) ||
        !Uri.TryCreate(temp, UriKind.Absolute, out this.uri) ||
        !config.TryGetValue("Timeout", out temp) ||
        !int.TryParse(temp, NumberStyles.Number, invariant, out timeout))
        {
            // fast fail/throw exception
        }
}

```

### Internals and Performance

ValueString builds lambda expressions that call the type's original
parsing methods, and cache the compiled delegates for future use.
Therefore they essentially work as fast as the underlying parsing methods themselves.

`As` overloads search for a parsing method in the following order:

1. `static T Parse(string, IFormatProvider)` method
2. `static T Parse(string)` method
3. `static bool TryParse(string, IFormatProvider, out T)` method
4. `static bool TryParse(string, NumberStyles, IFormatProvider, out T)` method
5. `static bool TryParse(string, IFormatProvider, DateTimeStyles, out T)` method
6. `static bool TryParse(string, out T)` method
7. [TypeConverterAttribute][2]
8. `T(string)` constructor

`Is` overloads search for a parsing method in the following order:

1. `static bool TryParse(string, IFormatProvider, out T)` method
2. `static bool TryParse(string, NumberStyles, IFormatProvider, out T)` method
3. `static bool TryParse(string, IFormatProvider, DateTimeStyles, out T)` method
4. `static bool TryParse(string, out T)` method
5. `static T Parse(string, IFormatProvider)` method
6. `static T Parse(string)` method
7. TypeConverterAttribute
8. `T(string)` constructor

Method family    | Behavior when a parsing method can't be found or has failed
---------------- | ----------------------------------------------------------
`ValueString.As` | Throws an `InvalidCastException`
`ValueString.Is` | Returns `false`
                     
`Is` overloads assume TryParse methods to be safe and in cases where they can't
find a TryParse method, they wrap the calls within try-catch blocks to provide
consistent behavior.
                     
See [the unit tests][1] for details.

### Default Parsing Arguments

ValueString, as mentioned above, supports some parsing methods that require
arguments of types `NumberStyles` and `DateTimeStyles`. Since `As` and `Is`
overloads don't let you specify any argument other than a format provider,
the following defaults are used when calling these parsing methods.

* `static bool TryParse(string, NumberStyles, IFormatProvider, out T)`

  ValueString passes `NumberStyles.Number` if this parsing method is used
  since it provides a good set of rules shared by the default parsing methods
  of most numeric types in the BCL.

* `static bool TryParse(string, IFormatProvider, DateTimeStyles, out T)`

   ValueString passes `DateTimeStyles.None` if this parsing method is used
   since it represents the default style used by the `DateTime`'s parsing
   methods that do not require a `styles` parameter.

### Special Cases

There are some special cases implemented for parsing value strings to the
following types.

* `System.Boolean`

  The default boolean parsing methods support converting only from the "True"
  and "False" strings (case-insensitive). ValueString, in addition to these
  two, also allows the following strings to be converted to `bool`:

  "1" and "Yes" are converted to `true`.  
  "0" and "No" are converted to `false`.

* `System.Uri`

  `Uri` class provides a static `TryCreate` method instead of a `TryParse`.
  ValueString allows parsing URI strings to `Uri` instances using this method,
  passing `UriKind.Absolute` as the mandatory `uriKind` argument since it is
  also the default URI kind used by the `Uri(string)` constructor.
  

### FAQ

* #### Why a custom struct instead of some string extensions?

  ValueString uses [CultureInfo.InvariantCulture][3] to convert any
  [IFormattable][4] object to string so it can be reliably persisted.
  Even though it is possible to pass a string that is created using a
  culture-sensitive method directly to `ValueString.Of(string)`, it
  is not recommended since ValueString indicates culture invariance.

  `ToString` methods use the current culture by default and in order for
  `d.ToString().As<double>()` to work, `As` method should also use the current
  culture as its default format provider. If we did that, we would lose the
  ability to persist the serialized data to be consumed by different machines.

  It would be trivial, though, to wrap ValueString
  methods in string extensions if you really want to.

  ```c#
  // Use current culture as default for consistency with ToString.
  public static As<T>(this string s)
      => new Dawn.ValueString(s).As<T>(CultureInfo.CurrentCulture);
  ```

* #### Why are there overloads that accept format providers?

  The primary use of the ValueString is the seamless serialization and
  deserialization using the invariant culture. But there are times when you
  get strings created using custom format providers that are not necessarily
  culture-sensitive. If you have, for any reason, created your own [IFormatProvider][5]
  implementation and want to use your beloved types with ValueString, you can.

* #### And why overloads instead of optional parameters?

  Passing null as a format provider almost universally indicates current culture.
  Every type in the BCL that does culture-sensitive formatting use the current
  culture when the format provider is passed null. But I can't guarantee that
  every implementation will do so. This is why the ValueString methods that accept
  a format provider pass it to the original parsing methods as-is.
  
  This way `v.As<int>()` passes the invariant culture to the `int.Parse` method
  but `v.As<int>(null)` passes null. Combining these overloads by making the
  format provider optional would cause both calls to pass the invariant culture.
  Even though this example makes no difference in the case of `Int32`, it may
  differ for other implementations.

* #### Have I just noticed a `Format` method hiding in there?

  As mentioned in the introduction, ValueString is used mostly for parsing simple
  configuration data, including string templates that, when supplied a model
  (in our case, as key/value pairs), can form a message, URL or some basic HTML:

  ```c#
  var pattern = ValueString.Of("https://github.com/{user}/{repo}");

  // ValueString.Format method works like string.Format, but it uses names
  // instead of indexes. "alignment" and "formatString" components that
  // are recognized by the string.Format are currently not supported.
  var s = pattern.Format(
      ("user", "safakgur"),
      ("repo", "value-string")); // string: "https://github.com/safakgur/value-string"

  // There is also a generic overload that converts the resulted
  // string to the specified type using the invariant culture.
  var uri = pattern.Format<Uri>(
      ("user", "safakgur"),
      ("repo", "value-string")); // Uri: "https://github.com/safakgur/value-string"
  ```

[1]: tests/ValueStringTests.cs
[2]: https://docs.microsoft.com/dotnet/api/system.componentmodel.typeconverterattribute
[3]: https://docs.microsoft.com/dotnet/api/system.globalization.cultureinfo.invariantculture
[4]: https://docs.microsoft.com/dotnet/api/system.iformattable
[5]: https://docs.microsoft.com/dotnet/api/system.iformatprovider
