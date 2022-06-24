---
layout: default
---

## Tipos y funciones

Voy a dejar comentarios de lo aprendido en Haskell, comparándolo con Swift. Adelantar, que la parte funcional de Swift está, prácticamente copiada de Haskell. Así que muchas cosas van a ser sencillamente idénticas.

De momento, usaremos la orden ghci, que viene a ser el terminal en haskell. Podemos usar operaciones como la suma, la multiplicación.

```haskell
2 + 2
```

```haskell
2 * 3
```

La división en Haskell no es entera
```haskell
3 / 2
# Ojo, es 1.5
```

```haskell
3 - 2
```

Los paréntesis en Haskell son importantes. La precedencia es la que es en todos los lenguajes de programación. Podemos seguirla o, bien, cambiarla. En Swift también podemos definir operadores y cambiar su precedencia o bien utilizar paréntesis.


```haskell
5 * 5 - 10
# 15
```

```haskell
(5 * 5) - 10
# 15
```

```haskell
5 * (5 - 10)
# -25
```

No estamos restringidos a usar números positivos. Pero tenemos que usar paréntesis.

```haskell
5 * -10
# Precedence parsing error
```

```haskell
5 * (-10)
```

Los tipos boleanos en Haskell empiezan por mayúsucla.

```haskell
True && False
```

```haskell
False || True
```

```haskell
not False
```

```haskell
not (False || True)
```

Si queremos comparar tipos, tenemos los operadores de igualdad y desigualdad. Pero, tienen que ser los mismos tipos.

```haskell
5 == 5
```

```haskell
1 /= 3
```

```haskell
5 == "Cinco"
# error, los tipos no coinciden
```

```haskell
5 == 4.5
```

En Haskell, el operador * es una función que toma dos parámetros y devuelve otro valor. En Swift es igual.

```swift
func multiply(a: T, b: T) -> T where T == Numeric
```

Pues en Haskel, es exáctamente igual. Algunas funciones en Haskell, se usan así.

```haskell
succ 10
```

La función succ devuelve la siguiente unidad, dada su entrada.

```haskell
succ 10.5
# Resultado 11.5
```

```haskell
min 9 10
```

La función min recibe dos parámetros y devuelve el menor valor de los dos.

```haskell
max 9 10
```

La composición de funciones es importante en Haskell, y la precedencia es importante.

```haskell
succ 10 + min 3 2 + 1
```

```haskell
(succ 10) + (min 3 2) + 1
```

Por último, existen dos formas de llamar a las funciones.

```haskell
div 90 10
```
```haskell
90 `div` 10
```