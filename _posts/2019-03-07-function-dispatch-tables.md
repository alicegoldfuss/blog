---
layout: post
title: "Function Dispatch Tables in C"
description: "A performant alternative to If/Else"
tags: [technical, code, performance, c]
modified: 2018-3-7
categories: 
---

One of the most powerful concepts in programming is branching, which most new programmers learn as If-Else statements. This approachable logic flow makes it easy to navigate choices and is present in nearly every codebase out there. In some languages there is a variation of this logic flow called a case statement (or switch statement), which is a cleaner way of dealing with many If-Else sequences.

However, there is yet another variation that, when used in the right circumstances, can make your code more elegant and your program more performant: the **function dispatch table**.

<!-- more -->

## Quick Review

Let’s review case/switch statements. Say we want to perform some algebraic operations on two integers.

```c
#include <stdio.h>
 
int main () {

   int first, second, choice, result;

   first = 2;
   second = 3;
   choice = 1;

    switch(choice) {
        case 0 :
            result = first + second;
            break;
        case 1 :
            result = first - second;
            break;
        case 2 :
            result = first * second;
            break;
        case 3 :
            result = first / second;
            break;
    }

   printf("Result is %d\n", result);
 
   return 0;
}
```

When we compile and run this program, it will print `Result is -1` because `choice = 1` which resolves to `2 - 3`. If `choice = 2` it would give us `2 * 3` or `Result is 6` instead.

This program works, but it has some issues. For one, it's difficult to tell what each `case` does at first glance without examining the operations it actually performs on `first` and `second`. This isn't a real problem for simple one-liners, but what if we needed to perform more complex operations? Sure, we could add comments, but we'd still have a very long case statement to maintain.

Let's break the operations into functions.

```c
// my_program.h

#ifndef MY_PROGRAM_H_ 
#define MY_PROGRAM_H_

int add(int first, int second);
int sub(int first, int second);
int mult(int first, int second);
int divide(int first, int second);

#endif
```

```c
// my_program.c

#include <stdio.h>
#include "my_program.h"

int main () {

   int first, second, choice, result;

   first = 2;
   second = 3;
   choice = 1;

    switch(choice) {
        case 0 :
            result = add(first, second);
            break;
        case 1 :
            result = sub(first, second);
            break;
        case 2 :
            result = mult(first, second);
            break;
        case 3 :
            result = divide(first, second);
            break;
    }

   printf("Result is %d\n", result);
 
   return 0;
}

int add(int first, int second){
    return first + second;
}

int sub(int first, int second){
    return first - second;
}

int mult(int first, int second){
    return first * second;
}

int divide(int first, int second){
    return first / second;
}
```

Yes, there are more lines of code (and now more files!) but it's easier to understand what each `case` statement does. Changing any of the functions (or adding new options) is more straight-forward and the `case` statement itself is shorter and easier to maintain.

But, what if we had 20 cases? Or 100? Not only would our code be long and difficult to navigate, it would be inefficent. Having 100 cases means we could potentially check 100 cases before selecting one. `case` statements are O(n), where `n` is the number of case/switch statements possible.

Enter the function dispatch table.

## What even is it

A **function dispatch table**, also known as a **jump table**, is an array of function pointers. Yes, I said *pointers*. Don't worry! We got this.

Quick refresher: a **pointer** is a location in memory aka a memory address. That memory address can contain anything: an integer, a float, the middle of a string. It can also store the name of a function, also known as its label. A function's name is its address in memory.

But, we don't have to go that deep! All you need to know for this is that you can use pointers to go directly to functions. Using this knowledge, we can build an array where the array indices point (or *jump*) to the functions assigned to them. Essentially, we will have an array like `my_array = {func0, func1, func2, func3}` and `my_array[3]` will take us directly to `func3` without having to evaluate for `func0` / `func1` / `func2` first. See how handy that could be?

So, how do we build a function dispatch table? I find it's easiest to work from the inside out and define our function type first.

Let's take a look at our algebraic function declarations:

```c
int add(int first, int second);
int sub(int first, int second);
int mult(int first, int second);
int divide(int first, int second);
```

They all have the same format. They take two `int`s as arguments and return another `int`. If we wanted to abstract these functions into a general form, it would look like:

```c
int math_function(int first, int second);
```

We could build our function dispatch table using this generalized form, but that can be a bit unwieldly. Therefore, I recommend using `typedef` and using this function declaration as a template for your own custom method.

```c
typedef int math_function(int first, int second);
```

We have now defined `math_function` as a custom function that takes two `int`s and returns another.

It's time to declare our array of function pointers! But, let's start a little simpler. How would we declare an array of four integers?

```c
int my_array[4];
```

What if we wanted to declare an array of four *pointers* to integers?

```c
int *my_array[4];
```

Now, instead of integers, let's point to our custom function type.

```c
math_function *my_array[4];
```

We have declared an array of four function pointers that point to four functions of type `math_function`!

Let's define the array with our four math functions:

```c
math_function *my_array[4] = {
    add,
    sub,
    mult,
    divide
};
```

We don't have to provide anything except the function names. Since they are type `math_function` we already know they take two `int`s and return one `int`.

## Putting it together

We have now built a function dispatch table! Let's rewrite our program to use it:

```c
// my_program.h

#ifndef MY_PROGRAM_H_ 
#define MY_PROGRAM_H_

int add(int first, int second);
int sub(int first, int second);
int mult(int first, int second);
int divide(int first, int second);

typedef int math_function(int first, int second);

#endif
```

```c
// my_program.c

#include <stdio.h>
#include "my_program.h"

int main () {

    int first, second, choice, result;
       
    first = 2;
    second = 3;
    choice = 1;

    math_function *my_array[4] = {
        add,
        sub,
        mult,
        divide
    };

    result = my_array[choice](first, second);

    printf("Result is %d\n", result);
 
    return 0;
}

int add(int first, int second){
    return first + second;
}

int sub(int first, int second){
    return first - second;
}

int mult(int first, int second){
    return first * second;
}

int divide(int first, int second){
    return first / second;
}
```

The line

```c
result = my_array[choice](first, second);
```

is saying "Result equals the function at array index `choice` given parameters `first` and `second`" or "Result equals the function at array index 1 given parameters 2 and 3." The function at `array[1]` is `sub`, so we jump right to the `sub` function and give it `2` and `3` as its parameters. Our program is now more efficient than our `case` statement with less lines of code.

Function dispatch tables can be very powerful and also very complex. You should always balance performance with ease of maintainence. Usually I would only use a function dispatch table if I had a high number of cases to evaluate, or if I was concerned more about performance than about readability. An example of a complex but necessary function dispatch table is the Linux kernel's [syscall dispatcher](https://linux-kernel-labs.github.io/master/lectures/syscalls.html#system-call-table){:target="_blank"}. As you can see, dispatch tables can quickly become opaque to new contributors and require time to understand.

I hope this post has been helpful! I encourage you to mess around with function dispatch tables on your own to get a better sense of pointers and flow.


