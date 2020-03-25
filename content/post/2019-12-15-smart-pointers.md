---
title: Ref Counted Objects and C++ Smart Pointers
date: 2019-12-10
---

In this article we will look into implementing a simple smart pointer class
with reference counting support for tracking the lifetime of a pointer. First
we will try to understand the primary models of memory management in most modern
programming languages.

<!--more-->

One of the primary considerations when picking a language is memory management
and there are primarily two main models. The first one is
the garbage collection(GC) model and the second one is the automatic reference
counting model(ARC). Any new modern programming language these days
adopts either of these two approaches for memory management. With these models, the developer is
freed the burden of tracking and freeing memory. One might also argue that the
applications can be less error prone as a result. Both the approaches have their
own pros and cons however. Lets dig a little deeper into these two models.

#### Garbage Collection(GC)

In the GC model, the application process typically runs a garbage collector thread
which runs through the object graph and tries to clean up the objects which are
no longer active(or useful) in the context of the user program execution. At at
high level this is a two step process, in the first step the thread runs through
the objects and marks the objects which are safe to delete, the same thread as part
of step 2, runs at a later(non deterministic) time and frees up the objects
marked as safe to delete in step 1.

The primary *shortcomings* of the GC model are that the object destructor calls are
now non deterministic. Secondly, the GC thread can slow down the application program, more
so in cases where the program requires a lot of memory objects and the process
or the system is running into a low memory conditions.

#### Automatic Reference Counting (ARC)
In this model ARC unlike the GC model there is no
separate thread of execution, but the logic of object clean up is embedded as
part of the language constructs. The objects are automatically reference counted
as part of object creation, copying/assignment operations and when the reference
count hits zero, the object is cleaned up. The compiler adds code to manage the
reference counting, relieving the developer of the duties. So in this case
calling the object destructors is now deterministic and unlike earlier model
there is no separate thread of execution.

The primary *shortcoming* of this model however is that it does not handle
weak references or weak pointers very well.

Now let us look at a simple smart pointer class definition which will help us
realize and understand how an ARC model works.

{{< highlight cpp "linenos=inline">}}

template <typename T> class SmartPointer {
private:
        T* mData;
        int* mCount;

public:
        SmartPointer(): mData(nullptr), mCount(nullptr) {
            mCount = new int();
            *mCount = 1;
        }

        SmartPointer(T* data): mData(data), mCount(nullptr) {
            mCount = new int();
            *mCount = 1;
        }

        SmartPointer(const SmartPointer<T> &sp):
                mData(sp.mData), mCount(sp.mCount) { ++*mCount; }

        ~SmartPointer() {
            if (!(--*mCount)) {
                delete mData;
                delete mCount;
            }
        }

        T operator-> () { return mData; }

        T& operator* () { return *mData; }

        SmartPointer<T>& operator=(const SmartPointer<T>& sp) {
            if (this == &sp)
                return *this;

            if (!(--*mCount)) {
                delete mData;
                delete mCount;
            }

            mData = sp.mData;
            mCount = sp.mCount;
            ++*mCount;

            return *this;
        }
};
{{< / highlight >}}

Let me try to explain this code. The template class SmartPointer is a wrapper for the
underlying pointer type. This class also hosts a reference count which tracks
the life cycle of the pointer. The mCount variable handles the reference count
of the pointer.

When a user plans to manage the pointers the smart way, they simply would have to
allocate the underlying object which needs tracking and then create an object of SmartPointer
type feeding it the pointer from the prior memory allocation. Now, as part of the
constructor for the SmartPointer class, a new reference count is instantiated and
initialized with value 1. Now for all intents and purposes this reference count is
tracking the life of this pointer. When copying or assigning pointers the
reference count is incremented, and when the object goes out of scope, the
reference count is decremented, and when the object goes out of scope, the
reference count is decremented. When the reference count hits 0, the underlying
object and the associated reference object are both freed up.

The major complexity of reference management comes in handling the copy
constructor and the overloaded assignment operator

{{< highlight cpp "linenos=inline">}}
        SmartPointer(const SmartPointer<T> &sp):
                mData(sp.mData), mCount(sp.mCount) { ++*mCount; }

        // example: The below statement calls the copy constructor
        SmartPointer<Employee> emp2 = emp1;
{{< / highlight >}}

This code snippet show the code for the copy constructor of the SmartPointer
class. When the copy constructor is called, the reference count pointer from incoming SmartPointer
object is copied over and then incremented as both emp1 and emp2 are now
tracking the same underlying object.

{{< highlight cpp "linenos=inline">}}
        SmartPointer<T>& operator=(const SmartPointer<T>& sp) {
            if (this == &sp)
                return *this;

            if (!(--*mCount)) {
                delete mData;
                delete mCount;
            }

            mData = sp.mData;
            mCount = sp.mCount;
            ++*mCount;

            return *this;
        }

        //example invocation of the overloaded assignment operator
        SmartPointer<Employee> emp3;
        emp3 = emp1;

{{< / highlight >}}

Assignment operator overload is slightly more involved than the copy constructor.
Here we need to first check if the incoming object is the same object as the
current object being operated upon. If they are not the same, then we need to
first reduce the reference count of the lhs object(emp3 in the example above) and if the
reference count hits zero, then delete the memory and the reference count of the
lhs object. Then
we copy over the pointer and the reference from the incoming rhs SmartPointer object(emp1)
and then increment the reference count. Note even in this case, both emp3 and
emp1 end up tracking the same underlying memory object.

{{< highlight cpp "linenos=inline">}}
struct Employee {
    std::string name;
    int age;
    std::string designation;

public:
    Employee(): name(""), age(0), designation("") {};

    Employee(std::string name, int age, std::string designation):
        name(name), age(age), designation(designation) {};
};

int main() {
    SmartPointer<Employee> emp1(new Employee("Alien1", 251,
                                    "Space Ship Janitor"));

    {
        // The below statement calls the copy constructor
        //ref count of emp1 is incremented to 2
        SmartPointer<Employee> emp2 = emp1;
        // Do something here with emp2

        // The below statement calls the overloaded assignment operator
        // ref count of emp3 is set to 0 and emp1 is incremented to 3
        SmartPointer<Employee> emp3;
        emp3 = emp1;
    } // emp2, emp3 destructor will be called here and ref count
      //of emp1 goes back to 1

    //emp1 destructor will be called herer
    //ref count goes to emp1 hits 0 and the underlying Employee
    //pointer is freed up
}
{{< / highlight >}}

Here is a simple example of using the SmartPointer objects for tracking the
pointers lifetime. SmartPointer emp1 is instantiated with an object of type
Employee allocated on the heap(new Employee() returns a pointer). This pointer is now
managed by the SmartPointer class with reference counting.

{{< highlight cpp "linenos=inline">}}
    SmartPointer<Employee> emp1(new Employee("Alien1", 251,
                                    "Space Ship Janitor"));
{{< / highlight >}}

I have added comments to explain the reference counting of the underlying
pointer object and when it is freed up.
