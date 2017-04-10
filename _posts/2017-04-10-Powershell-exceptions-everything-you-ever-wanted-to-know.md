---
layout: post
title: "PowerShell: Everything you wanted to know about exceptions"
date: 2017-04-10
tags: [PowerShell,Exceptions,Basic]
---

Error handling is just part of life when it comes to writing code. We can often check and validate conditions for expected behavior. When the unexpected happens, we turn to exception handling. You can easily handle exceptions generated by other people's code or you can generate your own exceptions for others to handle.

# Index

* TOC
{:toc}

# Basic terminology

We need to cover some basic terms before we jump into this one.

## Exception

An Exception is like an event that is created when normal error handling can not deal with the issue. Trying to divide a number by zero or running out of memory are examples of something that will create an exception. Sometimes the author of the code you are using will create exceptions for certain issues when they happen.

## Throw and Catch

When an exception like event happens, we say that an exception is thrown. To handle a thrown exception, you need to catch it. If an exception is thrown and it is not caught by something, the script will crash.

## The call stack

The call stack is the list of functions that have called each other. When a function is called, it gets added to the stack or the top of the list. When the function exits or returns, it will be removed from the stack.

When an exception is thrown, that call stack is checked in order for an exception handler to catch it.

## Terminating and non-terminating errors

An exception is a terminating error. A thrown error will either be caught or it will exit the current execution. By default, a non-terminating error is generated by `Write-Error` and it adds an error to the output stream without throwing an exception.

I point this out because `Write-Error` and other non-terminating errors will not trigger the catch script to execute.

## Eatting an exception

This is when you catch an error just to suppress it. Do this with caution because it can make troubleshooting issues very difficult. 

# Basic command syntax

Here is a quick overview of the basic exception handling syntax used in PowerShell.

## Throw

To create our own exception event, we throw an exception with the `throw` keyword.

    function Do-Something
    {
        throw "Bad thing happened"
    }

This creates a runtime exception that is a terminating error. It will be handled by a `catch` in a calling function or exit the script.

## Try/Catch

The way exception handling works in PowerShell (and many other languages) is that you first `try` a section of code and if it throws an error, you can `catch` it. Here is a quick sample.

    try
    {
        Do-Something
    }
    catch
    {
        Write-Output "Something threw an exception"
    }

The `catch` script only runs if there is an exception. If the `try` executes correctly, then it will skip over the `catch`.

## Try/Finally

Sometimes you don't need to handle an error but still need some code to execute if an exception happens or not. A `finally` script does exactly that.

Take a look at this example:

    $command = [System.Data.SqlClient.SqlCommand]::New(queryString, connection)
    $command.Connection.Open()
    $command.ExecuteNonQuery()
    $command.Connection.Close()

Any time you open or connect to a resource, you should close it. If the `ExecuteNonQuery()` throws an exception, the connection will not get closed. Here is the same code inside a try/finally block.

    $command = [System.Data.SqlClient.SqlCommand]::New(queryString, connection)
    try
    {         
        $command.Connection.Open()
        $command.ExecuteNonQuery()
    }
    finally
    {
        $command.Connection.Close()
    }

In this example, the connection will get closed if there is an error. It will also get closed if there is no error. The `finally` script will run every time.

Because you are not catching the exception, it will still get thrown up the call stack.

## Try/Catch/Finally

It is perfectly valid to use `catch` and `finally` together. Most of the time you will use one or the other, but you may find scenarios where you will use both.

# $PSItem

Now that we got the basics out of the way, we can dig a little deeper.

Inside the catch block, there is an automatic variable (`$PSItem` or `$_`) that contains the details about the exception. Here is a quick overview of some of the key properties.

## example script

This is the example script I used to generate the data used in the examples below. I knew `ReadAllText` would generate an error for me if I gave it a bad path. I set a breakpoint in the `catch` block so I could explore a live exception. 

    function Do-Something
    {
        [cmdletbinding()]
        param()
        Get-Resoruce
    }

    function Get-Resoruce
    {
        [cmdletbinding()]
        param()

        [System.IO.File]::ReadAllText( '\\test\no\filefound.log')
    }

    try
    {
        Do-Something
        Write-Output 'After do-something'
    }
    catch
    {
        Write-Output 'error handling'
        Write-Output $PSItem.ToString()
    }
    Write-Output 'After error handling'

