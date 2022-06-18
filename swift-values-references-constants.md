---
layout: default
---

## Valores, variables, referencias y constantes

Un value o valor es inmutable y para siempre. Nunca va a cambiar. Por ejemplo, 1 de tipo Int o true de tipo Bool, son valores inmutables. También recibe el nombre de literales. Un literal es un valor inmutable que se genera directamente en el código.

En tiempo de ejecución (runtime), un valor puede ser generados. También será inmutable, para siempre y nunca cambiará.

Cuando asignamos un valor a una variable, estamos creando una variable usando un nombre que sostiene un valor.

```swift
var x = [1, 2]
```

Pero, cuando cambiamos el valor de x, por ejemplo, añadiendo un valor.

```swift
x.append(3)
```

No estamos cambiando el valor de x. En vez de eso, x se reemplaza por un nuevo valor.

```swift
x = [1, 2, 3]
```

También podemos declarar constantes. Una ves ua constante tiene un valor, ya nunca más lo podremos cambiar.

Podemos declarar una constante cuando queramos y luego asignarle un valor a la constante. En otras, palabras, una constante es una variable en el que solo puede ser asignado un valor una única vez.

```swift
let x: Int
```

Los structs y los enums son value types o tipo valor. Cuando asignas una variable struct a otra variable struct, las dos variables contienen el mismo valor. Esto es relevante, porque aquí no hay copia de objetos.

Una referencia es un tipo especial de valor. Es un valor que apunta a otro espacio en memoria. Aquí si que puede haber mutation o mutación de dos valores.

Las clases y los actores son tipos de referencias. Tu no puedes acceder directamente a una instancia de una clase. Para acceder a ese objeto, la variable contiene la referencia a ese objeto y accedes a través de ella.

Los tipos de referencia tienen identidad. Para validar que dos variables se refieren al mismo objeto usarás el operador ===. También puedes validar que los dos objetos son iguales con ==.

Nota. Dos objetos que no son iguales (==) puede ser el mismo objeto (===).

Los tipos valor no tienen identidad. No tiene sentido hacer ===.

Una referencia puede ser constante. Un valor también. Que la referencia sea constante, no implica que el objeto pueda cambiar.

Cuando un tipo valor es copiado, se crea una copia profunda (deep copy). La copia puede ser eagerly (si se introduce una nueva variable) o lazily (si la variable cambia).

Aquí tenemos un caso interesante. Si el struct contiene tipos de referencia, la copia será perezosa. Esto se llama shallow copies. Si el struct contiene valores, enttonces la copia será cuanto antes. La técnica se llama copy-on-write. Lo importante es saber que la copia no es automática.

En general si los elementos de una colección son referencias, se copiarán las referencias. Los objetos no se copian.

Algunas clases son totalmente inmutables. Por tanto, no se copiarán las referencias.

Solo las clases marcadas como final pueden ser garantía de no ser subclaseadas añadiendole la capacidad de ser mutables.

Las funciones en Swift son por valor. Las funciones pueden tomar otras funciones y transformar elementos. Map es una función de alto nivel. Las clausuras permiten devolver variables cuando pasas funciones por parámetro.

Un método es una función declarada dentro de una clase. Hay propiedades, stored properties y computed properties.



<p>Bibliografía y más información en objc.io aquí</p>

<a href="https://www.objc.io/books/advanced-swift/">Más info</a>

[Volver](./)