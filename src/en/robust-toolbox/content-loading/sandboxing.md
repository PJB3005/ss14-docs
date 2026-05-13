# Sandboxing

*or, how not to ruin your friend's sandcastle.*

Because we need to be able to download code from servers, but don't want that to allow servers to go full malware on people, SS14 employs sandboxing techniques so that content can't, well, go full malware on people.

## Current Implementation

Sandboxing is currently implemented with analysis of the content assemblies before they are loaded. Assemblies are first checked to be verifiable IL to ensure they do not do any *funny pointer stuff*. All referenced members (methods and fields) of the assembly are then checked against a whitelist. All APIs that could possibly be used to break sandbox are, of course, denied. 

The full list of accessible symbols is available in [`Sandbox.yml`](https://github.com/space-wizards/RobustToolbox/blob/master/Robust.Shared/ContentPack/Sandbox.yml). Note that it is entirely possible for a sandbox-safe function to not be accessible due to "nobody asked yet" or "got added to .NET since the list was created." Requests for additions are always welcome!

## "Help, I get a sandbox violation"

It is very easily to accidentally introduce sandbox violations during development. While simple cases like "you tried to use `File.Open()` are pretty easy to find in your code, many violations are the result of less-than-intuitive behavior of C# and .NET. This section will attempt to list any common issues you may run into and what might cause them.

Robust attempts to locate in which C# function the violation ocurred, but this is neither fully reliable or accurate. Furthermore, the location may be weird compiler-generated code.

### Reference to `System.Activator` out of nowhere.

Internally, C# uses `Activator.CreateInstance()` if you instantiate a generic parameter like so:

```csharp
public static T Foo<T>() where T : new()
{
    return new T();
}
```

This *actually* compiles to:

```csharp
public static T Foo<T>() where T : new()
{
    return (T)Activator.CreateInstance(typeof(T));
}
```

As you can see, there is now a reference to `System.Activator`. We can't allow usage of this class directly since it can be very easily used to escape sandbox via e.g. the constructors on `StreamReader`.

To fix this, use `IDynamicTypeFactory.CreateInstance` or `ISandboxHelper.CreateInstance` instead. They verify that the type being constructed is a type defined in content, and is as such allowed.

### Reference to `CollectionsMarshal.SetCount` out of nowhere.

C#'s collection expressions (e.g. `List<int> x = [1, 2, 3, 4, 5]`) are internally lowered to quite complicated code using funny low-level APIs such as `CollectionsMarshal.SetCount`. These are not sandbox safe. The syntax here is extremely versatile, and whether or not it will raise a sandbox violation is up to the whims of the compiler. If you run into this, you can most likely switch to the somewhat more wordy collection initializer syntax, e.g. `List<int> x = new() {1, 2, 3, 4, 5}`.

### Unverifiable instruction

`stackalloc` in C# is unverifiable IL in all cases (yes, even when allocating to a `Span<T>` which is not unsafe) since the `localloc` instruction it uses is always unverifiable. Not a whole lot that can be done here sadly except "don't use `stackalloc`".

### Anything else

Of course the amount of not-whitelisted APIs is too large to count so for all of these I will defer to "ask on Discord if you can't figure out an alternative yourself".

## Disabling sandboxing in development

You can disable sandboxing by setting `ROBUST_DISABLE_SANDBOX=1` as an environment variable. This can be useful to improve client startup times by a second or two. Of course, make sure your CI is still testing sandboxing to make sure you don't push a bad build! 
