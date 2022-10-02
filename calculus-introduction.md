---
layout: default
---

# Introducción

El $\lambda $-cálculo es una colección de sistemas formales basadas en una notación inventada por Alonzo Church en 1930. Se trata de operadores o funciones que pueden ser combinadas por otros operadores.

Por ejemplo, supon estas funciones.

$$f(x) = x - y$$
$$g(y) = x - y$$
$$f: x \to = x - y$$
$$g: y \to = x - y$$

Estas funciones se escriben de diferente forma, pero, en realidad, representan lo mismo. Se escriben diferente, pero el resultado es el mismo. Church introdujo $\lambda $ como símbolo auxiliar.

$$f = \lambda x.x - y$$
$$g = \lambda y.x - y$$

Por ejemplo, considera las siguientes ecuaciones.

$$f(0) = 0 - y$$
$$f(1) = 1 - y$$

En la $\lambda $-notación sería lo siguiente.

$$(\lambda x.x - y)(0) = 0 - y$$
$$(\lambda x.x - y)(1) = 1 - y$$

Estas ecuaciones son más torpes que las originales, sin embargo, la intención de la $\lambda $-notación denota funciones de alto nivel; no solamente funciones de números. Esto tiene una ayuda fundamental en la programación funcional.

<strong>Definición 1.1 ($\lambda $-términos)</strong> Asumiendo que dados una secuencia infinita de expresiones

$$v_0, v_{00}, v_{000},...$$

llamadas _variables_, y una secuencia finita, infinita o vacía de expresiones llamadas _constantes atómicas_ diferentes de las variables.

Cuando una secuencia de constantes atómicas es vacía, el sistema recibe el nombre de _puro_, en caso contrario, _aplicado_.
El conjunto de expresiones llamada $\lambda $-términos se define inductivamente como sigue.

* todas las variables y constantes atómicas son $\lambda $-términos, llamados átomos.
* si M y N son $\lambda $-términos, entonces (MN) es un $\lambda $-término, llamada aplicación.
* si M es cualquier $\lambda $-término y x es cualquier variable, entonces ($\lambda x.M$) es un $\lambda $-término, llamada abstracción.