+++
date = '2026-11-04T22:43:20Z'
draft = true
title = 'C++ casts done right'
author = 'Johannes Quast'
+++

Everything seemed so easy when I first touched C++ code for a hobby project in 2015 more than 10 years ago.
Before C++, I primarily "worked" with other programming languages where converting from one datatype to another is easy:

* In Java, you can just write `MyCoolType coolVar = (MyCoolType) oldVar;` and boom, you are done (I'm generalizing but you get the point).
* In Python and Lua, we do not even concern ourself with this discussion. We have work to do.

In C++, things are bit more ... _complicated_.
So let's get started with the different ways we can have fun (and shoot ourself in the foot).

## C-style casting

What set C++ apart from most other languages is its rather long history and the long legacy inherited from C itself.
Even though C offers a way to cast between values, it is pretty much limited to the following operations:

* **Cast between two values**:
* 

## `static_cast`

With `static_cast`

## `dynamic_cast`

## `const_cast`

## `reinterpret_cast`

## `std::bit_cast`

## `std::dynamic_pointer_cast`
