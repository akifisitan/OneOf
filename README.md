# Vaerktojer.OneOf

> "Ah! It's like a compile time checked switch statement!" - Mike Giorgaras

This maintained fork targets .NET 8 and .NET 10. It keeps the `OneOf` namespace and public API while publishing under separate package IDs:

- `Vaerktojer.OneOf` for unions with one to nine alternatives
- `Vaerktojer.OneOf.Extended` for unions with 10 to 32 alternatives
- `Vaerktojer.OneOf.SourceGenerator` for generated `OneOfBase` types

## Getting started

```shell
dotnet add package Vaerktojer.OneOf
```

The library provides discriminated unions for C# through `OneOf<T0, ... Tn>`. Each instance contains one value whose type is one of its generic arguments. `Match` and `Switch` require a handler for every alternative.

## Use cases

### As a method return value

The most frequent use case is as a return value, when you need to return different results from a method. Here's how you might use it in an MVC controller action:

```csharp
public OneOf<User, InvalidName, NameTaken> CreateUser(string username)
{
    if (!IsValid(username)) return new InvalidName();
    var existingUser = _repo.FindByUsername(username);
    if (existingUser != null) return new NameTaken();
    var user = new User(username);
    _repo.Save(user);
    return user;
}

[HttpPost]
public IActionResult Register(string username)
{
    OneOf<User, InvalidName, NameTaken> createUserResult = CreateUser(username);
    return createUserResult.Match(
        user => new RedirectResult("/dashboard"),
        invalidName => {
            ModelState.AddModelError(nameof(username), $"Sorry, that is not a valid username.");
            return View("Register");
        },
        nameTaken => {
            ModelState.AddModelError(nameof(username), "Sorry, that name is already in use.");
            return View("Register");
        }
    );
}
```

#### As an option type

Use `OneOf<Something, None>` as an option type. The `OneOf.Types` namespace includes `Yes`, `No`, `Maybe`, `Unknown`, `True`, `False`, `All`, `Some`, and `None`.

#### Benefits

- The method signature describes every possible result.
- Callers must handle every case.
- Typed error values avoid using exceptions for expected control flow.

### As a method parameter value

You can also use `OneOf` as a parameter type, allowing a caller to pass different types without additional overloads.

```csharp
public void SetBackground(OneOf<string, ColorName, Color> backgroundColor) { ... }

// Accepts a string, a ColorName value, or a Color instance.
```

## Matching

You use the `TOut Match(Func<T0, TOut> f0, ... Func<Tn,TOut> fn)` method to get a value out. Note how the number of handlers matches the number of generic arguments.

### Advantages over `switch`, `if`, or exception-based control flow

Every alternative requires a handler. Adding another generic argument produces compiler errors at call sites that have not handled it.

```csharp
OneOf<string, ColorName, Color> backgroundColor = ...;
Color c = backgroundColor.Match(
    str => CssHelper.GetColorFromString(str),
    name => new Color(name),
    col => col
);
_window.BackgroundColor = c;
```

Use `.Switch` when the handlers do not return a value:

```csharp
OneOf<string, DateTime> dateValue = ...;
dateValue.Switch(
    str => AddEntry(DateTime.Parse(str), foo),
    date => AddEntry(date, foo)
);
```

### TryPick𝑥 method

As an alternative to `.Switch` or `.Match` you can use the `.TryPick𝑥` methods.

```csharp
//TryPick𝑥 methods for OneOf<T0, T1, T2>
public bool TryPickT0(out T0 value, out OneOf<T1, T2> remainder) { ... }
public bool TryPickT1(out T1 value, out OneOf<T0, T2> remainder) { ... }
public bool TryPickT2(out T2 value, out OneOf<T0, T1> remainder) { ... }
```

The return value indicates if the OneOf contains a T𝑥 or not. If so, then `value` will be set to the inner value from the OneOf. If not, then the remainder will be a OneOf of the remaining generic types. You can use them like this:

```csharp
IActionResult Get(string id)
{
    OneOf<Thing, NotFound, Error> thingOrNotFoundOrError = GetThingFromDb(id);

    if (thingOrNotFoundOrError.TryPickT1(out NotFound notFound, out var thingOrError)) //thingOrError is a OneOf<Thing, Error>
      return StatusCode(404);

    if (thingOrError.TryPickT1(out var error, out var thing)) //note that thing is a Thing rather than a OneOf<Thing>
    {
      _logger.LogError(error.Message);
      return StatusCode(500);
    }

    return Ok(thing);
}
```

### Reusable OneOf types using OneOfBase

You can declare a OneOf as a type, either for reuse of the type, or to provide additional members, by inheriting from `OneOfBase`. The derived class will inherit the `.Match`, `.Switch`, and `.TryPick𝑥` methods.

```csharp
public class StringOrNumber : OneOfBase<string, int>
{
    StringOrNumber(OneOf<string, int> _) : base(_) { }

    // optionally, define implicit conversions
    // you could also make the constructor public
    public static implicit operator StringOrNumber(string _) => new StringOrNumber(_);
    public static implicit operator StringOrNumber(int _) => new StringOrNumber(_);

    public (bool isNumber, int number) TryGetNumber() =>
        Match(
            s => (int.TryParse(s, out var n), n),
            i => (true, i)
        );
}

StringOrNumber x = 5;
Console.WriteLine(x.TryGetNumber().number);
// prints 5

x = "5";
Console.WriteLine(x.TryGetNumber().number);
// prints 5

x = "abcd";
Console.WriteLine(x.TryGetNumber().isNumber);
// prints False
```

### OneOfBase source generation

You can generate the `OneOfBase` constructor and conversions for a partial class marked with `GenerateOneOfAttribute`:

```shell
dotnet add package Vaerktojer.OneOf.SourceGenerator
```

and then define a stub like so:

```csharp
[GenerateOneOf]
public partial class StringOrNumber : OneOfBase<string, int> { }
```

During compilation the source generator will produce a class implementing the OneOfBase boiler plate code for you. e.g.

```csharp
public partial class StringOrNumber
{
    public StringOrNumber(global::OneOf.OneOf<string, int> _) : base(_) { }

    public static implicit operator StringOrNumber(string _) => new StringOrNumber(_);
    public static explicit operator string(StringOrNumber _) => _.AsT0;

    public static implicit operator StringOrNumber(int _) => new StringOrNumber(_);
    public static explicit operator int(StringOrNumber _) => _.AsT1;
}
```

## Development

The repository uses the .NET 10 SDK and tests the libraries on .NET 8 and .NET 10.

```shell
dotnet restore OneOf.slnx
dotnet build OneOf.slnx --configuration Release --no-restore
dotnet test OneOf.slnx --configuration Release --no-build --no-restore
```

The runtime union implementations are generated and checked in. Regenerate them after changing `Generator/Program.cs`, then confirm the generated files are unchanged or commit the intended diff:

```shell
dotnet run --project Generator/Generator.csproj --configuration Release -- .
git diff --exit-code -- OneOf/*.generated.cs OneOf.Extended/*.generated.cs
```

Create the three packages locally with:

```shell
dotnet pack OneOf/OneOf.csproj --configuration Release --output artifacts
dotnet pack OneOf.Extended/OneOf.Extended.csproj --configuration Release --output artifacts
dotnet pack OneOf.SourceGenerator/OneOf.SourceGenerator.csproj --configuration Release --output artifacts
```

The packages start at version `0.0.1`. Publishing is manual; CI uploads package files as workflow artifacts.
