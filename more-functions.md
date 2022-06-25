---
layout: default
---

## Más funciones

Sigamos con funciones. Crea un archivo multiply.hs y agrega lo siguiente.

```haskell
multiplyMe x = x * x
```

Ahora, abre el terminal, ponte en el mismo directorio del fichero creado y ejecuta el ghci. Usa la siguiente órden para cargar el archivo anterior.

```haskell
:l multiply
# Ok, one module loaded.
```
Ya puedes usar la función anterior.

```haskell
multiply 5
# 25
```

Podemos componer funciones.

```haskell
divideMe x y = x / y
```

```haskell
multiplyMe 5 + divideMe 10 2
```

```haskell
compositionMe x y z = multiplyMe x + divideMe y z
```

```haskell
compositionMe 5 10 2
```
Podríamos recurrir a condicionales if then else

```haskell
divideIfEventMultiplyIfOdd x = if even x then multiplyMe x else divideMe (x + 1) 2
```