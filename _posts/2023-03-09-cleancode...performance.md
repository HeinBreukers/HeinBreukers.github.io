---
layout: post
title: "Clean Code ... Performance"
subtitle: "In this post i will compare the performance of std::variant to its alternatives"
date: 2023-09-03 00:00:00 -0000
---

## Casey's case 

A while ago Casey Muratori posted the quite controversial post "Clean" Code, Horrible Performance. Here he explains that certain clean code best practices have a negative effect on performance. 
He looks at a piece of code that calculates the total area of different shapes. The "clean code" example implements this using virtual functions. To speed things up Casey firstly replaces the virtual function calls with an enum and a switch statement. The advantage of this method is that the different objects can be placed contiguously in an array by value, whereas in the virtual case they had to be stored as pointers. This results in less indirection and more cache friendly code which resulted in a 1.5x speed increase for his test. 
Secondly he realises that the area function of all objects follow the following signature:
$$ Area = ShapeConstant*Width*Height$$ This means that the switch can be replace by a single function call. Getting rid of the switch reduces the total number of instructions and also improves instruction cache locality. Switch statements are often implemented as jump tables, the program jumps to different locations depending on the argument, in this case the enum. This change results in a 10x speedup.

## The need for polymorphism remains

Casey's second speedup is not always possible since the function signature might be completely different for different objects. It is however a good idea to investigate if a common function signature exists before you start coding.

If the signature is different enough, some form of polymorphism is still very useful. Casey's preferred form is an enum with a switch statement. Purely usings switch statements can however get messy, for example if the layout of the objects is different. 

Different objects require different variables to get to the solution. To make sure the objects still can be stored in the same array you store them as a union of all the supported object types. Which means the switch statement would have to look like:
```cpp
int calc(Obj obj) 
{
    switch(obj.Type)
    {
        case Type::type1: return obj.type1.a;
        break;
        case Type::type2: return obj.type2.b; 
        break;
        case Type::type3: return obj.type3.a+obj.type3.c;
        break;
    }
}
```
or if the functions are encapsulated in member functions
```cpp
int calc(Obj obj) 
{
    switch(obj.Type)
    {
        case Type::type1: return obj.type1.calc();
        break;
        case Type::type2: return obj.type2.calc(); 
        break;
        case Type::type3: return obj.type3.calc();
        break;
    }
}
```
C++17 gave us std::variant which offers a more elegant and safe way to write a switch like this. 
```cpp
int calc(Variant obj) 
{
    return std::visit([](auto&& arg)
        {
            return calc();
        },obj);
}
```

But does std::variant with std::visist have any performance penalty? 

In today's post I want to compare std::variant with std::visit to switch cases, switch cases with member function calls and virtual function calls to see if we can still have polymorphism without its performance costs described by Casey.

## The test setup

For the test case I want to iteratively calculate either the sum, min or max of floating-point numbers. Let's say a user can define what operations they want to do on a set of input numbers. These operations are different enough so they do not have the same function signature and thus cannot be replace by a single function. We could in the future extend the operations with mean, standard deviation etc. 

The full implementation of the benchmark is available [here](https://github.com/HeinBreukers/CleanCode...Performance).


### Benchmark

The general benchmark looks as follows:
```cpp
static void Benchmark(benchmark::State& state) {
  // generate vector of operatoins
  auto ops = GenOps(size);
  // generate vector of random data
  auto data = GenData(size);
  for (auto _ : state)
  {
    float out = 0;
    for(size_t i=0; i<size;++i)
    {        
        out += ops[i](data[i]);
    }
    benchmark::DoNotOptimize(out);

  }
}
```
We create a vector of operations, and a vector of random data. Then during the benchmark we do the operation on the random input.

### Results

Since my machine is not properly setup for benchmarks, I will use quickbench to benchmark the alternatives. All benchmarks are compiled with "std = c++20", "optim = O3", "STL = libstdc++(GNU)" and a vector size of 1000. 
The results for three different compiler versions are:

####Clang 15.0:

![alt text](https://raw.githubusercontent.com/HeinBreukers/HeinBreukers.github.io/master/_posts/images/2023-03-09-cleancode...performance/clang150.png)

####GCC 12.2:

![alt text](https://raw.githubusercontent.com/HeinBreukers/HeinBreukers.github.io/master/_posts/images/2023-03-09-cleancode...performance/gcc122.png)

####GCC 9.5:

![alt text](https://raw.githubusercontent.com/HeinBreukers/HeinBreukers.github.io/master/_posts/images/2023-03-09-cleancode...performance/gcc95.png)

For Clang 15.0 and GCC 12.2 the virtual method alternative is the clear loser. For GCC 9.5 the variant performs similarly to the virtual implementation, why is discussed [here](https://www.reddit.com/r/cpp/comments/kst2pu/with_stdvariant_you_choose_either_performance_or/). 

For GCC 12.2 there are some small differences between the Variant, Union and CLike alternatives. The differences are small enough and not consistent enough that they can be attributed to chance. 
