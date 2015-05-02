# Hash - language for happy shell scripting

This project attempts to address the problem that many of us face when writing
shell scripts - dealing with the idiosyncratic behavior and syntax of Bash and
other shell scripting languages. The goal is to design a modern language that
could replace Bash for a whole range of scripts: from simple one-liners to
complex multi-module projects. Hash scripts will compile to Bash scripts for
ease of deployment and cross-platform compatibility. In general, the design
goals are:

  - Make shell scripting convenient, readable and safe
  - Ensure ease of deployment and cross-platform compatibility
  - Provide code-reuse facilities and support for 3rd-party libraries

These goals will be achieved through the following features:

  - Expressive syntax of the essential scripting constructs: pipes, output
    redirection, job control, conditionals, loops, etc.
  - Higher-level programming concepts: anonymous and higher-order functions,
    pattern matching, list and map comprehensions
  - Compile-time error checking
  - Compilation to Bash and/or other scripting languages
  - Built-in data structures: lists, maps
  - First-class support for command-line argument parsing and validation
  - Built-in mechanism for importing 3rd-party libraries, e.g., directly from
    Github
  - Batteries included - rich and versatile standard library

This document is a work in progress exploring the design of the language and its
feature set. The implementation has not started yet and the timeline is not
defined. The current focus is the design of the language, which will lead to a
specification that will guide implementation. We are open to all ideas and
contributions. Let's together create the dream shell scripting language!


# Language Features

## Variables

Variable types are determined dynamically and do not need to be declared.
Mutable variables are declared using the `var` keyword:

```Bash
var a = 1
a = 2  # the value of a is now 2
var b  # a variable can be declared without assigning a value
b = 1  # the value of b is now 1
c = 1  # compile-time error: use of an undeclared variable
```


## Constants

Constants (values) are defined with the `val` keyword:

```Bash
val a = 1
a = 2  # compile-time error: constant cannot be mutated
val b  # compile-time error: constant must be assigned a value when declared
```


## Comments

As seen above, single-line comments start with `#`:

```Bash
# This is a single-line comment
```

Multi-line comments are everything between `/*` and `*/`:

```Bash
/* This is
   a multi-line
   comment
*/
```


## Primitive Data Types

### Booleans

Boolean literals `true` and `false`, as well as the expected logical operators
(i.e., `and`, `or`) are supported. A boolean can be negated by prepending `not`:

```Bash
not false == true
true or false == true
true and false == false
```

### Numbers

Integer (e.g., `42`) and float (e.g., `3.14`) number literals are supported.


### Strings

String literals are surrounded with single or double quotes, where single-quoted
strings do not have variable nor expression interpolation, while double-quoted
strings support both (discussed in detail later):

```Bash
'plain string without variable or expression interpolation'
"string with $variable and expression (1 == 1) interpolation"
"the following symbols need to be escaped: \", \\, \$, \@, \(, \)"
```

String literals can also span multiple lines when surrounded with triple quotes:

```Bash
'''
multiline string literal
without interpolation
'''

"""
multiline string literal
with interpolation
"""
```


### File Names and Command-Line Options

Literals containing `/`, `-`, and `~` or starting with `.` are considered file
names / paths. Literals starting with `-` are considered command-line
options.[a] Both can be entered without the enclosing quotes:

```Bash
val home = /home/root
val lsOptions = -alh
```


## Data Structures

### Lists

List is a built-in data structure representing a sequence of elements. It's
defined using square brackets, where commas between elements are optional:

```Bash
[]                 # empty list
[1 2 3]            # list of 3 elements
1 :: 2 :: 3 :: []  # same as above defined using the cons operator (::)
[1 'abc' [1 2 3]]  # list elements can be heterogeneous, including other lists
```

Lists can also be defined using the range syntax:

```Bash
[0..5]     # yields [0 1 2 3 4 5]
[5..0]     # yields [5 4 3 2 1 0]
[0 2..10]  # yields [0 2 4 6 8 10]
```

