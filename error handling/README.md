#Error Handling

Learning how to diagnose, interpret, and overcome errors constitutes a major part of learning to program in any language.

##Three Kinds of Error

There are essentially three kinds of error in XQuery: static errors, dynamic errors, and type errors. The [XQuery Recommendation](http://www.w3.org/TR/2014/REC-xquery-30-20140408/#id-kinds-of-errors) defines and discusses the differences between these errors.

* Static Errors

>[Definition: An error that can be detected during the static analysis phase, and is not a type error, is a static error.] A syntax error is an example of a static error.

Static errors are the most common form of mistakes, especially when you are beginning to program. If you make a syntax mistake, such as using ```=``` for assignment rather than ```:=``` or forgetting to include a matching parenthesis/bracket, you have committed a static error. This is a static error because an XQuery interpreter will catch it before your program even runs (technically, it gets caught during the static analysis phase).  

* Dynamics Errors

>[Definition: A dynamic error is an error that must be detected during the dynamic evaluation phase and may be detected during the static analysis phase.] Numeric overflow is an example of a dynamic error.

A dynamic error, by contrast, may not be caught before your program runs. You may have gotten the syntax right but still have gone awry with your program logic. For example, you may have written a recursive function that does not specify properly a base case. If so, the function will continue to call itself until it runs out of space on the stack, resulting in a [stack overflow](http://en.wikipedia.org/wiki/Stack_overflow). This error may not be detectable until the program executes (technically, until the dynamic evaluation phase). This is the kind of "bug" that crashes your program after it begins running. In general, these kinds of "bugs" are harder to diagnose and resolve.

* Type Errors

>[Definition: A type error may be raised during the static analysis phase or the dynamic evaluation phase. During the static analysis phase, a type error occurs when the static type of an expression does not match the expected type of the context in which the expression occurs. During the dynamic evaluation phase, a type error occurs when the dynamic type of a value does not match the expected type of the context in which the value occurs.]

A common source of errors arises from type mismatches. We've already seen how to write functions that check the types of their inputs and outputs. Sending an argument of the wrong type to a function results in type error. A good example of this mistake is when your program produces a "number" that is actually a xs:string and you send that "number" to a function requiring an xs:integer as input. In order to avoid such an error, you will need first to cast the string to an integer first.

For example, this expression checks if a variable can be cast as an xs:integer and, if not, throws an error.

```xquery
let $num := "1"
let $int := if ($num castable as xs:integer) then xs:integer($num) else fn:error()
return $int
```

[Try it!](http://try.zorba.io/queries/xquery/meFXr1HHu%2BGBe7O3l1aj40GxStk%3D)

This style of defensive programing helps to surface type mismatches which might otherwise produce hard-to-diagnose bugs in your code.

##Reading Errors

Reading error messages takes a little practice.When you encounter an error, you will receive what many perceive as a cryptic message identifying the kind of error along with an indication of where that error occurred in your code.

For example, evaluating this expression

```xquery
xquery version "3.0";

3 * "2"
```

produces the following error

```
[XPTY0004] arithmetic operation not defined between types "xs:integer" and "xs:string"

Line 3, column 1.
```

[Try it!](http://try.zorba.io/queries/xquery/FOo3dt8sY4W01pSaqW6HcWo545A%3D)

Let's see if we can understand what the interpreter is telling us about what went wrong.

First, notice the code inside the square brackets. The code identifies the error according to the following formula from the [XQuery recommendation](http://www.w3.org/TR/2014/REC-xquery-30-20140408/#id-identifying-errors):

>* XX denotes the language in which the error is defined, using the following encoding:
	* XP denotes an error defined by XPath. Such an error may also occur XQuery since XQuery includes XPath as a subset.
	* XQ denotes an error defined by XQuery (or an error originally defined by XQuery and later added to XPath).
> * YY denotes the error category, using the following encoding:
	* ST denotes a static error.
	* DY denotes a dynamic error.
	* TY denotes a type error.
* nnnn is a unique numeric code.

*N.B. You will also see FO (functions and operators) and SE (serialization errors) as two digit language codes*

Using this information, we can decode some of the error identifier. The first two characters ```XP``` indicate that we have incurred an error in XPath. The second two characters ```TY``` tells us that we have caused a type error. But what about our four digit code ````0004````?

Here we need to read another W3C Document called the [XQuery and XPath Functions and Operators Error Codes Namespace Document](http://www.w3.org/2005/xqt-errors/). If we go to that document, we can look up the identifier of our error. It turns out that [XPTY0004](http://www.w3.org/TR/xpath-30/#ERRXPTY0004) is a pretty generic type mismatch error. 

The implementation tells us a little more about why this error occurred. We see that implementation tells us that we cannot add an xs:integer and an xs:string together. 

The implementation also provides a line and column number for the error. This may not always accurately reflect where we think the error needs to be remedied. For example, in this case, the implementation identifies column 1 or ```3``` as the source of the problem. This is, of course, the xs:integer we tried to add to an xs:string. To fix this problem, we'd probably want to change the xs:string in column 6 to an xs:integer.

##Defensive Programming

A leading idea of [defensive programming](http://en.wikipedia.org/wiki/Defensive_programming) is to make as few implicit assumptions about your code and data as practically possible. For instance, it is better not to assume that the input to a function will always have the expected type. Rather, try to make your assumptions explicit by checking the types of your function arguments and return values. Also, write code to test for and handle common error conditions.

compare the difference between

```xquery
xquery version "3.0";

declare function local:divide($num1, $num2)
{
  $num1 div $num2
};

local:divide(2,"a")
```

and

```xquery
xquery version "3.0";

declare function local:divide($num1 as xs:double, $num2 as xs:double) as xs:double
{
  if ($num2 != 0) then $num1 div $num2
  else fn:error(xs:QName("FOAR0001"), "Hey, you're trying to divide by zero!")
};

local:divide(2,"a")
```

[Try it!](http://try.zorba.io/queries/xquery/fCwpDjLwAaQgxal0auMkoeDR4tY%3D)

In the first example, we do not check the type of our arguments and thus allow problematic arguments (like xs:string types) be passed in to our function. We also do not check for common error conditions such as division by zero. The second example handles error conditions more robustly by guarding against inappropriate function arguments and gracefully handling division by zero errors.

##Try/Catch Expressions

XQuery 3.0 introduces a [try/catch expression](http://www.w3.org/TR/xquery-30/#id-try-catch) to assist with handling errors. A try/catch expression can be used to intercept dynamic errors and type errors that might otherwise bring your program to a sudden halt.

*N.B. Try/Catch expressions do not handle static errors, which get identified prior to the dynamic evaluation phase.*

Here's an example of how a try/catch expression works in practice:

```xquery
xquery version "3.0";

try {
    3 * "2"
}
catch * {
    "Something went wrong!"
}
```

[Try it!](http://try.zorba.io/queries/xquery/6ekWWgACzClLrdXCf5gyv%2FBiirs%3D)

The ```*``` after the catch keyword indicates that we'd like to catch all errors. We could specify the kind(s) of error(s) by substituting a corresponding error identifier(s) for ```*```. We might do this if we want to catch some errors, but leave others unhandled. For example,

```xquery
xquery version "3.0";

try {
    3 div 0 
}
catch XPTY0004 {
    "Something went wrong!"
}
```

This try/catch expression only handles type errors. If we attempt to 3 by zero, we'll get a ```[FOAR0001] division by zero``` error, which won't be caught and which will halt our program.

Deciding when to use try/catch expressions can be tricky. Obviously, try/catch expressions can be useful when you don't want your program to crash in the event of an error. For example, you might include a try/catch expression in your controller when writing a web application to handle unexpected errors in a graceful way rather than displaying cryptic error messages to your user. On the other hand, try/catch expressions can themselves become sources of "bugs," especially if you handle errors in logic which should have been fixed prior to the execution of your program.

The art of using try/catch expressions is [much debated](http://programmers.stackexchange.com/questions/64180/good-use-of-try-catch-blocks). A good rule of thumb is to use try/catch expressions when dealing with an external environment. For example, if you write an expression to get a string from a user and open a file on the file system with a name correspond to that string, you should probably enclose that expression in a try/catch expression to handle potential I/O errors gracefully.

##XUnit Annotations

##XQLint

##XQDoc

[XQDoc](http://xqdoc.org/) creates XQuery documentation from comments in your source file. While documentation is not strictly speaking a matter of error handling, good documentation can frequently prevent inadvertent errors. Good documentation also promotes code reuse. By commenting your code well and by producing readable documentation, you may be able to help others avoid making mistakes when using their code.

Let's consider an example of undocumented code:

```xquery
xquery version "1.0";

declare function local:pow($number as xs:integer, $power as xs:integer) as xs:integer {
  if ($power = 0) then 1
  else $number * local:pow($number, $power - 1)
};

local:pow(2,3)
```

To understand this function, we need to read through the code. Of course, it is easy enough to read through this small function. But things get complicated with larger functions.

We're also missing information about its author. We don't know that this function was adapted from an example provided by Michael Kay on [Stackoverflow](http://stackoverflow.com/a/15369640).

Let's look at the same example with XQDoc comments:

```xquery
xquery version "3.0";
 
(:~ The purpose of this main module is to demontrate the use of a recursive function.
:   @author Clifford Anderson
:   @version 1.0
:)
 
(:~ This function raises a given integer by a given power. 
:   @author Clifford Anderson
    @version 1.0
    @see http://stackoverflow.com/a/15369640;;Adapted from Michael Kay's example on Stackoverflow.
    @param $number An integer to be raised by a power
    @param $power An integer indicating the power to be raised
    @return An integer
:)
declare function local:pow($number as xs:integer, $power as xs:integer) as xs:integer {
  if ($power = 0) then 1
  else $number * local:pow($number, $power - 1)
};
 
local:pow(2,3)
```

##Conclusion