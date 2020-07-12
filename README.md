# Clash
Portable shell object-oriented programming

# Index

* [What](#What)
  * [Features](#Features)
  * [Compatibility](#Compatibility)
  * [How to use](#How-to-use)
    * [Lib](#Lib)
* [Why](#Why)
* [How](#How)
* [Q/A](#QA)
  * [What's the point when python, perl and other more robust scripting languages exist?](#Whats-the-point-when-python-perl-and-other-more-robust-scripting-languages-exist)
  * [What's the performance of clash? Show me the numbers!](#Whats-the-performance-of-clash-Show-me-the-numbers)
  * [Why not generate variables instead of functions?](#Why-not-generate-variables-instead-of-functions)
  * [I don't see `sh` in the compatibility table, does this support the Bourne shell?](#I-dont-see-sh-in-the-compatibility-table-does-this-support-the-Bourne-shell)
* [Notes](#Notes)

# What
Define classes and instantiate objects in shell with the usual shell functions
syntax.

Compatible with most Bourne like shells like bash/dash/ksh/osh/zsh

Basic demo of how **clash** works [here](https://lhoursquentin.github.io/clash-web-demo/)

## Features

What you would expect from basic object-oriented programming is supported:

- classes
- objects
- attributes
- methods
- constructor
- destructor
- generated getters
- generated setters

Not supported:

- inheritance

## Compatibility

Most modern shell builtin high level constructs such as local variables, arrays,
dictionaries and such are not defined by
[POSIX](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/contents.html)
and usually differ in behavior and syntax between different shells.

Portability is at the core of this project, there is a minimal use of non
POSIX features, allowing most of the code to be highly portable between many
different shells.

| Shell                                                     | Version(s) tested         | Support              |
| --------------------------------------------------------- | ------------------------- | -------------------- |
| [bash](https://www.gnu.org/software/bash)                 | 3.0 / 4.4 / 5.0           | Full                 |
| [busybox](https://busybox.net)                            | 1.28 (ash w/ bash compat) | Full                 |
| [dash](https://en.wikipedia.org/wiki/Almquist_shell#dash) | 0.5.10.2                  | Full                 |
| [ksh](https://en.wikipedia.org/wiki/KornShell)            | AT&T 93u+ (2012)          | clash 100%, lib 50%  |
| [mksh](http://www.mirbsd.org/mksh.htm)                    | R57                       | Full                 |
| [osh](https://www.oilshell.org)                           | 0.6.0                     | Full                 |
| [yash](https://yash.osdn.jp)                              | 2.48                      | Full                 |
| [zsh](https://www.zsh.org/)                               | 5.6.2                     | Full                 |

## How to use
```bash
# source the class definition
. ./clash
```

```bash
# Create a class
class Car   \
  brand     \
  speed     \
  _start    \
  __init__  \

# Methods are defined externally and are registered without the class name
# (see _start above)
Car_start() {
  printf 'Starting %s: ' "$self"
  if [ "$speed" -gt 100 ]; then
    echo 'Vroooooooom'
  else
    echo 'Vroom'
  fi
}

# If declared, the constructor is called when creating the object
Car__init__() {
  echo "New $1 added to the garage"
  "$self"_brand_is "$1"
  "$self"_speed_is "$2"
}
```

```bash
# Create a new car object named 'cool_bolid', Car__init__ function is called
Car cool_bolid Ferrari 200

# Create another one, Car__init__ function is called
Car family_truck Toyota 90
```

```
New Ferrari added to the garage
New Toyota added to the garage
```

```bash
# Call the _start method
cool_bolid_start
family_truck_start
```

```
Starting cool_bolid: Vroooooooom
Starting family_truck: Vroom
```

```bash
# check the family truck speed (generated getter)
echo "Family truck speed: $(family_truck_speed)"

# Modify the speed (generated setters are suffixed by _is)
echo 'Upgrading truck engine...'
family_truck_speed_is 120

# Start it again
family_truck_start
```

```
Family truck speed: 90
Upgrading truck engine...
Starting family_truck: Vroooooooom
```

### Lib

Simple implementations of **List** and **Vector** have been done so far
(**Dict** in progress), they can be found in the `lib` directory.

Following-up with our example:

```bash
# Load the list module (containing the List class)
. ./lib/list
```

```bash
# create a list of my cars
List my_cars cool_bolid family_truck

# Create a new one
Car minivan Toyota 80

# And add it to the list
my_cars_append minivan

# Print the elements
printf 'My cars: '
my_cars_print
```

```
New Toyota added to the garage
My cars: cool_bolid family_truck minivan
```

```bash
# Let's say we want the list of our Toyota cars, let's create a helper function
is_toyota() {
  # $1 is the car object, return 0 if the brand is Toyota, else 1
  [ "$("$1"_brand)" = Toyota ]
}

# And use the List filter method with our newly created helper function
my_cars_filter my_toyota_cars is_toyota

# Check the result
printf 'My Toyota cars: '
my_toyota_cars_print
```

```
My Toyota cars: family_truck minivan
```

Note: this code can be found in the `examples` directory.

# Why

Gathering and organizing data can be painful and dirty in shell, especially when avoiding non-portable features like arrays/maps.
Object-oriented programming is one solution to those issues, but does not exist natively in shell.

The goal of this project to provide a simple solution to the missing object paradigm and to be as portable as possible (working with **dash**/**bash**/**ksh**/**zsh**/**busybox ash**), the only non POSIX feature used being `local` (more info in the **notes** section).

Most importantly the syntax to create and use classes and objects is the exact same as usual shell, and that is because  what clash does is only generate functions, almost no parsing is done, and the grammar isn't altered using aliases.

# How

Two functions are generated per attribute, one for the getter with the attribute value hardcoded, and the other for the setter which when called redefines the original getter function with a new hardcoded value.

Methods are just generated functions wrapping a local declaration of all the attributes and a call of the initially declared function so that it has access to all the attributes in its frame.

All these methods and attributes are generated using `eval` and playing with quotes (a lot of quotes) to preserve IFS characters.

# Q/A

## What's the point when python, perl and other more robust scripting languages exist?

This is something that comes up really often, I've found that [Oil
Shell](http://www.oilshell.org/) main FAQ covers this topic thoroughly:

- [I don't understand. Why not use a different a programming language?](http://www.oilshell.org/blog/2018/01/28.html#i-dont-understand-why-not-use-a-different-a-programming-language)
- [Shouldn't we discourage people from writing shell scripts?](http://www.oilshell.org/blog/2018/01/28.html#shouldnt-we-discourage-people-from-writing-shell-scripts)
- [Shouldn't scripts over 100 lines be rewritten in Python or Ruby?](http://www.oilshell.org/blog/2018/01/28.html#shouldnt-scripts-over-100-lines-be-rewritten-in-python-or-ruby)

Be sure to check out [Oil](http://www.oilshell.org/) in more detail if you can,
it's a very impressive project.

## What's the performance of clash? Show me the numbers!

Pretty bad! But this depends heavily on which shell you are using and the number
and size of the objects you are creating. Creating a dozen of objects with a few
methods and attributes won't be noticeable. However creating hundreds or
thousands of objects will lead to (very) slow execution and a high load.

Considering how **clash** works this is to be expected, for instance one
single method call needs to do as many command substitutions as the number of
attributes of the object:

```sh
truck_start() {
  # Populate all object attributes for the actual function call
  local self=truck
  local speed=$(truck_speed)
  local brand=$(truck_brand)
  Car_start "$@" # Car_start function must be created by the user
}
```

Here we have two attributes, `speed` and `brand`, so **usually** at least two
new processes are spawned to just prepare the local variables used in the
method. This is where some shells shine more than others, **ksh93** for instance
does not necessarily spawn a new process for command substitution.

For example let's consider the following:

```sh
say_hello() {
  echo 'Hello!'
}

for i in 1 2 3 4 5; do
  result=$(say_hello)
done
```

Is there a need to create a new process to capture `say_hello` output? No, not
necessarily, and `echo` usually is a builtin, which means everything happening
here can be strictly done by the shell. Now let's see what the most common
shells do:

bash/mksh/zsh/dash/busybox/osh/yash
```
sh$ strace -cfe process bash say_hello.sh
strace: Process 27040 attached
strace: Process 27041 attached
strace: Process 27042 attached
strace: Process 27043 attached
strace: Process 27044 attached
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 48.70    0.000337          33        10         5 wait4
 28.76    0.000199         199         1           execve
 21.97    0.000152          30         5           clone
  0.58    0.000004           2         2         1 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.000692                    18         6 total
```

Total 5 new processes (excluding the initial execve), all `clone` calls, but
**ksh93** on the other hand:

```
sh$ strace -cfe process ksh say_hello.sh
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 97.10    0.000536         536         1           execve
  2.90    0.000016           8         2         1 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.000552                     3         1 total
```

0 clone calls, **ksh93** executes that script in a single process, fascinating
right? But you might think that it just understands that `result` is never used,
or that maybe if you were creating some variables inside that subshell it would
have needed to fork to be able to throw away the environment one it's done.
But no, even in all those cases **ksh93** does not fork.

Now let's take an example with **clash** to show how big of a difference this
can make:

```sh
sh$ shuf --input-range 1-30 > shuffled-numbers.txt # use a fixed random sequence
```
And here is `sort.sh`
```sh
. ./clash
. ./lib/list

List foo $(cat shuffled-numbers.txt)
foo_sort
foo_print
```

This creates a list `foo` of 30 shuffled elements, sorts it and prints it on
the standard output.

*(Please note that the results may vary a little due to the shuffling part of
this script, the version of the shells used, your machine etc. This is in no way
a serious benchmark)*

```
sh$ strace -cfe process ksh sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 73.24    0.000594          54        11         9 execve
 17.39    0.000141          23         6         4 wait4
  7.64    0.000062          31         2           clone
```

2 clone calls for **ksh93** that's not too bad, especially since one them is
caused by the need to execute `cat`, which isn't a builtin.

The failed `execve` calls are mostly due to some compatibility checks when
sourcing **clash**, we can ignore them.

Let's also check the running time that we'll compare with the other shells
```
sh$ time ksh sort.sh
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

real    0m0.081s
user    0m0.074s
sys     0m0.007s
```

Now let's try it with the other shells:

```
sh$ strace -cfe process bash sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 72.67    0.085384          50      1686       843 wait4
 27.11    0.031848          37       843           clone
  0.22    0.000255         127         2           execve
```

Around 800 clone calls, yes that's right, creating, sorting and printing a list
of 30 elements spawns no less than 800 processes with bash/zsh/dash/busybox.

And looking at bash running time:
```
sh$ time bash sort.sh
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

real    0m0.396s
user    0m0.293s
sys     0m0.119s
```

Roughly 5 times slower.

Don't see **mksh** mentioned? Well it is a bit special since it turns out that
in mksh `printf` is not a builtin, and clash heavily relies on `printf` which
explains the following results:

```
sh$ strace -cfe process mksh sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 93.57    0.829082         212      3908      1954 wait4
  3.83    0.033943          17      1954           clone
  2.43    0.021552          19      1112           execve
```

Almost 2000 **clone** and 1000 **execve**, this is wild.

```
sh$ time mksh sort.sh
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

real    0m5.224s
user    0m2.015s
sys     0m3.282s
```

5 seconds to sort a list of 30 elements, now you know what to use to train your
machine learning models.

Want more? Let's try with 1000 elements instead of 30, see how it goes with
**bash**:

```
sh$ strace -cf bash sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 66.95   33.078252         367     90108     45054 wait4
 27.64   13.652942         303     45054           clone
  1.89    0.932990           1    535447           rt_sigprocmask
[...]
```

```
sh$ time bash sort.sh
[...]
real    1m8.619s
user    0m17.712s
sys     0m53.141s
```

More than a minute to sort 1000 elements, please also note that on the total
90108 `wait4` syscalls, half of them resulted in errors (ECHILD, No child
processes, haven't investigated why though), same goes for **zsh** although it
definitely runs a bit faster.

Interestingly **dash** shows a negligible number of `wait4` errors, and takes
around 3x less time to run on average:

```sh
sh$ strace -cf dash sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 45.31    4.239508          94     45056         2 wait4
 39.09    3.657587          81     45054           clone
  5.53    0.516913           5     97154           read
[...]
```

On the other hand busybox is the complete opposite with 90% of its `wait4` calls
failing.

All this to say that you can't expect good performance when starting to use a
lot of objects, that's the way shell command substitution works and **clash**
design definitely doesn't help with that.

Even **ksh93** that avoids spawning new processes struggles to keep the lead in
terms of speed when going over a few thousands of objects in use.

## Why not generate variables instead of functions?

This is a valid point, especially one of the first thing that comes to mind is
why generate a function to set the attribute value (`obj_attr_is foo`) when you
could just do `obj_attr=foo`.

Well `obj_attr=foo` works fine, but what about `objname=obj;
${objname}_attr=foo`? It doesn't unless you cheat with assignment builtins such
as `declare`/`typeset`/`local` or use `eval`, which is going to either reduce
the portability of your script or is going to make it really hard to read. This
pattern of `${obj}_attr` is something you encounter all the time, once you pass
a variable name as a parameter of a function, how do you directly use its
original global variable name? You can't.

This is especially true for object-oriented programming, how do you make use of
`$self`? You can't use a full object name in class methods either, or else it
would only work for a specific instance and breaks the whole point of classes
being a generic model.

Let's take an example:

```sh
. ./clash

class Car \
  running \
  _start

Car_start() {
  "$self"_running=true
}

Car truck

truck_start
```

When calling that method we get:
```
truck_running=true: command not found
```
Because the shell didn't recognize an assignment (due to trying to expand a
variable in the lvalue) it deduced that we were trying to run a command called
`truck_running=true`, which of course does not exist.

Now we could trick the shell into deferring the assignment after the evaluation
of `$self` by using an assignment builtin like `declare/typeset/export`:

```sh
  declare -g "$self"_running=true # -g to set it in the global scope
```

It works!

Though the main issue here is portability, **busybox** and **dash** do not
support `declare` and `typeset`. On the other hand `readonly`, `export` and
`local` have a very specific purpose which doesn't fit here.

So let's just use the almighty `eval`!
```sh
  eval "$self"_running=true
```

That's even easier to read than `declare -g`.

But now what if we want to assign a value passed in parameter of the method?

```sh
class Car      \
  running      \
  song_playing \
  _start

Car_start() {
  eval "$self"_running=true
  eval "$self"_song_playing="$1"
}

Car truck

truck_start 'Another One Bites the Dust'
```

You should see something like: `One: command not found`, this is due to how
`eval` works. Simply put, `eval` just adds another round of expansion, meaning
you can take your line, expand the variables and then run it as if it was
written in your script as is. So basically the `eval` line is replaced by:

```sh
  truck_song_playing=Another One Bites the Dust
```

And this is not valid due to that space after `Another`, the shell understands
it as pass `truck_song_playing=Another` as an environment variable for the `One`
command, which does not exist. We could add some quotes, like `"'$1'"`, but what
if the string passed contains a single quote as well? This would break. Instead
we should find a way to defer `$1` expansion to happen after `$self` expansion.

So basically we want this to expand first to:

```sh
  truck_song_playing=$1
```

Simple enough, single quote your variable so that it gets expanded **after**
`eval` first expansion:


```sh
  eval "$self"_song_playing='$1'
```

Fantastic! Though you might understand where this is going, this is way more
complicated than it should be, and usually to avoid repeated complexity we tend
to create functions, which is **clash** `<obj_attr>_is` method whole point.

Using variables is not all bad though, if used internally it would definitely
improve a few things, including speeding up methods by avoiding one command
substitution per attribute when setting up the method context. This strategy is
available in **clash**, but disabled by default since performance improvement
was not as meaningful as expected (to use it set `CLASH_HYBRID=true` in the
shell environment).

Also one thing to note is that using variables would mean bypassing `getattr`
and `setattr` hooks when using direct access, which would defeat their purpose.

## I don't see `sh` in the compatibility table, does this support the Bourne shell?

Nowadays (2020), the Bourne shell and `/bin/sh` are usually two different
things. `sh` is on most systems just a symlink to another modern shell, usually
to bash, dash, mksh or busybox:
```
sh$ ls -l /bin/sh
lrwxrwxrwx. 1 root root 4 Aug  5 03:21 /bin/sh -> bash
```

So basically in this case running a script with `sh` is exactly the same as
running `bash --posix`, which also doesn't mean that it only uses POSIX defined
features but that it respects POSIX, which means that you can still use bash
arrays in POSIX mode for instance even if those do not appear once in the
[specification](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/contents.html).

So does **clash** work with the Bourne shell? No, the original Bourne shell is
not POSIX compliant, it didn't even support functions.

**clash** should be able to work with any POSIX compliant shell supporting
`local` assignments, and if it doesn't feel free to open an issue.

# Notes

- Using `local` (`typeset` for **ksh**) was a hard choice, considering it was the only thing preventing the framework to be strict POSIX compliant, but `local` can be really handy to avoid flooding all the frames with definitions and most importantly `local` prevents lower frames from overwriting upper frames variables values when using recursion.
- This project was inspired by the amazing **bash-oo-framework** (https://github.com/niieani/bash-oo-framework), which is a huge project aiming to provide modern features to bash. While object-oriented programming works perfectly well in **bash-oo-framework**, it diverges from the usual shell syntax to provide type safety and other cool features, it almost feels like a different language, which can take some time to learn. This was the motivation to write something closer to the shell syntax and more portable.
