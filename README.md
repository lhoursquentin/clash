# Clash
Portable shell oriented-object programming

# What
Define classes and instanciate objects in shell with the usual shell functions syntax.

Compatible with most Bourne like shells

| Shell    | Versions tested           | Support                          |
| -------- | ------------------------- | -------------------------------- |
| bash     | 3.0 / 4.4 / 5.0           | Full                             |
| busybox  | 1.28 (ash w/ bash compat) | Full                             |
| dash     | 0.5.10.2                  | Full                             |
| ksh      | AT&T 93u+                 | clash 100%, lib 80%, test 50%    |
| mksh     | R57                       | Full                             |
| osh      | 0.6.pre21                 | See [osh branch](https://github.com/lhoursquentin/clash/tree/osh) |
| yash     | 2.48                      | Full                             |
| zsh      | 5.6.2                     | Full                             |

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

# Notes

- Using `local` (`typeset` for **ksh**) was a hard choice, considering it was the only thing preventing the framework to be strict POSIX compliant, but `local` can be really handy to avoid flooding all the frames with definitions and most importantly `local` prevents lower frames from overwriting upper frames variables values when using recursion.
- This project was inspired by the amazing **bash-oo-framework** (https://github.com/niieani/bash-oo-framework), which is a huge project aiming to provide modern features to bash. While oriented object programming works perfectly well in **bash-oo-framework**, it diverges from the usual shell syntax to provide type safety and other cool features, it almost feels like a different language, which can take some time to learn. This was the motivation to write something closer to the shell syntax and more portable.