## PSItem.ToString()

This will give you the cleanest message to use in logging and general output. `ToString()` is automatically called if `$PSItem` is placed inside a string.

    catch
    {
        Write-Output "Ran into an issue: $PSItem"
    }

## $PSItem.Exception

This is the actual exception that was thrown.

### $PSItem.Exception.Message

This is the general message that describes the exception and is a good starting point when troubleshooting. Most exceptions have a default message but can also be set to something custom when the exception is thrown. 

    PS:> $PSItem.Exception.Message

    Exception calling "ReadAllText" with "1" argument(s): "The network path was not found."

This is also the message that is returned when calling `$PSItem.ToString()`.

### $PSItem.Exception.InnerException

Exceptions can contain inner exceptions. This is often the case when the code you are calling catches an exception and throws a different exception. They will place the original exception inside the new exception.

    PS:> $PSItem.Exception.InnerExceptionMessage
    The network path was not found.

I will revisit this later when I talk about re-throwing exceptions.

### $PSItem.Exception.StackTrace

This is the `StackTrace` for the exception. I mentioned stack traces when I was defining terms, but I wanted to show one here:

    at System.IO.__Error.WinIOError(Int32 errorCode, String maybeFullPath)
    at System.IO.FileStream.Init(String path, FileMode mode, FileAccess access, Int32 rights, Boolean useRights, FileShare share, Int32 bufferSize, FileOptions options, SECURITY_ATTRIBUTES secAttrs, String msgPath, Boolean bFromProxy, Boolean useLongPath, Boolean checkHost)
    at System.IO.FileStream..ctor(String path, FileMode mode, FileAccess access, FileShare share, Int32 bufferSize, FileOptions options, String msgPath, Boolean bFromProxy, Boolean useLongPath, Boolean checkHost)
    at System.IO.StreamReader..ctor(String path, Encoding encoding, Boolean detectEncodingFromByteOrderMarks, Int32 bufferSize, Boolean checkHost)
    at System.IO.File.InternalReadAllText(String path, Encoding encoding, Boolean checkHost)
    at CallSite.Target(Closure , CallSite , Type , String )

You will only get this stack trace when the event is thrown from managed code. I am calling a .Net framework function directly so that is all we can see in this example. Generally when you are looking at a stack trace, you are looking for where you code stops and the system calls begin.

## $PSItem.InvocationInfo

This property contains additional information collected by PowerShell.exe about the function or script where the exception was thrown. Here is the `InvocationInfo` from the sample exception that I created.

    PS:> $PSItem.InvocationInfo | Format-List *

    MyCommand             : Get-Resource
    BoundParameters       : {}
    UnboundArguments      : {}
    ScriptLineNumber      : 5
    OffsetInLine          : 5
    ScriptName            : C:\blog\throwerror.ps1
    Line                  :     Get-Resource
    PositionMessage       : At C:\blog\throwerror.ps1:5 char:5
                            +     Get-Resource
                            +     ~~~~~~~~~~~~
    PSScriptRoot          : C:\blog
    PSCommandPath         : C:\blog\throwerror.ps1
    InvocationName        : Get-Resource

The important details here show the `ScriptName`, the `Line` of code and the `ScriptLineNumber` where the invocation started.

## $PSItem.ScriptStackTrace

This property will show the order of function calls that got you to the current script or function that generated the exception.

    PS:> $PSItem.ScriptStackTrace
    at Get-Resource, C:\blog\throwerror.ps1: line 13
    at Do-Something, C:\blog\throwerror.ps1: line 5
    at <ScriptBlock>, C:\blog\throwerror.ps1: line 18

I am only making calls to functions in the same script but this would track the calls if multiple scripts were involved.

# Working with exceptions

There is more to exceptions than the basic syntax and exception properties. 

## Catching typed exceptions

You can be selective of the exceptions that you catch. Exception have a type and you can specify the type of exception you want to catch.

    try
    {
        Do-Something -Path $path
    }
    catch [System.IO.FileNotFoundException]
    {        
        Write-Output "Could not find $path"
    }
    catch [System.IO.IOException]
    {
         Write-Output "IO error with the file: $path"
    }
    catch
    {
        Write-Output "Something else happened when processing: $path"
    }