List elements can be accessed by index:

```Bash
val a = [0..5]
a[2]  # yields 3
```

Lists can be sliced:

```Bash
var a = [0 2..10]
a[0:2]  # yields [0 2 4]
```

Similarly to Python, `a[i:j]` is a slice of `a` from `i` to `j`, `a[i:j:k]` is a
slice of `a` from `i` to `j` with step `k`.

```Bash
val a = [5..10]
a[0:4:2]  # yields [5 7 9]
```

List comprehensions are also supported:

```Bash
[[i j] for i <- [1 2] j <- [1..4]]
# yields [[1 1] [1 2] [1 3] [1 4] [2 1] [2 2] [2 3] [2 4]]

[[i j] for i <- [1 2 3] j <- [3 1 4] where i != j]
# yields [[1 3] [1 4] [2 3] [2 1] [2 4] [3 1] [3 4]]
```


### Maps

Maps, or dictionaries, are collections of key-value pairs. Similarly to Clojure,
commas are considered whitespace and can be used to organize pairs in maps::

```Bash
{}                     # empty map
{'a' 1, 'b' 2, 'c' 3}  # map of 3 key-value pairs
{1 1, 'a' "value"}     # keys and values can be of heterogeneous types
```

Map values are accessed by keys:

```Bash
val a = {'first' 1, 'second' 2}
a['first']  # yields 1
```

Since commas are optional, they can be omitted, which is convenient in
multi-line map definitions:

```Bash
val a = {
    'a' 1
    'b' 2
    'c' 3
}
a['c']  # yields 3
```

Map comprehensions are also supported:

```Bash
{i j for i <- [1 2] j <- [1..4]}
# yields {1 1, 1 2, 1 3, 1 4, 2 1, 2 2, 2 3, 2 4}
```


## Control Flow

### If Conditions

`if` conditions have a multi-line syntax:

```Bash
if a < b
    a
else
    b
```

As well as an inline syntax:

```Bash
if a < b then a else b
```

Conditionals are also expressions, which means their result can be assigned to a
variable:

```Bash
val min = if a < b then a else b
```


### Switch

Multiple if-else clauses can be more conveniently expressed with `switch`, which
also supports pattern matching (discussed later):

```Bash
val key = 'bar'
switch key
    case 'foo'
        echo "Found foo"
    case 'bar'
        echo "Found bar"
    case []
        echo "Found an empty list"
    case x::xs
        echo "Found a list with the head of $x and tail of $xs"
    case x::_
        echo "Found a list with the head of $x, don't care about the tail"
    case ['foo' x]
        echo "Found a pair of foo and $x"
    case [a b]
        echo "Found a pair of $a and $b"
    case _
        echo "Anything else"
```


### For Loops

Print out numbers from 0 to 10:

```Bash
for i in [0..10]
    echo i
```


### While Loops

Print out numbers from 0 to 10:

```Bash
var i = 0
while i <= 10
    echo i
    i++
```


## Functions

### Named Functions

Named functions are defined as follows:

```Bash
fn factorial n =
    if n < 1 then 1
    else n * factorial (n - 1)
```

Where `factorial` is the name of the function, `n` is the argument, and
everything following `=` is the function body. The value of the last expression
automatically becomes the function return value. In other words, having the
`return` keyword is optional; however, it is useful when it's necessary to
return a value earlier.

Function application has a syntax similar to Bash and Haskell, it's the function
name followed by a space-separated list of arguments. For example, `factorial
10` would return the results of computing the factorial of 10. This style
requires parentheses around an expression, whose result is passed as an
argument. Therefore, to compute the factorial of `n - 1`, the function is called
as `factorial (n - 1)`.


### Anonymous Functions

Anonymous functions (lambdas) are defined using the arrow symbol `->` separating
the list of arguments and function body:

```Bash
map (x -> x * 2) [1 2 3]  # yields [2 4 6]
```

