# Clash
Portable shell oriented-object programming

# What
Define classes and instantiate objects in shell with the usual shell functions
syntax.

Compatible with most Bourne like shells like bash/dash/ksh/zsh

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
  echo "New $brand added to the garage"
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
Object-oriented programming is the solution to those issues, but does not exist natively in shell.

The goal of this framework to provide a simple solution to the missing object paradigm and to be as portable as possible (working with **dash**/**bash**/**ksh**/**zsh**/**busybox ash**), the only non POSIX feature used being `local` (more info in the **notes** section).

Most importantly the syntax to create and use classes and objects is the exact same as usual shell, and that is because  what clash does is only generate functions, almost no parsing is done, and the grammar isn't altered using aliases.

# How

Two functions are generated per attribute, one for the getter with the attribute value hardcoded, and the other for the setter which when called redefines the original getter function with a new hardcoded value.

Methods are just generated functions wrapping a local declaration of all the attributes and a call of the initially declared function so that it has access to all the attributes in its frame.

All these methods and attributes are generated using `eval` and playing with quotes (a lot of quotes) to preserve IFS characters.

# FAQ

## What's the point when python, perl and other more robust scripting languages exist?

Agreed, that wasn't reasonable, please submit a pull request so that we can
start converting clash to rust and target WebAssembly.

## What's the performance of clash? Show me the numbers!

Pretty bad! But this depends heavily on which shell you are using and the number
and size of the objects you are creating. Creating a dozen of objects with a few
methods and attributes and it won't be noticeable. However creating hundreds or
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

Here we have two attributes, so **usually** at least two new processes are
spawned to just prepare the local variables used in the method. This is where
some shells shine more than others, **ksh93** for instance does not necessarily
spawn a new process for command substitution.

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

bash/mksh/zsh/dash
```
$ strace -cfe process bash say_hello.sh
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
$ strace -cfe process ksh say_hello.sh
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 97.10    0.000536         536         1           execve
  2.90    0.000016           8         2         1 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.000552                     3         1 total
```

0 clone calls, **ksh93** executes that script in a single process, fascinating
right? But you might think that it just understands that result is never used,
or that maybe if you were creating some variables inside that subshell it would
have needed to fork to be able to throw away the environment one it's done.
But no, even in all those cases **ksh93** does not clone/fork.

Now let's take an example with clash to show how big of a difference this can
make:

```sh
. ./clash
. ./lib/list

List foo $(seq 30 | shuf)
foo_sort
foo_print
```

This creates a list foo of 30 shuffled elements, sorts it and prints it on
stdout.

*(Please note that the results may vary a little due to the shuffling part of
this script, this is in no way a serious benchmark)*

```
$ strace -cfe process ksh sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 63.43    0.000921         102         9         4 wait4
 30.92    0.000449          32        14        11 execve
  5.10    0.000074          24         3           clone
  0.55    0.000008           1         6         3 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.001452                    32        18 total
```

3 clone calls for **ksh93** that's not too bad, especially since 2 of them are due
to calling 2 external binaries, `seq` and `shuf`.

Let's also check the running time that we'll compare with the other shells
```
$ time ksh sort.sh
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

real    0m0.081s
user    0m0.074s
sys     0m0.007s
```

Now let's try it with the other shells:

```
$ strace -cfe process bash sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 69.10    0.069408          41      1653       826 wait4
 30.67    0.030803          37       827           clone
  0.23    0.000230          76         3           execve
  0.01    0.000006           1         6         3 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.100447                  2489       829 total
```

Around 800 clone calls, yes that's right, creating and sorting a list of 30
elements spawns no less than 800 processes with bash/zsh/dash/busybox.

And looking at bash running time:
```
$ time bash sort.sh
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

real    0m0.396s
user    0m0.293s
sys     0m0.119s
```

Roughly 5 times slower.

Don't see mksh mentioned? Well it can't be compared to bash/dash etc. since it
turns out that in mksh `printf` is not a builtin, and clash heavily relies on
`printf` which explains the following results:

