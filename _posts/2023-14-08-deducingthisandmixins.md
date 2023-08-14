---
layout: post
title: "Deducing this & Mixins"
date: 2023-08-14 00:00:00 -0000
---

Last post I looked at how to implement a compile time version of the observer pattern. I used the following pattern:
```cpp
template<typename Tuple>
struct AttatchType
{
    void notify(int in)
    {
        // Notify observers
    }

    Tuple m_observer;
};

struct Subject
{
    template<typename... T>
    auto attach(T&& ...observer)
    {
        return AttatchType<decltype(std::make_tuple(std::forward<T>(observer)...))>{std::make_tuple(std::forward<T>(observer)...)};
    }
};
```

The subject class here is kind of useless, why don't we just construct the AttachType? The reason is that the subject class might have some functionality we want to use, for example the subject might be an MQTT subscriber, and whenever it receives data it should be relayed to the attached observers.

In the example above there is no way the subject can call notify, since the AttatchType is a completely different type. The solution to this is Mixins an would look like this:

```cpp
template<typename Base, typename... T>
struct AttatchType: public Base
{
    AttatchType(Base base, T... observers): 
    Base(base),
    m_observer(observers...)
    {}

    void notify(int in)
    {
        // Notify observers
    }

    std::tuple<T...> m_observer;
};

struct Subject
{
    void baseMethod()
    {

    }
    template<typename... T>
    auto attach(T&& ...observer)
    {
        return AttatchType{*this, std::forward<T>(observer)...};
    }
};
```
Now we can call: 
```cpp
auto subject = Subject().attach(Observer{});
subject.baseMethod();
subject.notify(1);
```
We can still access the base after we constructed the mixin in the attach method.

### CRTP

Let's say we now want to access the notify method from the base class, we want to write a generic method that notifies all observers if we receive a message. This way we can use the same method for different subject types . Example:
```cpp

class Base
{
    void GenericNotifyMethod()
    {
        auto data = this->GetData();
        this->notify(data);
    }

};

class MQTTSubscriber: public Base
{
    int Base.GetData();
};

class ShmemSubscriber: public Base
{

    int Base.GetData();

};
```
This is traditionally done with a virtual method where the AttachType would override it, or it is done statically through the use of CRTP. Since we set out on a goal to do everything at compile time, we will use try to use CRTP. The question now is, how do you combine Mixins and CRTP? Does it even work? Let's have a look at CRTP:

```cpp
template<typename Derived>
class Base
{};


class Derived: public Base<Derived>
{}
```
You can see the derived class inherits a base class which has the derived as a template parameter. No lets try to bolt on Mixins:

```cpp
template<typename Derived>
class CRTPBase
{};

template<template<typename> typename Base>
class Derived: public Base<Derived<???>>
{}
```
Derived takes a Base class as a template argument and inherits the templated Base class. But the Base class has to take the Derived class as a template argument. But the Derived class takes a base class as a template argument and so on and so on. This solution would result in an infinite loop of template arguments. 

The Mixin derived class takes the base class as a template argument, the CRTP base class takes the derived class as a template argument, this is a circular dependency.

### Deducing this to the rescue

C++ 23 introduces deducing this, which means the code could look as follows:

```cpp

class Base
{
    int GetData();

    template <typename Self>
    void notify(this Self&& self, int) {
        auto data = GetData();
        self.notify(data);
    }
};

template<typename Base>
class Derived: public Base
{
    notify(int);
}
```
I have yet to test out if this actually works the way I imagine. At the time of writing no compiler on compiler explorer supports deducing this yet.