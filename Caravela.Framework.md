# Caravela.Framework

> You can try PostSharp "Caravela" in your browser, without installing anything, at https://try.postsharp.net/.

## Table of contents

- [Table of contents](#table-of-contents)
- [Introduction](#introduction)
- [Limitations](#limitations)
- [Example](#example)
- [Inference rules](#inference-rules)
- [Aspects, advices and Initialize](#aspects-advices-and-initialize)
- [Template context](#template-context)
- [Packaging an aspect](#packaging-an-aspect)

## Introduction

Caravela.Framework is an [AOP](https://en.wikipedia.org/wiki/Aspect-oriented_programming) framework based on templates written in pure C#.

These templates make it easy to write code that combines compile-time information (such as names and types of parameters of a method) and run-time information (such as parameter values) in a natural way, without having to learn another language or having to combine C# with some special templating language.

## Limitations

Caravela.Framework is in a very early preview, which means it currently has severe limitations:

* `OverrideMethod` is the only available aspect/advice;
* many constructs of C# (including very common ones) are not supported in templates;
* only a single advice can be applied to each method.


## Example

For example, consider this simple aspect, which logs the name of a method and information about its parameters to the console and then lets it execute as usual:

```c#
class Log : OverrideMethodAspect
{
    public override dynamic Template()
    {
        Console.WriteLine(target.Method.Name);
        foreach (var parameter in target.Parameters)
        {
            Console.WriteLine(parameter.Type + " " + parameter.Name + " = " + parameter.Value);
        }

        return proceed();
    }
}
```

This aspect can be applied to a method as an attribute:

```c#
[Log]
void CountDown(string format, int n)
{
    for (int i = 0; i < n; i++)
    {
        Console.WriteLine(format, i);
    }
}
```

This changes the method so that it behaves as if it was written like this:

```c#
void CountDown(string format, int n)
{
    Console.WriteLine("CountDown");
    Console.WriteLine("string format = " + format);
    Console.WriteLine("int n = " + n);
    for (int i = 0; i < n; i++)
    {
        Console.WriteLine(format, i);
    }
}
```

Notice that the compile-time `foreach` loop was unrolled, so that each parameter has its own statement and that the compile-time expressions `parameter.Type` and `parameter.Name` have been evaluated and even folded with the nearby constants. On the other hand, the run-time calls to `Console.WriteLine` have been preserved. The expression `parameter.Value` is special, and has been translated to accessing the values of the parameters.

## Inference rules

The template engine assigns any expression and statement in your template code to one of these two _scopes_: compile time, or run time.
It uses both inference and coercion rules. When conflicts happen between rules, you will get a compile-time error.

The inference and coercion rules are the following:

* If _x_ is run-time, then the result of any expression containing _x_ is also run-time (inference to run-time).
  
    Example: `DateTime.Now` is run-time, therefore `DateTime.Now.Day` and `DateTime.Now.Day + 5` are run-time too.
    
* If _F_ is a compile-time function, its parameters must be compile-time (coercion to compile-time).

    Example: in `target.Method.Parameters[i]`, `i` must be compile-time.

* The special method _compileTime(x)_ coerces _x_ to be compile-time.

    Example: `compileTime( DateTime.Now )` returns the compilation time.

* When a compile-time member returns a _dynamic_ value, for instance _IParameter.Value_, this value is run-time even if the 
  member itself is compile-time.

    Example: `parameter.Value` is run-time and `compileTime( parameter.Value )` is invalid.

* Some expressions have undetermined scope (for instance literals or instance of types that exist both at compile-time and run-time). In case
  of ambiguity, run-time scope is assumed. If the default scope is not adequate, you should use the _compileTime_ method.

    Example: `new StringBuilder()` is run-time but `compileTime( new StringBuilder() )` is compile-time.

* Local variables can be either compile-time or run-time. The scope is uniquely determined by the expression of the variable initializer.
  The previous rules are applied. Local variables without an initializer are run-time.

  Examples:
    * In `int i;`, `i` is run-time.
    * In `int i = 0`, `i` is run-time.
    * In `int i = compileTime(0)`, `i` is compile-time.
    * In `var p = target.Method.Parameters[i]`, `p` is compile-time.


* `if ... else` conditions are compile-time if and only if the condition is compile-time.

    Examples:

    * `if ( target.Method.ReturnType.Is(typeof(void)) ) {} else {}` is compile-time.
    * `if ( target.Method.Parameters[i].Value != null) {} else {}` is run-time.

* `foreach` loops are compile-time if and only if the expression is compile-time.

    Examples:

     * `foreach ( var p in target.Method.Parameters ) {}` is compile-time (therefore `p` is compile-time).
     * `foreach ( var i in new [] { 1, 2, 3 } ) { }` is run-time.

* `for` loops are compile-time if and only if _all_ members (initializer, condition, incrementation) are compile-time.
  
    TODO: `for` loops are now run-time only. Compile-time detection has not been implemented.

* `proceed()` can be invoked only once in a template. It cannot be invoked from a compile-time loop.

* Compile-time variables cannot be assigned from run-time loops.

## Aspects, advices and Initialize

While abstract aspects like `OverrideMethodAspect` work well for simple needs, more customization is required in more complex cases. For example, consider the situation where you want to apply an aspect attribute to a type and have it affect all its methods. In Caravela, you can do this by directly implementing the `IAspect<T>` interface and putting this logic into the `Initialize` method. For example:

```c#
public class CountMethodsAspect : Attribute, IAspect<INamedType>
{
    public void Initialize(IAspectBuilder<INamedType> aspectBuilder)
    {
        var methods = aspectBuilder.TargetDeclaration.Methods.GetValue();

        this.methodCount = methods.Count();

        foreach (var method in methods)
        {
            aspectBuilder.AdviceFactory.OverrideMethod(method, nameof(Template));
        }
    }

    int i;
    int methodCount;

    [OverrideMethodTemplate]
    public dynamic Template()
    {
        Console.WriteLine($"This is {++this.i} of {this.methodCount} methods.");
        return proceed();
    }
}
```

This aspect adds the `OverrideMethod` *advice* to each method in a marked type. Here, "advice" is some kind of modification applied to a single element in your code.

As you can see, the `Initialize` method can also be used for other purposes, like initializing values shared by the advices of the aspect.

## Template context

Inside a template method, extra operations are available through members of the `TemplateContext` class. These members are intended to be used directly, which requires adding `using static Caravela.Framework.Aspects.TemplateContext;` to the top of your files. To make these members look like special operations, they use the camelCase naming convention, violating .NET naming conventions, which require PascalCase.

These members are:

* `dynamic proceed()`: Gives control to the original code of the method the template is being applied to. When multiple advices per method are supported, this will instead give control to the next template in line, if there are any left.
* `ITemplateContext target { get; }`: Gives access to information about the code element the template is being applied to.
* `T compileTime<T>( T expression )`: Informs the templating engine that this expression should be considered to be compile-time, even when it normally would not. Other than that, the input value is returned unchanged.


## Packaging an aspect

There is nothing special about creating a NuGet package for a project that contains Caravela.Framework aspects, it works the same as creating a NuGet package for a regular .NET library.