The exception type is checked for each `catch` block until one is found that matches your exception. It is important to realize that exceptions can inherit from other exceptions. In the example above, `FileNotFoundException` inherits from `IOException`. So if the `IOException` was first, then it would get called instead. Only one catch block will be invoked even if there are multiple matches.

If we had a `System.IO.PathTooLongException` then the `IOException` would match but if we had a `InsufficientMemoryException` then the final default `catch` would be executed.

You also don't need to add that last default catch statement. You don't need to use it if all you are going to do is re-throw the exception.

## Throwing typed exceptions

You can also throw typed exceptions in PowerShell. Instead of calling `throw` with a string:

    throw "Could not find: $path"

Use an exception accelerator like this:

    throw [System.IO.FileNotFoundException] "Could not find: $path"

But you have to specify a message when you do it that way.

You can also create a new instance of an exception to be thrown. The message is optional when you do this because the system has default messages for all build in exceptions.

    throw [System.IO.FileNotFoundException]::new()
    throw [System.IO.FileNotFoundException]::new("Could not find path: $path")

If you are not yet using PowerShell 5.0, you will have to use the older `New-Object` approach.

    throw (New-Object -TypeName System.IO.FileNotFoundException )
    throw (New-Object -TypeName System.IO.FileNotFoundException -ArgumentList "Could not find path: $path")

By using a typed exception, you (or others) can catch the exception like me mentioned in the previous section.

### The big list of .Net exceptions

I compiled a master list that contains hundreds of .Net exceptions to complement this post. 

* [The big list of .Net exceptions](https://kevinmarquette.github.io/2017-04-07-all-dotnet-exception-list/?utm_source=blog&utm_medium=blog&utm_content=crosspost)

I start by searching that list for exceptions that feel like they would be a good fit for my situation. You should try to use exceptions in the base `System` namespace.

## Exceptions are objects

If you start using a lot of typed exceptions, remember that they are objects. Different exceptions have different constructors and properties. If we look at the [documentation](https://docs.microsoft.com/en-us/dotnet/api/System.IO.FileNotFoundException?view=netframework-4.7) for `System.IO.FileNotFoundException`, we will see that we can pass in a message and a file path.

    [System.IO.FileNotFoundException]::new("Could not find file", $path)

And it has a `FileName` property that exposes that file path.

    catch [System.IO.FileNotFoundException]
    {        
        Write-Output $PSItem.Exception.FileName
    }

You will have to consult the [.Net documentation](https://docs.microsoft.com/en-us/dotnet/api/index?view=netframework-4.7) for other constructors and object properties. 

## Re-throwing an exception

If all you are going to do in your `catch` block is `throw` the same exception, then don't `catch` it. You should only `catch` an exception that you plan to handle or perform some action when it happens.

But there are times where you want to perform an action on an exception but re-throw the exception so something downstream can deal with it. We could write a message or log the problem close to where we discover it but handle the issue further up the stack.

    catch
    {
        Write-Log $PSItem.ToString()
        throw $PSItem
    }

Interestingly enough, we can call `throw` from within the `catch` and it will re-throw the current exception.

    catch
    {
        Write-Log $PSItem.ToString()
        throw
    }

We want to re-throw the exception to preserve the original execution information like source script and line number. If we throw a new exception at this point it hide where the exception started.

### Re-throwing a new exception

If you catch an exception but you want to throw a different one, then you should nest the original exception inside the new one. This allows someone down the stack to access it as the `$PSItem.Exception.InnerException`.

    catch
    {
        throw [System.MissingFieldException]::new('Could not access field',$PSItem.Exception)
    }

## Write-Error with -ErrorAction Stop

I mentioned that Write-Error does not throw a terminating error by default. If you specify `-ErrorAction Stop` when calling a cmdlet or advanced function, then `Write-Error` generates a terminating error that is handled by `catch`.

### Use ToString() to get the message

It is important to note that Write-Error does not store the message in `$PSItem.Exception.Message` but instead places it in `$PSItem.ErrorDetails.Message`. The good news is that `$PSItem.ToString()` will correctly give you the right message either way.

# Closing remarks

Adding proper exception handling to your scripts will not only make them more stable but it will also make it easier for you to troubleshoot those exceptions.