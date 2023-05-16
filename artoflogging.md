# The Art of Logging

- [The Art of Logging](#the-art-of-logging)
  - [What is Logging](#what-is-logging)
  - [Simple Logging with Metalama](#simple-logging-with-metalama)
    - [Getting Started](#getting-started)
    - [An Interpolated String Builder](#an-interpolated-string-builder)
    - [Getting Started (reprieved)](#getting-started-reprieved)
    - [A Console for Testing](#a-console-for-testing)
    - [Refining our log entries](#refining-our-log-entries)
    - [Adding a Fabric to make life even easier](#adding-a-fabric-to-make-life-even-easier)
    - [Conclusion](#conclusion)
  - [Adding Complexity to Logging](#adding-complexity-to-logging)
    - [Timing](#timing)
    - [Debug versus Runtime Logging](#debug-versus-runtime-logging)
  - [Real World Logging with Metalama](#real-world-logging-with-metalama)
    - [Introduction](#introduction)
    - [ILogger](#ilogger)
    - [Sensible Naming and Documentation](#sensible-naming-and-documentation)
    - [Points of Interest](#points-of-interest)

## What is Logging

At its simplest logging could be described as being a form of running commentary on an application and that commentary can be directed to the console or some external place such as a file on the local hard drive.

Consider for a moment the following method.

```c#
public decimal Divide (int a, int b)
{
		return a / b;
}
```

It’s clearly somewhat contrived but it’s a good example to describe what we might need to do in order to obtain that ‘running commentary’ that we want. Let’s start with knowing that we have entered and exited the method.
Incidentally I used [Linqpad](https://www.linqpad.net/) for this example as it makes a great scratch pad for testing little bits of code like this.

I’ll add a couple of lines of code to the method to tell me when it’s entered and exited.

```c#
public decimal Divide (int a, int b)
{
	Console.WriteLine("Entering the Divide Method.");
		return a / b ;
	Console.WriteLine("Leaving the Divide Method.");
}
```

That probably took me about a minute to do, although a competent touch typist could well be quicker, however remember that minute.
If the code is run the following is output to the console.

Entering the Divide Method.

We now know when we have entered the method but once the method returns the result anything else is out of scope and won’t get called so we don’t actually know that it has finished. I could get around that by introducing a try / finally block.

```c#
public decimal Divide (int a, int b)
{
	Console.WriteLine("Entering the Divide Method.");
	try
	{
		return a / b;
	}
	finally
	{
		Console.WriteLine("Leaving the Divide Method.");
	}
}
```

This produces the following result.

Entering the Divide Method.<br>
Leaving the Divide Method.

That’s probably added another minute of additional work to my method. Now it would probably be useful to know what parameters are actually being passed to this method and also what result is being returned. Time to adjust our method once again.

````c#
public decimal Divide (int a, int b)
{
	Console.WriteLine($"Entering the Divide Method, with the following parameters; a = {a} and b = {b}.");
	decimal result = 0;
	try
	{
	    result = a / b;
		return result;
	}
	finally
	{
		Console.WriteLine($"Leaving the Divide Method, with the following result: {result}.");
	}
}

This produces the following;

Entering the Divide Method, with the following parameters; a = 2 and b = 1.
Leaving the Divide Method, with the following result 2.

Now that’s much more informative but we’ve had to add a variable to contain the result so that we could report it and rejig out two WriteLine statements.  Probably another couple of minutes used up.
There’s still and obvious flaw though that we haven’t covered, suppose the method throws an error. We will need to know about that.  Time for some more alterations.

```c#endregionpublic decimal Divide (int a, int b)
{
	Console.WriteLine($"Entering the Divide Method, with the following parameters; a = {a} and b = {b}.");
	decimal result = 0;
	try
	{
	    result = a / b;
	    return result;
	}
	catch (Exception e)
	{
		Console.WriteLine($"The Divide method encountered an error.  The error message was {e.Message}.");
		return result;
	}
	finally
	{

		Console.WriteLine($"Leaving the Divide Method. with the following result: {result}.");
	}
}
````

Now id we contrive a divide by zero exception by passing in 2 and 0 as parameters we get the following;

Entering the Divide Method, with the following parameters; a = 2 and b = 0.<br>
The Divide method encountered an error. The error message was Attempted to divide by zero.<br>
Leaving the Divide Method. with the following result: 0.

That is much more informative but we’ve had to add an awful lot of additional code to the method to get some meaningful commentary out. If we want a meaningful running commentary on our application we’re going to have to do that to every method that we want to keep track of. If we say that all of the changes would probably have added an extra two or so minutes work for a proficient touch typist, once multiplied over several methods in each project of a solution that time soon mounts up and in reality this is really pretty straightforward boiler plate code.

You can start to see why so many developers, despite appreciating what logging can do for them, don’t actually go to the trouble of fully logging their work. Surely there must be an easier way to do this. Fortunately there is.

## Simple Logging with Metalama

Without going into too much detail here as to how Metalama does what it does we will say that essentially Metalama is able to add code to your code at compile time. By decorating your code with an attribute, or attributes, you effectively instruct Metalama to execute a pre-defined template over the code that you’ve just decorated with said attribute(s).
As has already been mentioned in the introduction there will be a freely available solution that implements Metalama logging. With that in mind the steps that we take now may seem a little over the top but they are what you would want to be doing if you decide to create your own custom logging solution with Metalama.

To begin with we’ll create a simple logging solution that provides exactly the same amount of information as we have previously hand coded into our method. To remind ourselves we want to know which method we’re in, what parameters, if any, were passed to it and when the method was exited and what, if any, result was passed onwards. As with our previous handwritten example we will for the moment confine our output to the terminal console window.
We will do this with the aid of what in Metalama is referred to as an aspect. When we have finisher writing it we will be able to decorate any block of code that we want logging applied to with a simple [Log] attribute and at compile time Metalama will, as if by magic, insert all the necessary code that needs to be added to provide us with the logging that we want.
For the sake of clarification I’ll be working in the latest version of Visual Studio (Visual Studio 2022 v17.5 at the time of writing) with the .Net 6 SDK installed and the [Metalama Visual Studio Extension](https://marketplace.visualstudio.com/items?itemName=PostSharpTechnologies.Metalama).

### Getting Started

We’ll start off by creating a new Solution / Project. It will be a class library based on .Net Standard 2.0 and it will be named ‘<company>.Metalama.Logging’. In my case that will equate to VtlSoftware.Metalama.Logging. We’ll use .Net Standard as our base as that will provide us with the maximum potential to reuse this across a wide variety of .net based projects of our own, and we’ll set the C# language version to 10.0.

Once created add the Metalama.Framework Nuget package to the project (selecting the latest stable version).
Now delete the default Class1 and add a new class called LogAttribute.

```c#
namespace VtlSoftware.Metalama.Logging
{
    public class LogAttribute
    {
    }
}
```

We want to log methods so we’ll have this class inherit from Metalama’s OverrideMethodAspect, which in turn will require that we actively implement the OverrideMethod() method.

```c#
   public class LogAttribute : OverrideMethodAspect
    {
        public override dynamic? OverrideMethod()
        {
            throw new NotImplementedException();
        }
    }

```

**Note that dynamic is nullable**

At this point well take a slight diversion and create a helper method that will construct our log strings.

### An Interpolated String Builder

Create a new static method BuildInterpolatedString that returns a Metalama InterpolatedStringBuider. In its most basic form it will look like this;

```c#
private static InterpolatedStringBuilder BuildInterpolatedString(bool includeOutParameters)
        {
            var stringBuilder = new InterpolatedStringBuilder();

            // Include the type and method name.
            stringBuilder.AddText(meta.Target.Type.ToDisplayString(CodeDisplayFormat.MinimallyQualified));
            stringBuilder.AddText(".");
            stringBuilder.AddText(meta.Target.Method.Name);
            stringBuilder.AddText("(");


            return stringBuilder;
        }
```

The most important thing to understand from what has just been added is the Metalama class meta and its member Target. In the Metalama documentation it is defined as ‘the entry point for the meta model which can be used in templates to inspect the target code or access other features of the template language’.

In this case meta.Target will refer to the code that has the [Log] attribute added to it. The meta.Target.Type will refer to the class in which the method being logged resides and the meta.Target.Method.Name should, one hopes, be self- explanatory. Just a quick note about the CodeDisplayFormat. In it’s MinimallyQualified form we’ll get just the `<type name>.<method name>` back. If we were to opt for FullyQuallified we’d get `<namespace or alias name>.<type name>.<method name>`.

```c#
        private static InterpolatedStringBuilder BuildInterpolatedString(bool includeOutParameters)
        {
            var stringBuilder = new InterpolatedStringBuilder();

            // Include the type and method name.
            stringBuilder.AddText(meta.Target.Type.ToDisplayString(CodeDisplayFormat.MinimallyQualified));
            stringBuilder.AddText(".");
            stringBuilder.AddText(meta.Target.Method.Name);
            stringBuilder.AddText("(");


            // Include a placeholder for each parameter.
            var i = meta.CompileTime(0);

            foreach (var p in meta.Target.Parameters)
            {
                var comma = i > 0 ? ", " : "";

                if (p.RefKind == RefKind.Out && !includeOutParameters)
                {
                    // When the parameter is 'out', we cannot read the value.
                    stringBuilder.AddText($"{comma}{p.Name} = <out> ");
                }
                else
                {
                    // Otherwise, add the parameter value.
                    stringBuilder.AddText($"{comma}{p.Name} = {{");
                    stringBuilder.AddExpression(p.Value);
                    stringBuilder.AddText("}");
                }

                i++;
            }

            stringBuilder.AddText(")");

            return stringBuilder;
        }
```

Now that you have a basic understanding of the meta class you can probably work out for yourself exactly what is being done here. Essentially if the method has parameters we are reading through them one by one, obtaining both their name and value, and adding them to our stringBuilder. Because we can’t read the value of ‘out’ parameters those, if there are any, are being specifically excluded.

### Getting Started (reprieved)

Having created a helper method to construct an informative string message containing the basic name information of our method and its parameters we can return to our LogAttribute and start to flesh that out. We’ll start by creating the message that provides that information, and we’ll be doing that in the overridden OverrideMethod().

```c#
public override dynamic OverrideMethod()
        {
            // Write entry message.
            var entryMessage = BuildInterpolatedString(false);
            entryMessage.AddText(" started.");
            Console.WriteLine(entryMessage.ToValue());
        }

```

Next we’ll want to run the actual method code. If the method is void (it won’t return anything) then we’re just interested in knowing if it succeeded or failed. If it actually returns something we need to know what it returned in addition to whether or not it failed.
A try catch block will allow us to determine failure and report back on it. Our overridden method should now look this.

```c#
public override dynamic OverrideMethod()
        {
            // Write entry message.
            var entryMessage = BuildInterpolatedString(false);
            entryMessage.AddText(" started.");
            Console.WriteLine(entryMessage.ToValue());

            try
            {
                // Invoke the method and store the result in a variable.
                var result = meta.Proceed();

                // Display the success message. The message is different when the method is void.
                var successMessage = BuildInterpolatedString(true);

                if (meta.Target.Method.ReturnType.Is(typeof(void)))
                {
                    // When the method is void, display a constant text.
                    successMessage.AddText(" succeeded.");
                }
                else
                {
                    // When the method has a return value, add it to the message.
                    successMessage.AddText(" returned ");
                    successMessage.AddExpression(result);
                    successMessage.AddText(".");
                }

                Console.WriteLine(successMessage.ToValue());

                return result;
            }
            catch (Exception e)
            {
                // Display the failure message.
                var failureMessage = BuildInterpolatedString(false);
                failureMessage.AddText(" failed: ");
                failureMessage.AddExpression(e.Message);
                Console.WriteLine(failureMessage.ToValue());

                throw;
            }
        }

```

In the try part of the block we have just added we are using meta.Proceed to actually run our method, and depending upon whether or not it is a void we go on to report success as either a plain “succeeded” where there is no return value to log or “returned” where there is.

Potential failure is catered for in the catch block and here we log a “failed” message along with the actual failure message to give us some idea as to why it actually failed.

That’s pretty much it. We’ll build our project and if successful build a release version.

### A Console for Testing

With that done let’s create a new Visual studio solution. This time were going to create a .Net 6 console application and we’ll add as a reference our VtlSoftware.MetaLama.Logging.dll from the release bin of our Metalama.Logging solution. You will also need to add a direct reference to the Metalama.Framework Nuget Package.

Now in the new console app add a new class called Calculator and within that four simple methods.

```c#
internal static class Calculator
    {

        public static double Add(double x, double y) => x + y;


        public static double Subtract(double x, double y) => x - y;


        public static double Divide(double x, double y) => x / y;


        public  static int DivideInt(int  x, int y) => x / y;
    }
```

As we have a reference to out Metalama.Logging project we can add logging to these methods really easily by the simple process of adding a [Log] attribute.

```c#
internal static class Calculator
    {
        [Log]
        public static double Add(double x, double y) => x + y;

        [Log]
        public static double Subtract(double x, double y) => x - y;

        [Log]
        public static double Divide(double x, double y) => x / y;

        [Log]
        public  static int DivideInt(int  x, int y) => x / y;
    }

```

In Visual Studio (but only visual studio as it alone can utilise the Metalama extension for Visual Studio) place the current on the name of one of your methods and invoke the right click context menu. Once it opens select ‘Show Metalama diff’.

If we look at the Add method in the new code window that has opened we can see the code that Metalama has added to our method to enable logging. Moreover if we scroll through all of the methods we can see consistent boilerplate logging code has been added to each method to which we added the [Log] attribute.

```c#
        [Log]
        public static double Add(double x, double y)
        {
            Console.WriteLine($"Calculator.Add(x = {{{x}}}, y = {{{y}}}) started.");
            try
            {
                double result;
                result = x + y;

                Console.WriteLine($"Calculator.Add(x = {{{x}}}, y = {{{y}}}) returned {result}.");
                return (double)result;
            }
            catch (Exception e)
            {
                Console.WriteLine($"Calculator.Add(x = {{{x}}}, y = {{{y}}}) failed: {e.Message}");
                throw;
            }

        }
```

I’m sure you’ll agree that that is impressive. More to the point that’s a lot of code that you haven’t actually had to write yourself.

Let’s quickly put this to the test.
In the program file in the main method add the following code.

```c#
    static void Main(string[] args)
        {
            try
            {
                Calculator.Add(1, 1);
                Calculator.Subtract(2, 1);

                // the following will report infinity as the result as we are dealing with doubles
                Calculator.Divide(2, 0);

                //the following will report a Divide by zero exception as we're dealing with integers
                Calculator.DivideInt(2, 0);
            }
            catch
            {

            }
        }
```

Run the program and observe the logging sent to the console.

Calculator.Add(x = {1}, y = {1}) started.<br>
Calculator.Add(x = {1}, y = {1}) returned 2.<br>
Calculator.Subtract(x = {2}, y = {1}) started.<br>
Calculator.Subtract(x = {2}, y = {1}) returned 1.<br>
Calculator.Divide(x = {2}, y = {0}) started.<br>
Calculator.Divide(x = {2}, y = {0}) returned 8.<br>
Calculator.DivideInt(x = {2}, y = {0}) started.<br>
Calculator.DivideInt(x = {2}, y = {0}) failed: Attempted to divide by zero. <br>

By all accounts that’s pretty impressive and all you had to do was add a log attribute to each method you wanted to see a log entry for. It should be apparent that you can save a lot of time by using Metalama to provide logging for your application(s).

### Refining our log entries

We’ve seen how efficiently our method parameters have been returned but that might not always be such a good idea. What about methods that handle things like passwords? Do we really want to pass our passwords as unredacted plain strings into logs that in theory, at least, anyone could read? There could be serious, possibly even legal implications if we do that, so let’s look at ways to avoid that sort of thing happening.

We’ll start by adding a new static class to our solution called Strings, to which we’ll add a contrived Login method and add our log attribute to it, finally calling it from our main method.

```c#
internal static class Strings
    {
        [Log]
        public static void Login(string username, string password)
        {
            Console.WriteLine("This is the Login Method");
        }
    }

   static void Main(string[] args)
        {
            try
            {
                //Calculator.Add(1, 1);
                //Calculator.Subtract(2, 1);

                //// the following will report infinity as the result as we are dealing with doubles
                //Calculator.Divide(2, 0);

                ////the following will report a Divide by zero exception as we're dealing with integers
                //Calculator.DivideInt(2, 0);

                Strings.Login("dom", "mySuperSecretPassword");
            } catch
            {
            }
        }
```

When we run this we get;

Strings.Login(username = {dom}, password = {mySuperSecretPassword}) started.<br>
This is the Login Method<br>
Strings.Login(username = {dom}, password = {mySuperSecretPassword}) succeeded.<br>

Oops!

To correct this we’ll return to our Metalama.Logging project and add a new static class SensitiveParameterFilter, which we’ll decorate with the Metalama [CompileTime] attribute to ensure that this class is available to Metalama at compile time only (where it will be of the most use to us).

Within that we’ll add the following;

```c#
[CompileTime]
    internal static class SensitiveParameterFilter
    {


        private static readonly string[] _sensitiveNames = new[] { "password", "credential", "pwd" };

        public static bool IsSensitive(IParameter parameter)
        {
            if(parameter.Attributes.OfAttributeType(typeof(NotLoggedAttribute)).Any())
            {
                return true;
            }

            if(_sensitiveNames.Any(n => parameter.Name.ToLowerInvariant().Contains(n)))
            {
                return true;
            }

            return false;
        }

    }
```

Let’s look at what is going on here. To begin with we creating a static read only string array of what we will consider to be potential parameter names for parameters that we might wish to either avoid logging altogether or if we do log them do so in a way in which their actual value is redacted.
NB As is, this array is fixed. To be really useful it ought to be dynamic and possibly read from an external source.
Next up we have a Boolean method IsSensitive that returns true if either a parameter is marked with the [NotLogged] attribute (see below) or the parameter name is in our list of names declared earlier.

```c#
[AttributeUsage(AttributeTargets.Parameter | AttributeTargets.ReturnValue ]
    public sealed class NotLoggedAttribute : Attribute
    {
    }

```

Next we want to make an adjustment to our LogAttribute OverirideMethod() method

```c#
public override dynamic? OverrideMethod()
        {
            // Write entry message.
            var entryMessage = BuildInterpolatedString(false);
            entryMessage.AddText(" started.");
            Console.WriteLine(entryMessage.ToValue());

            try
            {
                // Invoke the method and store the result in a variable.
                var result = meta.Proceed();

                // Display the success message. The message is different when the method is void.
                var successMessage = BuildInterpolatedString(true);

                if(meta.Target.Method.ReturnType.Is(typeof(void)))
                {
                    // When the method is void, display a constant text.
                    successMessage.AddText(" succeeded.");
                } else
                {
                    // When the method has a return value, add it to the message.
                    successMessage.AddText(" returned ");

                    //check to see if we are dealing with sensitive data
                    if (SensitiveParameterFilter.IsSensitive(meta.Target.Method.ReturnParameter))
                    {
                        successMessage.AddText("<redacted>");
                    }
                    else
                    {
                        successMessage.AddExpression(result);
                    }

                    successMessage.AddText(".");
                }

                Console.WriteLine(successMessage.ToValue());

                return result;
            } catch(Exception e)
            {
                // Display the failure message.
                var failureMessage = BuildInterpolatedString(false);
                failureMessage.AddText(" failed: ");
                failureMessage.AddExpression(e.Message);
                Console.WriteLine(failureMessage.ToValue());

                throw;
            }
        }
```

To finish off we also need to make a small adjustment to our BuildInterpolatedString method to take our new IsSensitve Boolean into consideration.

```c#
   private static InterpolatedStringBuilder BuildInterpolatedString(bool includeOutParameters)
        {
            var stringBuilder = new InterpolatedStringBuilder();

            // Include the type and method name.
            stringBuilder.AddText(meta.Target.Type.ToDisplayString(CodeDisplayFormat.MinimallyQualified));
            stringBuilder.AddText(".");
            stringBuilder.AddText(meta.Target.Method.Name);
            stringBuilder.AddText("(");

            // Include a placeholder for each parameter.
            var i = meta.CompileTime(0);

            foreach(var p in meta.Target.Parameters)
            {
                var comma = i > 0 ? ", " : string.Empty;

                if(SensitiveParameterFilter.IsSensitive(p))
                {
                    // Do not log sensitive parameters.
                    stringBuilder.AddText($"{comma}{p.Name} = <redacted> ");
                }
                else if(p.RefKind == RefKind.Out && !includeOutParameters)
                {
                    // When the parameter is 'out', we cannot read the value.
                    stringBuilder.AddText($"{comma}{p.Name} = <out> ");
                } else
                {
                    // Otherwise, add the parameter value.
                    stringBuilder.AddText($"{comma}{p.Name} = {{");
                    stringBuilder.AddExpression(p.Value);
                    stringBuilder.AddText("}");
                }

                i++;
            }

            stringBuilder.AddText(")");

            return stringBuilder;
        }
```

Rebuild the Metalama.Logging project.

Return to the console project and amend the Strings class and the main method and then run it.

```c#
internal static class Strings
    {


        [Log]
        public static void Login(string username, string password) { Console.WriteLine("This is the Login Method"); }

        [Log]
        public static void MyNewLogin(string username, [NotLogged] string secret)
        { }


    }
```

```c#
static void Main(string[] args)
        {
            try
            {
                //Calculator.Add(1, 1);
                //Calculator.Subtract(2, 1);

                //// the following will report infinity as the result as we are dealing with doubles
                //Calculator.Divide(2, 0);

                ////the following will report a Divide by zero exception as we're dealing with integers
                //Calculator.DivideInt(2, 0);

                Strings.Login("dom", "mySuperSecretPassword");
                Strings.MyNewLogin("dom", "mySuperSecret");
            } catch
            {
            }
        }

```

Strings.Login(username = {dom}, password = <redacted> ) started.<br>
This is the Login Method<br>
Strings.Login(username = {dom}, password = <redacted> ) succeeded.<br>
Strings.MyNewLogin(username = {dom}, secret = <redacted> ) started.<br>
Strings.MyNewLogin(username = {dom}, secret = <redacted> ) succeeded.<br>

Now that is much better, and implementation to protect sensitive data will be easy.
However, I’m a naturally lazy individual and as I don’t touch type the mere thought of having to add a [Log] attribute to every method in every class that I want to be logged fills me with dread. Surely there’s an easier way?
As luck would have it there is. Metalama has a little magic trick up it’s sleeve that it calls Fabrics. Essentially these are relatively small bits of code that we can write to weave code into our target project(s).

### Adding a Fabric to make life even easier

Return to your Metalama.Logging solution and add a class called Fabric to it containing the following code.

```c#
   public class Fabric : TransitiveProjectFabric
    {
        public override void AmendProject(IProjectAmender amender)
        {
            amender.Outbound
            .SelectMany(compilation => compilation.AllTypes)
            .Where(type => type.Accessibility is Accessibility.Public or Accessibility.Internal)
            .SelectMany(type => type.Methods)
            .Where(method => method.Accessibility == Accessibility.Public && method.Name != "ToString")
            .AddAspectIfEligible<LogAttribute>();
        }
    }
```

There’s quite a lot going on here and it will pay us to discuss it in a little more detail.

The Metalama documentation describes a Fabric as ‘a class that, when inherited by a type in a project (under any name or namespace), allows that type to analyse and add aspects to that project’. Whilst that description is technically accurate it may not be entirely obvious to the reader what is actually being described here.

Our Metalama.Logging project has been designed with a view to allowing other projects and solutions that we create in the future to inherit it and thus provide that magic ingredient that is Metalama logging. Fabrics take that magic a little further. When we add a Fabric to our Logging project and that project is in turn referenced by other projects, those projects will at compile time, be able to to be amended by the Fabric.

Returning to the fabric that we have just created. It inherits from TransitiveProjectFabric (so it will be applied to multiple projects in their entirety. We are required to override the AmendProject() provided by the ProjectFabric and in so doing we will instruct the fabric as to what we want it to do to the projects that it has access to.

In this particular case we are instructing the Fabric to select all items in our project that are Methods and have a ‘public’ signature and whose name isn’t equal to ‘ToString’. We need to avoid logging ToString methods because doing so could cause an infinite recursion.

We are then saying that if there are methods in our project that meet the criteria that we have specified then the Fabric should go ahead and add the [Log} attribute to those classes. If you’ve grasped the significance of what has just been described then you’ll already have some idea of just how powerful Fabric’s can be.

Now rebuild the Metalama.Logging project and when that is done return to the console project. We’ll add two new classes to this (NewCalculator and NewStrings) and in each we’ll replicate the methods that we added in their previous incarnations, only this time we’ll not add the [Log] attribute to them. Amend the Main method to call your new classes (taking care to comment out the Calculator.DivideInt method as calling that early on will terminate the app as we designed the method to throw an error). If all goes well you see numerous new log entries from classes and methods that you haven’t had to manually edit with the [Log] attribute.

This alone should be sufficient to prove to you just how powerful Metalama is and what a time saver it can be. There is one more benefit which might just have slipped your attention. Your main application code is EXACTLY as it was. If you save it to a code repository like GitHub you’ll just be saving your code. It remains easily readable and its intention is still clear as there is no boiler plate code for you to negotiate your way around.

### Conclusion

As we have just seen, simple informative logging is easily achievable with Metalama and for relatively little effort on your part (even less if you make use of the Nuget package that goes with this pamphlet!). If you’re not logging comprehensively (at least in the development stage) then you’re doing yourself a disservice and making any debugging that you have to do that much more difficult.

## Adding Complexity to Logging

Thus far we’ve looked at implementing some very simple straightforward logging to a project and in the process seen that we can return a wealth of information about what is actually happening in our code. Let’s start to add some useful bits and pieces into the mix.

### Timing

There are occasions when knowing just how long it’s taking for something to happen can prove to be really useful and adding such a facility to our Log aspect is in fact relatively straight forward. However there is a cost involved in timing everything and frankly there are really only certain areas (getting information from an external service for example) where knowing how long the operation took might prove to be useful.

This is, arguably, something that the developer writing the code in the first place ought to know and therefore it’s reasonable to argue that asking them to specifically add an attribute to that one piece of code is not really that onerous a job. Given that we’ll create a separate logging aspect to providing a record of how long a method took to run, as well as all the other information we are currently collecting.

In the Metalama.Logging project create a new class called TimedLogAttribute which, just as with the original LogAttribute, will inherit from OverrideMethodAspect. To start with copy and paste the code inside your Log Attribute to this class (omitting the interpolatedStringBuilder). You’ll see an immediate error with the call to BuilInterpolatedString which we’ll rectify by moving that method to a static helper class so that we don’t have to constantly repeat it. You’ll need to decorate your helper class with the [CompileTime] attribute.

With that done we’ll make a couple of small changes to our new TimedLogAttribute.

```c#
  public class TimedLogAttribute : OverrideMethodAspect
    {

        public override dynamic OverrideMethod()
        {
            // Write entry message.
            var entryMessage = LogHelpers.BuildInterpolatedString(false);
            entryMessage.AddText(" started.");
            Console.WriteLine(entryMessage.ToValue());

            //add a means to time the method's execution
            var stopwatch = new Stopwatch();
            stopwatch.Start();

            try
            {
                // Invoke the method and store the result in a variable.
                var result = meta.Proceed();

                // Display the success message. The message is different when the method is void.
                var successMessage = LogHelpers.BuildInterpolatedString(true);

                if(meta.Target.Method.ReturnType.Is(typeof(void)))
                {
                    // When the method is void, display a constant text.
                    successMessage.AddText(" succeeded.");
                } else
                {
                    // When the method has a return value, add it to the message.
                    successMessage.AddText(" returned ");

                    //check to see if we are dealing with sensitive data
                    if(SensitiveParameterFilter.IsSensitive(meta.Target.Method.ReturnParameter))
                    {
                        successMessage.AddText("<redacted>");
                    } else
                    {
                        successMessage.AddExpression(result);
                    }

                    successMessage.AddText(".");
                }

                Console.WriteLine(successMessage.ToValue());

                return result;
            } catch(Exception e)
            {
                // Display the failure message.
                var failureMessage = LogHelpers.BuildInterpolatedString(false);
                failureMessage.AddText(" failed: ");
                failureMessage.AddExpression(e.Message);
                Console.WriteLine(failureMessage.ToValue());

                throw;
            } finally
            {
                //we need to know how long all of this took
                stopwatch.Stop();
                var elapsedMessage = LogHelpers.BuildInterpolatedString(false);
                elapsedMessage.AddText(" elapsed: ");
                elapsedMessage.AddExpression(stopwatch.ElapsedMilliseconds);
                elapsedMessage.AddText("ms");
                Console.WriteLine(elapsedMessage.ToValue());
            }
        }
    }

```

It should be pretty obvious what is going on here so once you’ve made these changes go to the Fabric class and change <LogAttribute> to <TimedLogAttribute>. Now build the Metalama.Logging project, then return to the console project and rebuild that and then run it. Observe the new details that are now in the log.
In reality we don’t want everything to be timed so let’s make a slight adjustment to our LogAttribute.

Add the following method to the LogAttribute Class.

```c#
   public override void BuildAspect(IAspectBuilder<IMethod> builder)
        {
            if(!builder.Target.Attributes.OfAttributeType(typeof(TimedLogAttribute)).Any())
            {
                    builder.Advice.Override(builder.Target, nameof(this.OverrideMethod));
            }
        }

```

In essence we are ensuring that the LogAttribute will not be added to any Methods that are already decorated with our new [TimedLog] attribute.

Go back to the Fabric class and reverse the previous adjustment that you made there and now rebuild your Metalama.Logging project.

Return to your console project and very specifically add a [TimedLog] attribute to one of the methods therein. Build and run the application. Notice that all but the one method to which you added the [TimedLog] attribute are returning logs without timing info (so we know that they are being decorated with the [Log] attribute by the Fabric. However the one method that we manually added the [TimedLog] attribute to is providing additional information to the log about how long it took to execute.

### Debug versus Runtime Logging

Thus far we have been concentrating our efforts on producing comprehensive logs that will provide us as developers with as much information as possible whilst we actually develop our applications (which we will typically be doing in debug mode). Naturally we all like to think that we write bug free and hence exception free code but the harsh reality is that we don’t. It’s inevitable that some errors will make it to release mode. It would be really useful to have those logged but avoid logging all the extra stuff that went into our logs during development. Metalama provides a way for us to do that.

Lets amend our BuildAspect method in the LogAttribute Class.

```c#
   public override void BuildAspect(IAspectBuilder<IMethod> builder)
        {
            if (!builder.Target.Attributes.OfAttributeType(typeof(TimedLogAttribute)).Any())
            {
                if (builder.Target.Compilation.Project.Configuration == "Debug")
                {
                    builder.Advice.Override(builder.Target, nameof(this.OverrideMethod));
                }
                else
                {
                    builder.Advice.Override(builder.Target, nameof(this.CondensedOverrideMethod));
                }
            }
        }

```

Notice that we are now adding an additional condition to check for the actual build configuration. If it’s a debug build we’ll go on to use the original OverrideMethod to add the code we want added to log in detail, but if it’s release we use an alternative method to provide some slimmed down logging.

```c~
[Template]
        public dynamic CondensedOverrideMethod()
        {

            try
            {
                // Invoke the method and store the result in a variable.
                var result = meta.Proceed();
                return result;
            } catch(Exception e)
            {
                // Display the failure message.
                var failureMessage = LogHelpers.BuildInterpolatedString(false);
                failureMessage.AddText(" failed: ");
                failureMessage.AddExpression(e.Message);
                Console.WriteLine(failureMessage.ToValue());

                throw;
            }
        }

```

If you do some experimentation in your Console project (having obviously first rebuilt your Metalama.Logging project) and switch between running Debug and Release builds you’ll immediately notice the difference in what is actually logged.

In reality taking this approach with logging is probably not the best idea. What is really needed is a way to switch logging levels at runtime and we will cover that a little later on but this exercise has served to illustrate the control that you can bring to the code that Metalama will generate on your behalf.

## Real World Logging with Metalama

Now that we have seen what we can do let's actually refine this into a real world example that can be used.

All of the code examples that follow are contained within an online public [GitHub Repository](https://github.com/domsinclair/VtlSoftware.LoggingWithMetalama) and the class libraries that are discussed will be available on Nuget.

This means in effect that you have no need to actually create these libraries on your own, however if you do you should in reality gain form the experience because it will broaden your knowledge about how Metalama works and how you could use it to make your own day to day programming that bit easier.

As the libraries are the main source for this document the actual Nuget Packages will not be available until work on this document and by extension the supporting code is complete. As soon as they are available though links to them will appear just below here.

### Introduction

Logging directly to the console window, as we have done up to now, if fine for development work and where the logging being done is relatively minor.

That though is unlikely to be the case. Logs provide an invaluable tool in the debugging process and in reality, especially during the development phase, developers will want to log as much information about their application as they can.

This is where Logging Frameworks come into their own.

We'll start by using The `ILogger` interface and as we will find out in due course it will allow us to easily substitute a Logging Framework of our own choice.

The code that we'll be discussing can be found in the Vtl.LogToConsole project in the repository. The principles that we'll apply are much as we've already discussed but refined in such a way as to make them better fit the structure of a class library created with reuse in mind.

The class libraries you'll find in the repository are all built on .Net Standard 2.0. This offers the greatest flexibility for using them against both .Net Framework and .Net Core applications. The test apps included are themselves based on .Net Core (6.0).

The test application is just a simple console app and once again logging will be directed to the console but it will illustrate the basics well and provide a firm building block.

### ILogger

If you're familiar with the ILogger interface provided by [Microsoft.Extensions.Logging](https://www.nuget.org/packages/Microsoft.Extensions.Logging/8.0.0-preview.3.23174.8#versions-body-tab) (available through NuGet) then you can skip this section but if ILogger is new to you then it is worth you learning a little about it first.

There is a wealth of [documentation](https://learn.microsoft.com/en-us/dotnet/core/extensions/logging?tabs=command-line) within microsoft's own documentation site but this can be a little dry.

If you want a slightly more down to earth explanation of what it is and why it's worth using then Tim Corey produced an excellent [video on YouTube](https://www.youtube.com/watch?v=oXNslgIXIbQ) that explains the basic principles very well.

In essence what ILogger brings to the table is both a way to make use of some extensive logging capabilities that Microsoft has itself already baked into .Net but it rather neatly opens to door for you, the developer, to then substitute a logging Framework of your own choice to provide logging, opening up the possibility of logging to a range of different places and also creating structured logs.

> <span style="color:yellow">It should be noted that Microsoft's ILogger does itself provide structured logging but in reality providers like [Serilog](https://serilog.net/) tend to do it a litter more comprehensively.<span>

Using `Ilogger` will require a reference to it in every single class that we want to log. There are basically two ways of doing this.

The first is manually. That would require that we add, at the very least, the following to each and every class that we create.

```c#
   internal class TestClass
    {
        private readonly ILogger<TestClass> logger;

        public TestClass(ILogger<TestClass> logger)
        {
            this.logger = logger;
        }

    }
```

This implies that we would probably need some sort of check that this is present at the point that we add our [Log] attribute to provide logging.

We could on the other hand use Dependency injection to do this.

There are two good basic examples of how to do this in the [MetaLama Samples](https://github.com/postsharp/Metalama.Samples). Take a look specifically at Log Samples 4 and 5. Notice how sample 5 provides some examples of Diagnostic verification that you can add to aspects to ensure that they are used correctly.

I personally chose to go down the Dependency Injection route, but either approach is perfectly acceptable.

> <span style="color:yellow">There is one point that you should note carefully. ILogger can't be used in Static Classes as they don't have a constructor and by extension no means to introduce the ILogger, either manually or via Dependency Injection.<span><br>
>
> <span style="color:yellow">This means that you will need to ensure that end users of the class library know that they won't be able to log static classes (and obviously any static methods that they contain) with any class Library that you provide that utilises ILogger.<span>
>
> <span style="color:yellow">You can find reference, by doing searhes on the internet, to ways that you could circumvent this. This isn't the place to discuss such approaches but we will cover how to avoid automatically adding the attribute to static methods when using a Fabric to cover a project.<span>

### Sensible Naming and Documentation

This may seem blindingly obvious but by considering the names that you apply to the various aspects that you create you will make it much more apparent to the end user what it is that they are;

- Intended to do.
- Where they are meant to be applied.

It also makes good sense to document the aspects that you create thus providing your end users with some useful help via intellisense.

### Points of Interest

With that said let's look at some of the points of Interest arising from applying some real world implementation to the basics of the simple logging discussed earlier.