There is a shorter Scala-like syntax for lambdas with an implicit argument list:

```Bash
map (_ * 2) [1 2 3]  # yields [2 4 6]
```

Lambdas can have multiple arguments:

```Bash
fold (a b -> if a >= b then a else b) 0 [1 2 3]  # yields 3
```

The shorter syntax is supported for multi-argument lambdas:

```Bash
reduce (_ * _) [1 2 3]  # yields 6
```

A no-argument anonymous function is defined by just omitting the argument list:

```Bash
map (-> 42) [1 2 3]  # yields [42 42 42]
```

This can, for example, be used to implement a `sudo` function that lifts some
code into the superuser privileges:

```Bash
sudo (->
    doSomethingImportant1
    doSomethingImportant2
    doSomethingImportant3
)
```


### Pattern Matching

Pattern-matching is done similarly to Haskell, except that cases are defined
with the `case` keyword:

```Bash
fn factorial
    case 0 = 1
    case n = n * factorial (n - 1)
```

Arguments can be ignored with `_`; destructuring is also supported:

```Bash
fn map
    case _ [] = []
    case f (x::xs) = f x :: map f xs
```

Pattern-matching can be done across a variable number of arguments:

```Bash
fn length
    case xs = length 0 xs
    case n [] = n
    case n (x::xs) = length (n + 1) xs
```


## File Name Pattern Expansion

File name pattern expansion works similarly to Bash, except that the pattern
must be surrounded with `-`:

```Bash
ls -*.sh-  # equivalent to 'ls *.sh' in Bash
```


## External Programs

Hash tries to blur the distinction between functions and external programs in
$PATH, where a function overrides the external program with the same name.

```Bash
echo "test"  # outputs "test" to stdout using the external 'echo' program
```

File name and option literals can be used here as arguments making it look
indistinguishable from Bash:

```Bash
find / -name "needle.sh"
```

The exit status can be accessed using the special `status` variable, which gets
automatically populated based on the previous execution:

```Bash
echo status
```

An external program can be made to return its exit status by appending `?` to
its name:

```Bash
if find? / -name "needle.sh"
    echo "Found needle.sh"
```

A call to an external program can be transformed into an anonymous function by
enclosing it in backticks:

```Bash
val ls = `ls -al`  # ls is a constant containing a lambda
ls -*.sh-  # executes 'ls -al *.sh'
```


### Pipes

Pipes work similarly to Bash making pipelines as the following feel natural:

```Bash
find . | grep "needle"
```

Pipelines can also include functions, where the output gets automatically
transformed into a list by splitting the content by newlines:

```Bash
map (length _) (find . | grep "needle")  # outputs the number of matching files
```

This example can also be simplified by passing the `length` function directly as
an argument instead of wrapping it in an anonymous function:

```Bash
map length (find . | grep "needle")
```

If a value is piped into a function, it becomes its last argument:

```Bash
[1 2 3] | filter (_ > 1) | reverse  # yields [3 2]
```


### Output Redirection

Output redirection works similarly to Bash, except that instead of the numbered
streams (i.e., 1> and 2>), more descriptive names are used (i.e., out>, err>,
all>):

```Bash
grep "needle" -*- out> ./stdout.log err> ./stderr.log
grep "needle" -*- out>> ./stdout.log  # append stdout to ./stdout.log
grep "needle" -*- err>&out  # redirect stderr to stdout
grep "needle" -*- all> /dev/null  # both stdout and stderr can be redirected with all>
grep "needle" -*- all>_  # redirecting to /dev/null can be done by appending _
'input' in> sed 's/input/output/'  # here-string, directing a string to stdin of sed
```


## Command-Line Arguments

Parsing command-line arguments and options by scripts is a very common task;
therefore, Hash provides special support and syntax for that. Command-line
arguments and options are declared and referred to with the `@` prefix.

First of all, the description string of the script is defined by calling the
`description` function in the beginning of the script:

```Bash
description('This is a demo of the command-line argument support')
```

Let's say there is one required and one optional positional arguments, in Hash
they can be declared as follows:

```Bash
@1: 'Description of the first required positional argument'
@2: 'Description of the second optional positional argument' optional

echo "Received arguments: @1 and @2"
```

Based on the provided specification, a help message (when called with `-h` or
`--help`), validation logic, and error reporting code are automatically
generated. Positional arguments can also be given descriptive names:

```Bash
@1 as @foo: 'Description of the first required positional argument'
@2 as @bar: 'Description of the second optional positional argument' optional

echo "Received arguments: @1 and @2"
```

Named command-line options can also be defined:

```Bash
@foo: 'The --foo option name is automatically provided'
@bar: 'Multiple options names can be specified manually' [-b --bar]
@baz: 'A required type can be specified to be automatically validated' int
@qux: 'A custom validation function can be specified' validate=myValidationFunction
```

In addition, argument validation rules can be defined declaratively using a
built-in DSL:

```Bash
@foo one of ['one' 'two' 'three']  # must be one of the enumerated values
@foo and @bar  # @foo and @bar must both be present or absent
@bar xor @baz  # only @bar or @baz can be present, not both
(@baz > 0 and @qux) or (@baz < 0 and not @qux)  # value-based validation
```

The complete array of arguments is available in `@args`.


## Libraries (Hashlets)

Hash allows importing functions from external files and git repositories with
special support for Github. These external libraries of functions are called
'hashlets'. Each hashlet acts as a namespace for functions. Here is how a
hashlet can be imported from a local file `stringutils.hash` by specifying its
path relatively to the location of the current script:

```Bash
import ./stringutils

# or just
import stringutils
```

Then, functions from the `stringutils` hashlet can be called by prefixing their
names with the hashlet name and a dot:

```
stringutils.split 'foo bar'  # yields ['foo' 'bar']
```

When imported, hashlets can be given aliases:

```Bash
import stringutils as str
str.split "foo bar"  # yields ['foo' 'bar']
```

Specific functions can be imported directly into the current namespace by
listing them after the hashlet name in square brackets:

```Bash
import stringutils [split toUpper]
map toUpper (split "foo bar")  # yields ['Foo' 'Bar']
```

Hashlets can be imported from subdirectories and any other local path relatively
to the location of the current script:

```Bash
# import functions copy and join from ./subdir/fileutils.hash
import subdir/fileutils [copy join]
```

Hashlets can be imported directly from Github repositories from a specific
commit. Specifying a commit hash is mandatory to avoid breaking the code when
the remote branch gets updated. For example, to import functions `strpos` and
`split` from `github.com/hash-lang/strings` at `459c616`:

```Bash
import hash-lang/strings 459c616 [strpos split]
```

In addition to Github, any git repository is supported:

```Bash
import git://... 459c616 as mylib
```

The compiler will then first fetch the remote repositories into into the
`.hashlets` directory and use the function definitions from them. Only functions
and constants are imported from hashlets, all the other statements get ignored
during the import. The compiler will only fetch the dependencies once and use
the cached files for future executions.

If any of the imported functions use `sudo` internally, by default the compiler
will issue a warning, as that might be a security concern when using 3rd-party
code. To suppress the warning, `sudo` can be explicitly allowed by adding the
`allow sudo` keywords to the import statement:

```Bash
import 3rdparty/lib 459c616 allow sudo
```


# Translating Bash into Hash

