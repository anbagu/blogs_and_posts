# Metaclass `__prepare__()`
>`>>> A tangible example of metaclasses with Python`

![](https://media.istockphoto.com/photos/professional-male-potter-working-in-workshop-picture-id948451850?b=1&k=6&m=948451850&s=170667a&w=0&h=kodqDPAXk0n5CHyj9KlTQafD2-aylQIIBKXkM6uhd_8=)

### Why meta?

_"In Python everything is an object and every object is an instance of something we call a class, which in turn is an object and therefore an instance of something called a metaclass which in turn is an object.
In python, therefore, everything is an object."_
`"Someone once said."`

Let's check it out.
Taking the `int(1)` object we can evaluate that it is an instance of the `int` class.
```python
int(1)
# Output: 1
...
int(1).__class__
# Output: <class 'int'>
...
isinstance(int(1), int)
# Output: True
```

That is, `int` is the class of `int(1)`.
Let's iterate, with the same idea, over `int`. It's supposed to be an object.
```python
isinstance(int, object)
# Output: True
```
So, which class is it an instance of?

```python
type(int)
# Output: <class 'type'>
isinstance(int, type)
# Output: True
```

So it seems that `type`, apart from being a 'callable' that retrieves the type of an object, behaves like the class of something that is a class.

**That's right**, in the same way that `object` is the base class of any other class, `type` is the base metaclass of any other metaclass.

> **Alternative use of `type`_:**
> we can use `type` to dynamically create classes. A quick example:
> ```python 
> CustomInteger = type(
>    "Dummy",
>    (int,),
>    {"custom_to_str": lambda self, custom_msg: f"{custom_msg} {str(self)}"}
> )
> custom_int = CustomInteger(1)
> print(custom_int, CustomInteger)
> # Prints: 1 <class '__main__.Dummy'>
> print(custom_int.custom_to_str(custom_msg="this is my custom int ->"))
> # Prints this is my custom int -> 1
> ```
> Let's notice that `type` admits as parameters the name of the class, an n-tuple with the bases and a dictionary with the class attributes.
> In this case we have used it to dynamically define the `custom_to_str` method.

### Why I should learn meta?

Many times it is not easy to explain what real use cases metaprogramming with Python can have. Nor explain or justify its use in a project that does not require in-depth study.

Maybe the simple pleasure of 'playing' and learning more advanced concepts is enough reason to get interested.
But, if that were not enough, we are going to present a challenge and we will see
how to solve it from a metaprogramming approach.

### Goal

Let's imagine that in our project we represent certain logic through a class that has different methods that are executed with a certain flow order.
We may be interested in controlling/monitoring, in a certain way, what is happening. For example, log the invocation of each method. Or for example validate
that the types of the parameters of a function or method correspond to the annotations defined in the signatures. Or for example record the execution time of each method.
For this kind of problem, a pythonic approach is to use decorators for each method. Decorators are wrappers, and they can encapsulate the execution of one function inside another,
which generally modifies, registers, caches, etc...

> **_Statement_:** Suppose that our class involves a good number of methods, and we don't feel like decorating each of them.
> Let's try to define a **decoration policy** that affects all the methods of the class.
> 
> The idea is to avoid this,
> ```python
> class A:
>   @my_decorator1
>   @my_decorator2
>   def method_1(self): ...
>   ...
>   @my_decorator1
>   @my_decorator2
>   def method_n(self): ...
> ```
> and that somehow the body of the function is cleaner without repeating the decorators' declaration.

### Approach
> **_Disclaimer_:** this is not intended to be a complete python metaprogramming tutorial.
> Among other things, it is not a trivial question and its study requires patience and time to assimilate.
> _Based on my experience I have reached a certain point and I have created my own idea but I do not consider myself an expert in this area._

> _**For the interested reader**, I share what I believe is the best
> material to go into depth about it. This is the [Ionel blog](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/).
> Although he makes an introduction to metaclasses from scratch, he goes through and explains in quite detail the flow of the methods that participate in the creation, preparation, instantiation and so on,
> both at the class, object, and metaclass level. If anyone is interested, I invite you to play with the examples that he proposes since they allow you to follow the creation flows of objects, classes and metaclasses._
Next we are going to look at the diagrams that Ionel exposes since they are quite illustrative to **minimally explain the theory** behind the **construction flow**.


> **_Creation of an object_**
>
> When we instantiate an object the following happens:
> 1. `Metaclass.__call__`, triggers the instantiation process and retrieves the class object to execute `Class.__new__`.
> 2. `Class.__new__`, constructs an instance and returns it to `Metaclass.__call__` for initialization.
> 3. `Class.__init__`, initializes the instance with the instantiation parameters.
> 4. Once the object is constructed and initialized, the instantiation call is returned from `Metaclass.__call__`.
 
![alt text](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/instance-creation.png "Flow behind object instantiation")

Last pain in the ass before programming.
> **_Creation of a class_**
> 
> A class can be instantiated directly as we saw before with `type(NAME, BASES, ATTRIBUTES)`. Although generally
> we declare it with `class A:...`. If declared, Python takes care of its **instantiation**.
>
> In any case, the idea of instantiating a class is based on filling in attributes and methods defined or passed dynamically.
> This process follows this flow.
>
> 1. `M̀etaclass.__prepare__` is a special method. This is going to be the **cornerstone** for the solution that we are going to propose. It is a static method: it can be called from outside the metaclass (see diagram).
>    1. You must provide a **`DictLike`** object. This object is going to be the blank template to create the class from which to 'start doing things'. It is as if it were a **painter's blank canvas** or a **potter's ball of clay**.
>    2. It does not have to be an empty dictionary although by default it is.
>    3. You can customize `DictLike` methods like `__set_item__`, `__get_item__`, etc...
> 2. `Metametaclass.__call__` triggers the recovery of the metaclass.
> 3. `Metaclass.__new__` invokes the instantiation of the class-instance and takes into account the `DictLike` returned by `Metaclass.__prepare__`.
> 4. `Metaclass.__init__` is executed and parameters of the class-instance are initialized.
> 5. The class-instance is released from `Metametaclas.__call__`
>

> _**Note:**_ there is an analogy with the previous diagram, removing the `__prepare__` part.
 
![alt text](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/class-creation.png "Flow behind class instantation")

### Take a Break

If at this point it is confusing to assimilate the concepts, I recommend practicing with Ionel's examples.
### Resolution
Below we are going to address a possible solution to the problem, step by step.
#### Metaclass definition

Import this. We only need those external libraries.

```python
from functools import wraps
from time import sleep
from typing import Self, Generator

import pytest
```

We first define the `DictLike` class. Here is the key.
```python
class WrapStackPreparer(dict):

    def __init__(self, array_wraps):
        super().__init__()
        self.array_wraps = array_wraps

    def __setitem__(self, key, value):
        if callable(value):
            for wrap in self.array_wraps:
                value = wrap(value)
        dict.__setitem__(self, key, value)
```
Let's remember that we want to apply a list of wrappers.
1. In the `__init__` we expect a parameter `array_wraps` that contains each of the wrappers to apply.
2. In `__set_item__` we alter the default behavior of the `dict` class.
   1. This method will be executed every time a class attribute is registered or a method in the class is being 'prepared'.
   2. If it is a class attribute, nothing will happen. On the other hand, for the methods, which are `callable`, we will wrap them.
   3. Finally we invoke the `dict.__setitem__` that will register the already encapsulated method.

Now let's define the metaclass.
```python
class MetaWrap(type):

    @classmethod
    def __prepare__(metacls, name, bases, **kwargs):
        return WrapStackPreparer(**kwargs)

    def __new__(metacls, name, bases, classdict, **kwargs):
        result = type.__new__(metacls, name, bases, classdict)
        return result
```
Let us note that:
1. We set `type` as a base.
2. We define `__prepare__` as `classmethod` returning the `DictLike` and propagating `**kwargs` so that, as we will see later, we send the array of wraps.
3. We define a standard `__new__`, without customizing anything, to build the class.

### Use cases
#### Logger
We define a basic logging wrapper.
```python
def logged(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result

    return wrapper
```

Now we can create a class that uses it.

```python
class LoggedClass(metaclass=MetaWrap, array_wraps=[logged]):
    def method1(self):
        sleep(0.1)

    def method2(self):
        sleep(0.15)

    def main(self):
        self.method1()
        self.method2()
```
To use the metaclass we created we pass `metaclass=MetaWrap`. And to propagate the list of wraps so that it can reach the `__prepare__`
 we define the parameter `array_wraps=[logged]` as `kwarg`.

If we now execute,
```python
logged_cl = LoggedClass()
print("Calling directly")
logged_cl.method1()
logged_cl.method2()
print("Calling from main")
logged_cl.main()
```

we obtain as output,
```text
Calling directly
Calling method1
Finished method1
Calling method2
Finished method2
Calling from main
Calling main
Calling method1
Finished method1
Calling method2
Finished method2
Finished main
```
 and we check that it describes the execution flow as we expected.
 
#### Type validator

We now write a slightly more elaborate wrapper that takes into account the types declared in the signature of a method and encapsulates its execution 
to evaluate that the input and output of a method fulfill with the type annotations.
To do this, we inspect the `__annotations__` attribute, typical of any `callable`. The complete wrapper looks like this,
```python
def validate_annotations(func, validate_return=False):
    def validate(val, typ):
        if typ is Self:
            return

        if typ is None:
            typ = typ.__class__
        assert isinstance(val, typ), (
            f"Error {"invoking" if validate_return is False else "returning value from"} {func.__name__}"
            f"\n"
            f"Expected {typ} type, got {val} item of type {val.__class__.__name__}")

    @wraps(func)
    def wrapper(*args, **kwargs):

        annotations = func.__annotations__

        for k, val in kwargs.items():
            typ = annotations.get(k)
            validate(val, typ)

        for i, arg in enumerate(args):
            validate(arg, list(annotations.values())[i])

        result = func(*args, **kwargs)

        validate(result, validate_return=annotations.get("return") is not None)

        return result

    return wrapper

```

* We have defined in the body of the wrapper the `validate` function, which throws an exception when some annotation is not met.
* It also offers a minimal readable explanation of why the failure occurred.
* Then the wrapper encapsulates the call and validates both positional arguments, `key-value` arguments and the type of returned object.
> _**Note:**_ _This is a limited validator to exemplify, for a real use case we can go to the validator offered by `pydantic`, for example._

To test it, let's define the following class with annotated methods.

```python
class ValidatedClass(metaclass=MetaWrap, array_wraps=[validate_annotations]):

    def concat_str(self: Self, s1: str, s2: str) -> str:
        return s1 + s2

    def sum_int(self: Self, gen: Generator) -> int:
        return sum(gen)

    def wrong_return_type(self: Self) -> None:
        return 1
```
And now, if we execute the following calls,
```python
    validated_cl.concat_str("Hello", "World")
    validated_cl.sum_int(i ** 2 for i in range(10))
    with pytest.raises(AssertionError) as e:
        print("This should fail")
        validated_cl.wrong_return_type()
    print(e)

    with pytest.raises(AssertionError) as e:
        print("This should fail, also")
        validated_cl.concat_str("Hello", 1)
    print(e)
```
we get the following output.
```text
## Validate benchmark
This should fail
<ExceptionInfo AssertionError("Error returning value from wrong_return_type\nExpected <class 'NoneType'> type, got 1 item of type int") tblen=3>
This should fail, also
<ExceptionInfo AssertionError("Error invoking concat_str\nExpected <class 'str'> type, got 1 item of type int") tblen=3>
```


* As you can see above, the first two calls are successful, since the parameters values are aligned with the types annotated in the definition of the methods.
* On the other hand, the last two calls raise exceptions. In the first case due to the function being poorly defined with respect to the annotation. In the second case due to misuse in the call.

#### Putting all together

We can define a class where we require applying both wrappers.
```python
class LogValClass(metaclass=MetaWrap, array_wraps=[validate_annotations, logged]):
    def print_two_strings(self: Self, s1: str, s2: str) -> None:
        print(s1, s2)
```
And running the following,
```python
 logval_cl = LogValClass()
 logval_cl.print_two_strings("Hello", "World")
```
we get the output,
```text
Calling print_two_strings
Hello World
Finished print_two_strings
```

### Conclusion
That's all. As we have previously commented, we have proposed a solution to the problem from the perspective of metaclasses.
If you liked it, I invite you to investigate and study by yourself.

If you have any correction or suggestion you can reach out to [antoni.bayo@piperlab.es](mailto:antoni.bayo@piperlab.es) or [bayocostur@gmail.com](mailto:bayocostur@gmail.com).

### References
> Lastly I quote the two main sources that have helped me learn about the subject and invent this example and write the post
> * [1] [Ionel's blog](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses), as I said, seems to me to be the best resource to learn about metaclasses and deep dive into the subject.
> * [2] [The Python Morsels project](https://pythonmorsels.com/), in general, has helped me a lot to take a step further and do advanced things with python. Pedagogically it seems brutal to me. I highly recommend his blog and tutorials in which he explains fundamental concepts in detail and also offers very well thought out exercises so that you can break your brain and fully understand why things happen. The bad thing is that some content is paid, yes.

> From Python Morsels I have learnt what **Metaclass.__prepare__()** is and how it works so I'd like to congratulate [**Trey Hunner**](https://treyhunner.com/) for his work.


### Code

<details>
<summary>Complete code</summary>

```python
from functools import wraps
from time import sleep
from typing import Self, Generator

import pytest


class WrapStackPreparer(dict):

    def __init__(self, array_wraps):
        super().__init__()
        self.array_wraps = array_wraps

    def __setitem__(self, key, value):
        if callable(value):
            for wrap in self.array_wraps:
                value = wrap(value)
        dict.__setitem__(self, key, value)


# The metaclass
class MetaWrap(type):

    @classmethod
    def __prepare__(metacls, name, bases, **kwargs):
        return WrapStackPreparer(**kwargs)

    def __new__(metacls, name, bases, classdict, **kwargs):
        result = type.__new__(metacls, name, bases, classdict)
        return result


# decorator
def logged(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished {func.__name__}")
        return result

    return wrapper


def validate_annotations(func):
    def validate(val, typ, validate_return=False):
        if typ is Self:
            return

        if typ is None:
            typ = typ.__class__
        assert isinstance(val, typ), (
            f"Error {"invoking" if validate_return is False else "returning value from"} {func.__name__}"
            f"\n"
            f"Expected {typ} type, got {val} item of type {val.__class__.__name__}")

    @wraps(func)
    def wrapper(*args, **kwargs):

        annotations = func.__annotations__

        for k, val in kwargs.items():
            typ = annotations.get(k)
            validate(val, typ)

        for i, arg in enumerate(args):
            validate(arg, list(annotations.values())[i])

        result = func(*args, **kwargs)

        validate(result, annotations.get("return"), validate_return=True)

        return result

    return wrapper


class LoggedClass(metaclass=MetaWrap, array_wraps=[logged]):
    def method1(self):
        sleep(0.1)

    def method2(self):
        sleep(0.15)

    def main(self):
        self.method1()
        self.method2()


class ValidatedClass(metaclass=MetaWrap, array_wraps=[validate_annotations]):

    def concat_str(self: Self, s1: str, s2: str) -> str:
        return s1 + s2

    def sum_int(self: Self, gen: Generator) -> int:
        return sum(gen)

    def wrong_return_type(self: Self) -> None:
        return 1


class LogValClass(metaclass=MetaWrap, array_wraps=[validate_annotations, logged]):
    def print_two_strings(self: Self, s1: str, s2: str) -> None:
        print(s1, s2)


if __name__ == '__main__':
    print("\n\n", "## Logger benchmark", sep="")
    logged_cl = LoggedClass()
    print("\n", "### Calling directly", sep="")
    logged_cl.method1()
    logged_cl.method2()
    print("\n", "### Calling from main", sep="")
    logged_cl.main()
    ...
    print("\n\n", "## Validate benchmark", sep="")
    validated_cl = ValidatedClass()

    validated_cl.concat_str("Hello", "World")
    validated_cl.sum_int(i ** 2 for i in range(10))
    with pytest.raises(AssertionError) as e:
        print("This should fail")
        validated_cl.wrong_return_type()
    print(e)

    with pytest.raises(AssertionError) as e:
        print("This should fail, also")
        validated_cl.concat_str("Hello", 1)
    print(e)

    ...
    print("\n\n", "## LogVal benchmark", sep="")
    logval_cl = LogValClass()
    logval_cl.print_two_strings("Hello", "World")

```

</details>

