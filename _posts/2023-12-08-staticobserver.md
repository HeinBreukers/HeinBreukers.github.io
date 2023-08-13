---
layout: post
title: "Static Observer"
subtitle: "In this post we will look at how to implement a static version of the Observer pattern"
date: 2023-08-12 00:00:00 -0000
---

# Static Observer

Recently I have been working on a streaming API. The user is able to define a set of streams, each with their own functionality and can link the streams together. 
An example could be as follows. 

Source -> PreprocessingStream -> ProcessingStream -> Sink

<!-- 
```flow
st=>start: Soure
op=>operation: PreProcessing Stream
op2=>operation: Processing Stream
e=>end: Sink

st->op->op2->e
``` -->
The observer pattern provides a mechanism to link streams together. Each stream could then be both an observer, a subject or both. A stream could observe the previous stream, then do its own task, and then notify all the streams that are regestered to it. 
The traditional observer pattern relies dynamic polymorphism, however I already know what stream structure looks like at compile time, so i want to implement a static version of the observer pattern.

Firstly let us have a look at the dynamic version:
```cpp
class Observer
{ 
public:
    virtual void update() const = 0;
};

// Subject is the base class for event generation
class Subject
{
public:
    void notify()
    {
        for (const auto& observer: observers) 
        {
            observer.update();
        }
    }
  
    // Add an observer
    void attach(Observer* observer) 
    {
        observers.push_back(&observer);
    }
private:
    std::vector<Observer*> observers;
};

// Example of usage
class ConcreteObserver: public Observer
{
public:
  
    // Get notification
    void update() const override
    {
        std::cout << "Got a notification" << std::endl;
    }
};

int main() 
{
    Subject subject;
    ConcreteObserver concreteObserver;
    subject.attach(&concreteObserver);
    subject.notify();
}
```

For the static version we have to find a way to replace the vector of base pointers with something that can store different types statically.

Firstly lets have a look at how we are going to implement the Subject class. 

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

As you can see the Subject type itself does not handle the observers anymore. Instead it creates a new object 'AttachType' when a set of observers is attatched to the subject. 
The attach method takes a variadic number of observers and makes a tuple of them, this tuple is then passed to the AttachType object where it is stored.
The AttachType object can then notify all observers with the notify method. It can do this as follows:

```cpp
template<typename... Observers>
void updateSet(int in, T&&... set)
{
    (set.update(in),...);
}


template<typename Tuple>
struct AttatchType
{
    void notify(int in)
    {
        std::apply([&in](auto&... args){ updateSet(in, args...); }, m_observer);
    }

    Tuple m_observer;
};
```
UpdateSet takes the input argument to the update function (in this case a single int) and the set of observers. It then does a comma fold expression, which will result in the update method of each observer being called.

To call this updateSet method we will use std::apply in combination with a lambda to pass the input argument of the update funciton.

The end result can then look as follows:
```cpp
struct ConcreteObserver
{
    void update(int in)
    {
        std::cout<<in<<'\n';
    }
};

struct ConcreteObserver2
{
    void update(int in)
    {
        std::cout<<in*2<<'\n';
    }
};

int main()
{
    ConcreteObserver concrete1;
    ConcreteObserver2 concrete2;
    auto subject = Subject().attach(concrete1, concrete2);
    subject.notify(1);
    return 0;
}
```

The next steps would be to make the input type of the update (in this case int) generic, and check them during the attach. Concepts have to be implemented and we have to determine if we want to store values or pointers in the tuple.
Also often in the observer pattern, the subject is passed to the observer during construction where the observer is then attached to the subject. How to implement this i will leave for a future post.