In this section, we are going to rewrite all the code samples from "BASH
Programming - Introduction HOW-TO"
(http://tldp.org/HOWTO/Bash-Prog-Intro-HOWTO.html) in Hash. The main goal is to
show how Bash can be directly translated into Hash. However, sometimes,
alternative (more idiomatic) versions of Hash snippets implementing the same
functionality will also be shown.


## Traditional hello world script

Bash

```Bash
#!/bin/bash
echo Hello World
```

Hash

```Bash
#!/bin/hash
echo "Hello World"
```


## A very simple backup script

Bash

```Bash
tar -cZf /var/my-backup.tgz /home/me/
```

Hash

```Bash
tar -cZf /var/my-backup.tgz /home/me/
```


## stdout 2 file

Bash

```Bash
ls -l > ls-l.txt
```

Hash

```Bash
ls -l out> ls-l.txt
```


## stderr 2 file

Bash

```Bash
grep da * 2> grep-errors.txt
```

Hash

```Bash
grep da -*- err> grep-errors.txt
```


## stdout 2 stderr

Bash

```Bash
grep da * 1>&2
```

Hash

```Bash
grep da -*- out>&err
```


## stderr 2 stdout

Bash

```Bash
grep * 2>&1
```

Hash

```Bash
grep -*- err>&out
```


## stderr and stdout 2 file

Bash

```Bash
rm -f $(find / -name core) &> /dev/null
```

Hash

```Bash
rm -f (find / -name "core") all> /dev/null
```

Hash (alternative using `_` to ignore the output target)

```Bash
rm -f (find / -name "core") all>_
```


## Simple pipe with sed

Bash

```Bash
ls -l | sed -e "s/[aeio]/u/g"
```

Hash

```Bash
ls -l | sed -e "s/[aeio]/u/g"
```


## An alternative to ls -l *.txt

Bash

```Bash
ls -l | grep "\.txt$"
```

Hash

```Bash
ls -l | grep "\.txt$"
```


## Hello World! using variables

Bash

```Bash
STR="Hello World!"
echo $STR
```

Hash

```Bash
var str = "Hello World!"
echo str
```


## A very simple backup script (little bit better)

Bash

```Bash
OF=/var/my-backup-$(date +%Y%m%d).tgz
tar -cZf $OF /home/me/
```

Hash

```Bash
val of = /var/my-backup-(date '+%Y%m%d').tgz
tar -cZf of /home/me/
```


## Local variables

Bash

```Bash
HELLO=Hello
function hello {
    local HELLO=World
    echo $HELLO
}
echo $HELLO  # outputs: Hello
hello        # outputs: World
echo $HELLO  # outputs: Hello
```

Hash

```Bash
val hello = "Hello"
fn hello =
    val hello = "World"
    echo hello
echo hello  # outputs: Hello
hello       # outputs: World
echo hello  # outputs: Hello
```


## Basic conditional example if .. then

Bash

```Bash
if [ "foo" = "foo" ]; then
    echo expression evaluated as true
fi
```

Hash

```Bash
if "foo" == "foo"
    echo "expression evaluated as true"
```


## Basic conditional example if .. then ... else

Bash

```Bash
if [ "foo" = "foo" ]; then
    echo expression evaluated as true
else
    echo expression evaluated as false
fi
```

Hash

```Bash
if "foo" == "foo"
    echo "expression evaluated as true"
else
    echo "expression evaluated as false"
```


## Conditionals with variables

Bash

```Bash
T1="foo"
T2="bar"
if [ "$T1" = "$T2" ]; then
    echo expression evaluated as true
else
    echo expression evaluated as false
fi
```

Hash

```Bash
val t1 = "foo"
val t2 = "bar"
if t1 == t2
    echo "expression evaluated as true"
else
    echo "expression evaluated as false"
```


## For

Bash

```Bash
for i in $( ls ); do
    echo item: $i
done
```

Hash

```Bash
for i in (ls)
    echo "item: $i"
```

Hash (alternative using map)

```Bash
map (ls) (echo "item: $_")

# or just
map (ls) echo
```


## C-like for

Bash

```Bash
for i in `seq 1 10`;
do
    echo $i
done
```

Hash

```Bash
for i in [1..10]
    echo i
```

Hash (alternative using map)

```Bash
map [1..10] echo
```


## While

Bash

```Bash
COUNTER=0
while [ $COUNTER -lt 10 ]; do
    echo The counter is $COUNTER
    let COUNTER=COUNTER+1
done
```

Hash

```Bash
var counter = 0
while counter < 10
    echo "The counter is (counter)"
    counter++
```


## Until

Bash

```Bash
COUNTER=20
until [ $COUNTER -lt 10 ]; do
    echo COUNTER $COUNTER
    let COUNTER-=1
done
```

Hash

```Bash
var counter = 20
while counter >= 10
    echo "COUNTER (counter)"
    counter++
```


## Functions

Bash

```Bash
function quit {
    exit
}
function hello {
    echo Hello!
}
hello
quit
echo foo
```

Hash

```Bash
fn quit = exit
fn hello = echo "Hello!"
hello
quit
echo "foo"
```


## Functions with parameters

Bash

```Bash
function quit {
    exit
}
function e {
    echo $1
}
e Hello
e World
quit
echo foo
```

Hash

```Bash
fn quit = exit
fn e message = echo message
e "Hello"
e "World"
quit
e "foo"
```


## Using select to make simple menus

Bash

```Bash
OPTIONS="Hello Quit"
select opt in $OPTIONS; do
    if [ "$opt" = "Quit" ]; then
        echo done
        exit
    elif [ "$opt" = "Hello" ]; then
        echo Hello World
    else
        clear
        echo bad option
    fi
done
```

Hash

```Bash
val options = "Hello Quit"
while true
    val opt = select options
    if opt == "Quit"
        echo "done"
        exit
    else if opt == "Hello"
        echo "Hello World"
    else
        clear
        echo "bad option"
```

Hash (alternative using switch)

```Bash
while true
    switch (select "Hello Quit")
        case "Quit"
            echo "done"
            exit
        case "Hello"
            echo "Hello World"
        case _
            clear
            echo "bad option"
```


## Using the command-line

Bash

```Bash
if [ -z "$1" ]; then
    echo usage: $0 directory
    exit
fi
SRCD=$1
TGTD="/var/backups/"
OF=home-$(date +%Y%m%d).tgz
tar -cZf $TGTD$OF $SRCD
```

Hash

```Bash
@1 as @srcd: "The source directory"
val tgtd = /var/backups/
val of = home-(date '+%Y%m%d').tgz
tar -cZf "$tgtd$of" @srcd
```


## Reading user input with read

Bash

```Bash
echo Please, enter your name
read NAME
echo "Hi $NAME!"
```

Hash

```Bash
echo "Please, enter your name"
val name = read
echo "Hi $name"
```


## Reading user input with read (multiple values)

Bash

```Bash
echo Please, enter your firstname and lastname
read FN LN
echo "Hi! $LN, $FN !"
```

Hash

```Bash
echo "Please, enter your firstname and lastname"
val [firstName lastName] = read  # using destructuring
echo "Hi! $lastName, $firstName !"
```


## Arithmetic evaluation

Bash

```Bash
echo 1 + 1
echo $((1 + 1))
echo $[1+1]
echo 3/4|bc -l
```

Hash

```Bash
echo "1 + 1"
echo (1 + 1)
echo (1 + 1)
echo (3 / 4)
```


## Getting the return value of a program

Bash

```Bash
cd /dada &> /dev/null
echo rv: $?
cd $(pwd) &> /dev/null
echo rv: $?
```

Hash

```Bash
cd /data all>_
echo "rv: $status"
cd (pwd) all>_
echo "rv: $status"
```

Hash (alternative using ?)

```Bash
echo "rv: (cd? /data all>_)"
echo "rv: (cd? (pwd) all>_)"
```


## Capturing a commands output

Bash

```Bash
DBS=`mysql -uroot -e"show databases"`
for b in $DBS ;
do
    mysql -uroot -e"show tables from $b"
done
```

Hash

```Bash
val dbs = mysql -uroot -e"show databases"
for b in dbs
    mysql -uroot -e"show tables from $b"
```


## String comparison operators

Bash

```Bash
s1 = s2
s1 != s2
s1 < s2
s1 > s2
-n s1
-z s1
```

Hash

```Bash
s1 == s2
s1 != s2
s1 < s2
s1 > s2
not empty s1
empty s1
```


## String comparison examples

Bash

```Bash
S1='string'
S2='String'
if [ $S1=$S2 ];
then
    echo "S1('$S1') is not equal to S2('$S2')"
fi
if [ $S1=$S1 ];
then
    echo "S1('$S1') is equal to S1('$S1')"
fi
```

Hash

```Bash
val s1 = 'string'
val s2 = 'String'
if s1 == s2
    echo "S1\('$s1'\) is not equal to S2\('$s2'\)"
if s1 == s1
    echo "S1\('$s1'\) is not equal to S1\('$s1'\)"
```


## Arithmetic operators

Bash

```Bash
+
-
*
/
%
```

Hash

```Bash
+
-
*
/
%
```


## Arithmetic relational operators

Bash

```Bash
-lt
-gt
-le
-ge
-eq
-ne
```

Hash

```Bash
<
>
<=
>=
==
!=
```


## File re-namer

Bash

```Bash
if [ $1 = p ]; then
    prefix=$2 ; shift ; shift
    if [$1 = ]; then
        echo "no files given"
        exit 0
    fi

    for file in $*
    do
        mv ${file} $prefix$file
    done

    exit 0
fi
if [ $1 = s ]; then
    suffix=$2 ; shift ; shift

    if [$1 = ]; then
        echo "no files given"
        exit 0
    fi

    for file in $*
    do
        mv ${file} $file$suffix
    done

    exit 0
fi

if [ $1 = r ]; then
    shift

    if [ $# -lt 3 ] ; then
        echo "usage: renna r [expression] [replacement] files... "
        exit 0
    fi

    OLD=$1 ; NEW=$2 ; shift ; shift

    for file in $*
    do
        new=`echo ${file} | sed s/${OLD}/${NEW}/g`
        mv ${file} $new
    done
    exit 0
fi

echo "usage;"
echo " renna p [prefix] files.."
echo " renna s [suffix] files.."
echo " renna r [expression] [replacement] files.."
exit 0
```

Hash

```Bash
@1 as @mode: 'Mode of matching'
@2 as @prefix @suffix @old: 'Prefix, suffix, or old depending on the mode'
@3 as @repl: 'The replacement' optional

@mode oneof ['p' 's' 'r']
(@mode == 'r' and @repl) or @mode

if @mode == 'p'
    for file in @args[2:]
        mv file "@prefix$file"
else if @mode == 's'
    for file in @args[2:]
        mv file "$file@suffix"
else
    for file in @args[3:]
        val new = echo file | sed "s/@old/@repl/g"
        mv file new
```

Hash (alternative using switch)

```Bash
@1 as @mode: 'Mode of matching'
@2 as @prefix @suffix @old: 'Prefix, suffix, or old depending on the mode'
@3 as @repl: 'The replacement' optional

@mode oneof ['p' 's' 'r']
(@mode == 'r' and @repl) or @mode

switch @mode
    case 'p'
        for file in @args[2:]
            mv file "@prefix$file"
    case 's'
        for file in @args[2:]
            mv file "$file@suffix"
    case _
        for file in @args[3:]
            mv file (replace file @old @repl)
```


## File renamer (simple)

Bash

```Bash
criteria=$1
re_match=$2
replace=$3

for i in $( ls *$criteria* );
do
    src=$i
    tgt=$(echo $i | sed -e "s/$re_match/$replace/")
    mv $src $tgt
done
```

Hash

```Bash
@1 as @criteria: 'File pattern'
@2 as @rematch: 'Regex matcher'
@3 as @replace: 'Replacement'

for src in (ls -*@criteria*-)
    mv src (src | sed "s/@rematch/@replace/")
```