```
$ strace -cfe process mksh sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 93.74    0.867441         225      3850      1924 wait4
  3.73    0.034554          17      1925           clone
  2.36    0.021831          19      1099           execve
  0.17    0.001548           0      2198      1099 arch_prctl
------ ----------- ----------- --------- --------- ----------------
100.00    0.925374                  9072      3023 total
```

Almost 2000 clone and 1000 execve, this is wild.

```
$ time mksh sort.sh
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30

real    0m4.978s
user    0m1.953s
sys     0m3.104s
```

5 seconds to sort a list of 30 elements, now you know what to use to train your
machine learning models.

Want more? Let's try with 1000 elements instead of 30, see of it goes with
**bash**:

```
$ strace -cf bash sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 65.29   36.087855         399     90243     45121 wait4
 29.03   16.042587         355     45122           clone
  2.03    1.122136           2    536207           rt_sigprocmask
[...]
```

```
$ time bash sort.sh 1000
[...]
real    1m5.525s
user    0m18.009s
sys     0m49.866s
```

More than a minute to sort 1000 elements, please also note that on the total
90243 `wait4` syscalls, half of them resulted in errors (ECHILD, No child
processes, haven't investigated why though), same goes for **zsh** although it
definitely runs a bit faster.

Interestingly **dash** shows a negligible number of `wait4` errors, and takes
around 3x less time to run on average:

```sh
$ strace -cf dash sort.sh
[...]
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 49.54    4.880581         108     45016         8 wait4
 35.92    3.538674          78     45008           clone
  5.03    0.495295           5     97044           read
[...]
```

On the other hand busybox has 90% of its wait4 failing.

## Why not generate variables instead of functions?

This is a valid point, especially one of the first thing that comes to mind is
why generate a function to set the attribute value (`obj_attr_is foo`) when you
could just do `obj_attr=foo`. Well `obj_attr=foo` works fine, but what about
`objname=obj; ${objname}_attr=foo`? It doesn't unless you cheat with assignment
builtins such as `declare`/`typeset`/`local` or use `eval`, which isn't going to
like quotes.  This pattern of `${obj}_attr` is something you encounter all the
time, once you pass an object name as a parameter of a function, how do you
directly use its original global variable name? You can't.

Though using variables would definitely improve a few things, including speeding
up methods by avoiding one command substitution per attribute when setting up
the method context. This is definitely something that might be worth trying at
least.

Also one thing to note is that using variables would mean bypassing `getattr`
and `setattr` hooks when using direct access, which would defeat their purpose.

## I don't see `sh` in the compatibility table, does this support the Bourne shell?

Nowadays (2020), the Bourne shell and `sh` are usually two different things.
`sh` is on most systems just a symlink to another modern shell, usually to
bash/dash/mksh/busybox:
```
$ ls -l /bin/sh
lrwxrwxrwx. 1 root root 4 Aug  5 03:21 /bin/sh -> bash
```

So basically in this case running something `sh` is exactly the same as running
`bash --posix`, which also doesn't mean that it only uses POSIX defined features
but that it respects POSIX, which means that you can still use bash arrays in
POSIX mode for instance even if those do not appear once in the [specification](http://pubs.opengroup.org/onlinepubs/9699919799/utilities/contents.html).

So does clash support the Bourne shell? Well it hasn't been tested (though it's
unlikely since the original Bourne shell didn't support functions), but
technically any POSIX compliant shell supporting `local` should be able to use
**clash**, and if it doesn't feel free to open an issue.

# Notes

- Using `local` (`typeset` for **ksh**) was a hard choice, considering it was the only thing preventing the framework to be strict POSIX compliant, but `local` can be really handy to avoid flooding all the frames with definitions and most importantly `local` prevents lower frames from overwriting upper frames variables values when using recursion.
- This project was inspired by the amazing **bash-oo-framework** (https://github.com/niieani/bash-oo-framework), which is a huge project aiming to provide modern features to bash. While oriented object programming works perfectly well in **bash-oo-framework**, it diverges from the usual shell syntax to provide type safety and other cool features, it almost feels like a different language, which can take some time to learn. This was the motivation to write something closer to the shell syntax and more portable.
