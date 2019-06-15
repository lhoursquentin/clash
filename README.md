# Clash
Portable shell oriented-object programming

# What
Define classes and instanciate objects in shell with the usual shell functions syntax.

Compatible with most Bourne like shells, including:

- dash
- bash
- ksh
- zsh
- busybox shell (ash with bash compatibility)

## Features

Most of what you would expect from basic object-oriented programming is supported:

- classes
- objects
- attributes
- methods
- constructor
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
class Car \
  brand   \
  speed   \
  _start  \

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
_Car() {
  echo "New $brand added to the garage"
}
```

```bash
# Create a new car object named 'cool_bolid', _Car function is called
Car cool_bolid Ferrari 200

# Create another one, _Car function is called
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

# upgrade the engine (generated setters are suffixed by _is)
family_truck_speed_is 120

# start it again
family_truck_start
```

```
Family truck speed: 90
Starting family_truck: Vroooooooom
```

Note: this code can be found in the `examples` folder.

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
