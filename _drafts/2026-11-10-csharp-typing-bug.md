---
title: "I Found an Ambiguity in the C# Language"
date: 2024-12-18 17:00:00 +1100

categories: [Misc]
tags: [misc]
image:
    path: "https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTnWiR7FeKFI4NLvecB-IPHQTpoRB-ebL15ypv8siBoEw&s=10"
---

I found an ambiguity in the C# language. It causes a compile-time failure.

Remember that [instances of covariant interfaces indexed by a type $T$ can be assigned to instances of that interface indexed by a subtype of $T$](https://mukunds.blog/posts/variance-for-people-in-a-hurry/) (and the reverse for instances of contravariant interfaces).

Consider the following variant interfaces and their concrete, empty implementations:

```csharp
public interface ICo<out T> {}
public class Co<T> : ICo<T> {}

public interface IContra<in T> {}
public class Contra<T> : IContra<T> {}
```

Now consider the following method:

```csharp
static ICo<T> FunnyMethod<T>(ICo<T> co, IContra<T> contra) {
    return co;
}
```

Some observations about the above method. The first parameter can be substituted for `ICo<Supertype of T>` while the second can be substituted for `IContra<Subtype of T>`. So if we don't annotate the specific type by calling `FunnyMethod<T>`, there is ambiguity in what the type `T` actually is. Depending on how the language decides the type of each parameter.

```csharp
using System;

public class Animal {}
public class Dog : Animal {}

public class Program {
    public static void Main(string[] args) {
        ICo<Dog> dogCo = new Co<Dog>();
        IContra<Animal> animalContra = new Contra<Animal>();

        ICo<Animal> x = dogCo;
        IContra<Dog> y = animalContra;

        var test = FunnyMethod(dogCo, animalContra);
        Console.WriteLine("Done!");
    }
}
```

In the above example, at compile time `dogCo` can be interpreted as `ICo<Dog>`, `ICo<Animal>`, or `ICo<object>`. `animalContra` can be interpreted as `IContra<Animal>`, `IContra<Dog>`, or `IContra<{subtype of dog}>`.

It is possible for the language to interpret the types respectively as `ICo<Dog>` and `IContra<Dog>`- in which the type of the method would be `Dog` and return type would be `ICo<Dog>`- or `ICo<Animal>` and `IContra<Animal>`- in which case the type of the method would be `Animal` and return type would be `ICo<Animal>`.

If there were one parameter, the language would simple make the type as specific as it could be. For `ICo<T>` this means pushing the type down as much as possible and for `IContra<T>` it means pushing the type up.

But in this case, there are equal and opposite forces. It ends up taking `Animal` arbitrarily, the higher type. If we replace `dogCo` with `poodleCo` and `animalContra` with `objectContra` then it resolves to `objectContra`. Which, fair enough, is a valid type. But the catch is that if we specify the type of `test` is `ICo<Dog>`, which is valid, it fails at compile time with error `error CS0266: Cannot implicitly convert type 'ICo<Animal>' to 'ICo<Dog>'. An explicit conversion exists (are you missing a cast?)`. The language is not able to use `test`, that we are inferring *from*, to evaluate `FunnyMethod`, that we are inferring *to*.

To be clear, this is not an accident or a bug. It is intended behaviour- just one that is counterintuitive and confusing to most developers. Eric Lippert, Designer of C#, discusses another similar bug [at this stackoverflow answer](https://stackoverflow.com/questions/50390761/generic-interface-type-inference-weirdness-in-c-sharp).

```csharp
// Source - https://stackoverflow.com/a/50396188
// Posted by Eric Lippert, modified by community. See post 'Timeline' for change history
// Retrieved 2026-11-11, License - CC BY-SA 4.0

interface I<T> {}
class X<T> {}

class Y {}
class Z {}  
class C : I<X<Y>>, I<Z> {}

public class P
{   
    static void M<T>(I<X<T>> i) { }
    public static void Main()
    {
        M(new C());
    }
}
```

Fails with error `error CS0411: The type arguments for method 'Program.M<T>(I<X<T>>)' cannot be inferred from the usage. Try specifying the type arguments explicitly.`

He explains; "You ask why, and why questions are always difficult to answer, so let me rephrase the question:

What line of the specification disallows this inference?"

He also goes on to say:

"What factors were considered when the design team made this decision?

You are right that in theory we could detect in your case that `X<Y>` is intended and `Z` is not. If you would care to propose a type inference algorithm that can handle non-unique situations like this that never makes a mistake -- remember, `Z` could be a subtype or a supertype of `X<Y>` or `X<Something Else>` and `I` could be covariant -- then I am sure that the C# team would be happy to consider your proposal."

He ends with a much broader takeaway:

"A question which you did not ask but comes to mind:

Could the error message be better?

...

the fact that I didn't immediately diagnose the problem and had to go back to the spec is indicative that the message is unclear."
