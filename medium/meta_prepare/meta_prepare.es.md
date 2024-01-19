# Metaclass `__prepare__()`
>`>>> Un ejemplo tangible de metaclases con Python`
>
![](https://media.istockphoto.com/photos/professional-male-potter-working-in-workshop-picture-id948451850?b=1&k=6&m=948451850&s=170667a&w=0&h=kodqDPAXk0n5CHyj9KlTQafD2-aylQIIBKXkM6uhd_8=)

### ¿Por qué meta?

_"En python todo es un objeto y todo objeto es una instancia de algo que llamamos clase, que a su vez es un objeto y por lo tanto instancia de algo que se llama metaclase que a su vez es un objeto.
En python, por lo tanto, todo es un objeto."_
`"Dijo alguna vez alguien."`

Vamos a comprobarlo.
Tomando el objeto `int(1)` podemos evaluar que es una instancia de la clase `int`.

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

Es decir, `int` es la clase de `int(1)`.
Iteremos, con la misma idea, sobre `int`. Se supone que es un objeto.
```python
isinstance(int, object)
# Output: True
```
Entonces, ¿de qué clase es instancia?

```python
type(int)
# Output: <class 'type'>
isinstance(int, type)
# Output: True
```

Entonces parece que `type`, a parte de ser un 'callable' para recuperar el tipo de un objeto, se comporta como la clase de algo que es una clase.

**Así es**, de la misma manera que `object` es la clase base de cualquier otra clase, `type` es la metaclase base de cualquier otra metaclase.

> **_Uso altenativo de `type`_:** 
> podemos usar `type` para crear dinámicamente clases. Un ejemplo rápido:
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
> Notemos como `type` admite como parámetros el nombre de la clase, una n-tupla con las bases y un diccionario con los atributos de clase.
> En este caso lo hemos utilizado para definir dinámicamente el método `custom_to_str`.

### ¿Para qué meta?

Muchas veces no es facil explicar qué casos de usos reales puede tener la metaprogramación con Python. Tampoco explicar ni justificar su empleo en un proyecto que no requiera profundizar.

Puede que el simple placer de 'jugar' y aprender conceptos más avanzados sea suficiente motivo como para interesarnos. Pero, por si eso no fuera suficiente, vamos a presentar un reto y veremos
como resolverlo desde la óptica de la metaprogramación.

### Objetivo

Imaginemos que en nuestro proyecto representamos cierta lógica mediante una clase que tiene diferentes métodos que con un cierto flujo van siendo ejecutados. 
Puede que nos interese controlar/monitorizar, en cierta manera, aquello que está pasando. Por ejemplo loggear la invocación de cada método. O por ejemplo validar
que los tipos de las variables que se transmiten se corresponden con los de las anotaciones definidas en las firmas. O por ejemplo registrar el tiempo de ejecución de cada método.
Para este tipo de problemas un enfoque pythonico es el de usar decoradores para cada método. Los decoradores son un 'wrap' y encapsulan la ejecución de una función dentro de otra,
que generalmente, modifica, registra, cachea, etc...

> **_Enunciado_:** Supongamos que en nuestra clase intervienen un buen número de métodos y no nos apetece decorar cada uno de ellos.
> Vamos a tratar de definir una **política de decoración** que afecte a todos los métodos de la clase.
> 
> La idea es evitar esto,
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
> y que de alguna manera el cuerpo de la función nos quede más limpio sin repetir la declaración de los decoradores.

### Enfoque
> **_Disclaimer_:** éste no pretende ser un tutorial completo de metaprogramación en python. 
> Entre otras cosas no es una questión trivial y su estudio requiere de paciencia y tiempo para asimilar.
> _En base a mi experiencia he llegado hasta cierto punto según y me he creado mi propia idea pero no me considero un experto._ 

> _Para el lector interesado, eso sí, comparto el que creo es el mejor
> material para profundizar al respecto. Se trata del [blog de Ionel](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/).
> Aunque hace una introducción desde 0 a las metaclases, va desgranando y explicando con bastante detalle el flujo de los métodos que participan en la creación, preparación, instanciación y demás, 
> tanto a nivel de clase, como de objeto, como de metaclase. Si alguien tiene interés invito a jugar con los ejemplos que el propone ya que permiten seguir los flujos de creación de objetos, clases y metaclases._

A continuación vamos fijarnos en los diagramas que Ionel expone ya que son bastante ilustrativos para **explicar mínimamente la teoría** que hay detrás del **flujo de construcción**.
> **_Creación de un objeto_**
> 
> Cuando instanciamos un objeto ocurre lo siguiente:
> 1. `Metaclass.__call__`, desencadena el proceso de instanciación y recupera el objeto clase para ejecutar `Class.__new__`.
> 2. `Class.__new__`, construye una instancia y la devuelve a `Metaclass.__call__` para que la inicialice.
> 3. `Class.__init__`, inicializa dicha instancia con los parámetros de instanciación.
> 4. Una vez construído e inicializado el objeto se devuelve la llamada de instanciacion desde `Metaclass.__call__`.
> 
>![alt text](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/instance-creation.png "Flow behind object instantiation")

Última chapa antes de programar.
> **_Creación de una clase_**
> 
> Una clase se puede instanciar directamente como vimos antes con `type(NAME, BASES, ATTRIBUTES)`. Aunque generalmente 
> la declaramos con `class A:...`. En caso de declararla Python se encarga de su **instanciación**.
> 
> En cualquier caso la  idea de instanciar una clase se basa en ir rellenando atributos y metodos definidos o pasados dinámicamente.
> Este proceso atiende el siguiente flujo.
>
> 1. `M̀etaclass.__prepare__` es un método especial. Esto va a ser la **piedra angular** para la solución que vamos a proponer. Es un método estático: puede ser invocado desde fuera de la metaclase (ver diagrama).
>    1. Debe proporcionar un objeto **`DictLike`**. Este objeto va a ser el molde en blanco para crear la clase a partir del cual 'empezar a hacer cosas'. Es como si fuera el **lienzo en blanco de un pintor** o la bola de barro del **alfarero**.
>    2. No tiene por que ser un diccionario vacío aunque generalmente lo es.
>    3. Se pueden personalizar métodos `DictLike` como `__set_item__`, `__get_item__`, etc ...
> 2. `Metametaclass.__call__` desencadena la recuperación de la metaclase.
> 3. `Metaclass.__new__` invoca la instanciación de la clase-instancia y tiene en cuenta el `DictLike` que retorna el `Metaclass.__prepare__`.
> 4. `Metaclass.__init__` es ejecutado y se inicializan parámetros de la clase-instancia.
> 5. Se libera la clase- instancia desde `Metametaclas.__call__`
> 
> _**Notemos como**_ _existe una analogía con el anterior diagrama, quitando la parte de `__prepare__`_ .
> 
> ![alt text](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses/class-creation.png "Flow behind class instantation")

### Break

Si llegados a este punto resulta confuso asimilar los conceptos recomiendo practicar con los ejemplos de Ionel.

### Resolución
A continuación vamos a abordar una posible solución al problema planteado, paso a paso.
#### Definición de la metaclase

Importa esto.

```python
from functools import wraps
from time import sleep
from typing import Self, Generator

import pytest
```

Definimos primero la clase `DictLike`. Aquí está la clave.
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
Recordemos que queremos aplicar una serie de wraps.
1. En el `__init__` esperamos un parametro `array_wraps` que contiene cada uno de los wraps a aplicar.
2. En `__set_item__` alteramos el comportamiento que tiene por defecto la clase `dict`.
   1. Este método se ejecutará cada vez que se registrá un atributo de clase o un método en la clase se esté 'preparando'.
   2. En caso de ser un atributo de clase no sucederá nada. En cambio para los métodos, que son `callable`, realizaremos el encapsulamiento de cada uno de los wraps.
   3. Finalmente invocamos el `dict.__setitem__` que registrara el método ya encapsulado.

Ahora definamos la metaclase.
```python
class MetaWrap(type):

    @classmethod
    def __prepare__(metacls, name, bases, **kwargs):
        return WrapStackPreparer(**kwargs)

    def __new__(metacls, name, bases, classdict, **kwargs):
        result = type.__new__(metacls, name, bases, classdict)
        return result
```
Notemos que:
1. Ponemos como base `type`.
2. Definimos `__prepare__` como `classmethod` devolviendo el `DictLike` y propagando `**kwargs` para que como más adelante veremos enviar el array de wraps.
3. Definimos un `__new__` standard, sin personalizar nada, para construir la clase.

### Casos de uso
#### Logger
Definimos un wraper de logging básico.
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

Ahora ya podemos crear una clase que lo use.

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
Para usar la metaclase que hemos creado pasamos `metaclass=MetaWrap`. Y para propagar y que le pueda llegar al `__prepare__`
el array de wraps definimos como `kwarg` el parámetro `array_wraps=[logged]`.

Si ahora ejecutamos,
```python
logged_cl = LoggedClass()
print("Calling directly")
logged_cl.method1()
logged_cl.method2()
print("Calling from main")
logged_cl.main()
```

obtenemos como output,
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
 y comprobamos que describe el flujo de ejecución según hemos definido las llamadas.
 
#### Validador de tipos

Escribimos ahora un wrap un poco más elaborado que tiene en cuenta los tipos declarados en la firma de un método proporcionados y encapsula su ejecución para evaluar que lo que entra y lo que sale cumpla las anotaciones
Para ello inspeccionamos el attributo `__annotations__`, propio de cualquier `callable`. El wrapper completo queda así,
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

* Hemos definido en el cuerpo del wrapper, la función `validate`, que lanza una excepción cuando no se cumple alguna anotación. Admás ofrece una minima explicación legible del por qué del fallo.
* Luego el wraper encapsula la llamada y valida tanto argumentos posicionales, como argumentos `key-value` o como el tipo del return.

> _**Nota:**_ _Se trata de un validador limitado para ejemplificar, para un caso de uso real podemos acudir al validador que ofrece `pydantic`, por ejemplo._

Para probarlo definamos la siguiente clase con métodos anotados.
```python
class ValidatedClass(metaclass=MetaWrap, array_wraps=[validate_annotations]):

    def concat_str(self: Self, s1: str, s2: str) -> str:
        return s1 + s2

    def sum_int(self: Self, gen: Generator) -> int:
        return sum(gen)

    def wrong_return_type(self: Self) -> None:
        return 1
```
Y ahora, si ejecutamos las siguientes llamadas,
````python
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
````
obtenemos el siguiente output.
```text
## Validate benchmark
This should fail
<ExceptionInfo AssertionError("Error returning value from wrong_return_type\nExpected <class 'NoneType'> type, got 1 item of type int") tblen=3>
This should fail, also
<ExceptionInfo AssertionError("Error invoking concat_str\nExpected <class 'str'> type, got 1 item of type int") tblen=3>
```


* Vemos que en las dos primeras llamadas obtenemos éxito, pues estamos haciendo un uso debido con respecto a lo que hey anotado en la definición de la clase.
* En cambio, en las dos últimas llamadas obtenemos excepciones, en el primer caso por estar mal definida la función con respecto a la anotación y en el segundo por hacer un mal uso en la llamada.

#### Todo a la vez

Podemos definir una clase donde exijamos aplicar ambos wrappers.
```python
class LogValClass(metaclass=MetaWrap, array_wraps=[validate_annotations, logged]):
    def print_two_strings(self: Self, s1: str, s2: str) -> None:
        print(s1, s2)
```
Y ejecutando lo siguiente,
```python
 logval_cl = LogValClass()
 logval_cl.print_two_strings("Hello", "World")
```
obtenemos el output, 
```text
Calling print_two_strings
Hello World
Finished print_two_strings
```

### Conclusión
Esto ha sido todo. Como hemos comentado previamente hemos dado una resolución al problema desde la perspectiva de la metaclases.
Si te ha  gustado, te invito a investigar y estudiar por tu cuenta.

Si tienes alguna corrección o una sugerencia puedes escribir a antoni.bayo@piperlab.es o bayocostur@gmail.com.

### Referencias
> Por último cito las dos principales fuentes que me han ayudado para aprender sobre el tema e inventarme este ejemplo y escribir el post
> * [El blog de ionel](https://blog.ionelmc.ro/2015/02/09/understanding-python-metaclasses), como he dicho me parece el mejor recurso para aprender de metaclases y empaparse.
> * [El proyecto de Python Morsels](https://pythonmorsels.com/), en general me ha ayudado mucho a dar un pasito más y hacer cosas avanzadas con python. Pedagógicamente me parece brutal. Recomiendo tanto su blog y tutoriales en los que explica con detalle conceptos fundamentales y además ofrece ejercicios muy bien pensados para que te rompas la cabeza y entiendas bien el por qué de las cosas. Lo malo es que cierto contenido es de pago, eso sí.


### Código

<details>
<summary>Código completo</summary>

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

