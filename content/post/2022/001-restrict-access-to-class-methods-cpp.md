---
title: "C++: Restrict Access To Class Methods From All But Few Outsiders"
date: 2022-12-02T00:31:11+05:30
draft: false
aliases: []
description: ''
slug: 'restrict-access-to-class-methods-cpp'
tags: [cpp, design-patterns]
type: post
weight: 0
---

### The Problem

Say we have a class `ParkingSlot` with method `Park()`. Outside the class, this method should only be accessible to another class `ParkingLot`. How do we implement this?

One (really lazy) way is to make the method private and declare the other class a friend.

```cpp
class ParkingSlot
{
    // omitting other private members

    void Park(Vehicle *vehicle);
    friend class ParkingLot;

public:
    // omitting public members
};
```
This works, but now the friend class can access all other private members. This breaks encapsulation and is the reason why friendship is often frowned upon.

Fortunately, there are better ways to solve this.


### Attorney-Client Idiom

Friendship did help us above, but it granted more access than we desired. Is there a way to restrict this friendship?

Instead of granting complete access, we can introduce a trusted attorney class as the intermediary. We declare `ParkingLot` (or its specific methods) as a friend of the attorney class, and the attorney itself is a friend of `ParkingSlot`.

```cpp
class ParkingSlot   // client class
{
    void Park(Vehicle *);
    friend class ParkingSlotAttorney;    // introducing Saul Goodman
public:
};

class ParkingLot    // accessor class
{
    ParkingSlot pslot;
public:
    void Park(Vehicle *);
};

class ParkingSlotAttorney
{
    friend void ParkingLot::Park(Vehicle *);    // grant access to specific method
    static void Park(ParkingSlot &pslot, Vehicle *vehicle)  // proxy method
    {
        pslot.Park(vehicle);
    }
};

void ParkingLot::Park(Vehicle *vehicle)
{
    ParkingSlotAttorney::Park(pslot, vehicle);   // better call Saul
}
```
The accessor method `ParkingLot::Park()` has complete access to the attorney, which has full access to the client. But the attorney has only proxy methods and no data members. So the access is restricted.

As per our need, we can declare more friendly accessors in the attorney class and add more proxy methods. We can also declare multiple attorneys of the client, each granting access to a different set of methods.

### PassKey Idiom

There is another clever way to restrict this friendship. This, too, involves an intermediary: the Key class.

```cpp {linenos=true}
class Key
{
    friend class ParkingLot;
    Key() {}    // only friends can create an instance
    Key(const Key &) {}
};

class ParkingSlot
{
public:
    void Park(Key, Vehicle *);  // needs an instance of Key
};

class ParkingLot
{
    ParkingSlot pslot;
public:
    void Park(Vehicle *vehicle)
    {
        Key key;    // creating an instance of Key
        pslot.Park(key, vehicle);
    }
};
```
You see this? The method we need to restrict access to (`ParkingSlot::Par()`) is now public, but one of its arguments demands an instance of the Key class. But only a friend of Key can supply this instance hence restricting access.

The instance isn't actually used in the method. It is just a tool to restrict access.

Note that the copy constructor is private. It prevents unauthorized classes/functions from passing an instance of Key as an argument in a function call. An unauthorized actor may still acquire an instance of Key if it is returned by some method of the friendly class, but it won't be able to use it.

We need to explicitly declare both the constructor and copy constructor as private because the default ones are public in C++.

A special case is when we apply the PassKey restriction on the constructor of a class. This can allow object creation through a particular class without compromising private members.

```cpp
class Key { friend class ParkingLot; Key() {} Key(const Key &) {} };

class ParkingSlot
{
    // omitting private members
public:
    ParkingSlot(Key key){}
    // omitting other public members
};

class ParkingLot
{
    ParkingSlot *pslot;
public:
    // instead of explicitly creating a key instance...
    // we can perform value-initialization using empty initializer {}
    ParkingLot(): pslot(new ParkingSlot({})) {}
};
```

Another cool thing is that we can generalize the Key class using templates. This is especially useful if we want to provide restricted access to multiple classes.

```cpp
template<typename T>
class Key {
    friend T;
    Key() {}
    Key(const Key &) {}
};

class B;
class C;

class A {
public:
    void Task1(Key<B>, int);
    void Task2(Key<C>, int);
};

class B {
    void TaskB(A &a) {
        a.Task1({}, 2);
    }
};

class C {
    void TaskC(A &a){
        a.Task2({}, 1);
    }
};
```

References:
- [StackOverflow: Clean Granular Friend Equivalent](https://stackoverflow.com/a/3218920/8216204)
- [WikiBooks: More C++ Idioms/Friendship and the Attorney-Client](https://en.wikibooks.org/wiki/More_C++_Idioms/Friendship_and_the_Attorney-Client)