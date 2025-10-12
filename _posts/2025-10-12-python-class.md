---
layout: single
title: "[Python] 1.1 About Class" 
date: 2025-09-24
categories: Python
#header:
  #overlay_image: /assets/images/docker-container-tech.svg
  #overlay_filter: 0.5
  #teaser: /assets/images/docker-container-tech.svg
  #caption: "Create Docker Image"
Typora-root-url: ../
---

# Python Classes: A Beginner's Guide to Object-Oriented Programming

At some point, as I unconsciously used generative AI for coding, I thought that Python's basics were getting forgotten, so I thought I should review them little by little every day to strengthen the basics.

From today, I will study little by little every day and post about the fundamental and basic content about Python.

First of all, let's take an easy look at class, which is a barrier that is difficult to pass on to novice programmers.

Before we go into the concept of a class, let's learn about the need for a class through a simple example.

Let's make a simple calculator program and find out what is class.

## Understanding classes through a simple calculator

```python
class Calculator:
    def __init__(self):
        self.result = 0
    
    def add(self, num):
        self.result += num
        return self.result

cal1 = Calculator()
cal2 = Calculator()
```

```python
print(cal1.add(3))
print(cal1.add(4))
print(cal2.add(3))
print(cal2.add(7))
```

```
3
7
3
10
```

The calculator objects `cal1` and `cal2`, created from the `Calculator` class, each perform their own tasks and maintain independent result values. By using a class in this way, we can easily create additional calculator instances as needed—making it much more convenient than relying solely on functions.

It is also easy to add new features. Let's add a subtraction function to the class.

```python
class Calculator:
    def __init__(self):
        self.result = 0

    def add(self, num):
        self.result += num
        return self.result

    def sub(self, num):
        self.result -= num
        return self.result
    
cal1 = Calculator()
cal2 = Calculator()
```

```python
print(cal1.add(3))
print(cal1.add(4))

print(cal2.sub(3))
print(cal2.sub(7))
```

```
3
7
-3
-10
```

## Understanding the class concept

- A cookie mold = class
- Cookies made with the mold = objects

A class can be thought of as a blueprint or design that allows you to repeatedly create identical structures. 
An object, on the other hand, is a creation made from that class.

Each object created from a class has its own unique characteristics, and objects do not interfere with one another.

```python
class Cookie:
    pass

a = Cookie()
b = Cookie()
```

The class above does not contain any actual functionality: it is just an empty shell.
However, even such a class can still be used to create objects.

## Difference between object and instance

When you create `a = Cookie()`, `a` is an object.
At the same time, the object `a` is an instance of the `Cookie` class.

In other words, the term instance is used to describe the relationship between an object and the class it was created from.
So, it's common to say that "`a` is an instance of `Cookie`."

## Creating a four-function arithmetic class

```python
class FourCal:

    def __init__(self, first, second):
        self.first = first
        self.second = second

    def add(self):
        result = self.first + self.second
        return result
    
    def sub(self):
        result = self.first - self.second
        return result
    
    def mul(self):
        result = self.first * self.second
        return result
    
    def div(self):
        result = self.first / self.second
        return result
```

```python
a = FourCal(4,2)
print(a.first)
print(a.second)
```

```
4
2
```

```python
print(a.add())
print(a.sub())
print(a.mul())
print(a.div())
```

```
6
2
8
2.0
```

## Class inheritance

When creating a new class, you can make it inherit the features of another class.

For example, let's create a `MoreFourCal` class that inherits from the `FourCal` class.

```python
class MoreFourCal(FourCal):
    pass

a = MoreFourCal(4,2)

print(a.add())
print(a.sub())
print(a.mul())
print(a.div())
```

```
6
2
8
2.0
```

### Why use inheritance?

Inheritance is useful when you want to add new functionality or modify existing behavior without changing the original class.
If the base class is part of a library or cannot be modified directly, inheritance allows you to extend it safely.

For instance, you can implement a method in MoreFourCal that calculates $a^b$.

```python
class MoreFourCal(FourCal):
    def pow(self):
        result = self.first ** self.second
        return result
    
a = MoreFourCal(4,2)

print(a.pow())

print(a.add())
print(a.sub())
print(a.mul())
print(a.div())
```

```
16
6
2
8
2.0
```

## Method overriding

Let's run the FourCal class in this way:

```python
a = FourCal(4,0)
a.div()
```

```
ZeroDivisionError: division by zero
```

When you divide a number by zero, a `ZeroDivisionError` occurs.

To handle this, let's create a new class `SafeFourCal` that inherits from `FourCal`.

```python
class SafeFourCal(FourCal):
        def div(self):
                if self.second == 0:
                        return 0
                else: 
                        return self.first/self.second 
```

You can override (redefine) a method from the parent class with the same name - this process is called method overriding.

```python
a = SafeFourCal(4,0)

print(a.div())
```

```
0
```

## Class variables

While instance variables (object variables) maintain independent values for each object, there are also class variables, which behave differently.

```python
class Family:
    lastname = "kim"
```

In this example, lastname is a class variable.

A class variable is defined inside the class but outside any methods, and it can be accessed using the syntax ClassName.variable_name.

```python
Family.lastname
```

```
'kim'
```

```python
a = Family()
b = Family()

print(a.lastname)

print(b.lastname)
```

```
kim
kim
```

```python
Family.lastname = "park"

print(a.lastname)
print(b.lastname)
```

```
park
park
```

If you modify the value of a class variable, the change will be reflected across all instances created from that class — meaning class variables are shared among all objects.

### What happens if an instance variable has the same name?

```python
a.lastname = "choi"

print(a.lastname)
```

```
choi
```

If an object creates its own variable with the same name as a class variable, the object variable takes precedence only within that specific instance.
In other words, changing a.lastname does not affect Family.lastname — it simply creates a new instance variable that shadows the class variable.

```python
print(a.lastname)

print(Family.lastname)

print(b.lastname)
```

```
choi
park
park
```
## Sources

- Jump to Python: https://wikidocs.net/28