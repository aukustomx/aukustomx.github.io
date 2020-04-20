En Java, los objetos `String` son [inmutables](https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.3.3), esto es, una vez creados ya no es posible cambiarlos. 

La inmutabilidad de los `String`s en Java tiene ventajas, tales como la reutilización de cadenas dentro del pool de cadenas, seguridad, thread-safe, rendimiento, etc. Sin embargo, si no se conoce bien su funcionamiento, una mala implementación podría generar serios problemas de *performance*. 

En este post nos centraremos en la concatenación de cadenas dentro de un ciclo.

Por ejemplo, si creamos una cadena: `String cad1 = "Hola"` (primera instancia `String`) y deseamos concatenarla con la cadena `" mundo!"` (segunda instancia `String`), haciendo `cad1 = cad1 + " mundo!"` se crearía una tercera instancia (`"Hola mundo!"`).

![String concatenation creation]({{ site.url }}/img/string_concatenation_objects.jpg) 

En una aplicación donde sea necesario concatenar pocas cadenas y con bajísima frecuencia, el efecto negativo de performance es imperceptible. Pero en el caso de concatenar miles o millones de cadenas, aún más en un entorno concurrente, una mala implementación de concatenación de cadenas acarrea serios problemas de rendimiento. 

Algunos de estos problemas pueden ser, por ejemplo, demasiado trabajo al GC, altos tiempos de respuesta, utilización excesiva de memoria, cpu, entre otros.

## El operador `+` en los  `String`s

En Java, utilizar el operador `+` sobre `String`s para concatenarlas es equivalente a usar el método [`String::concat`](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/String.html#concat(java.lang.String\)). Revisemos lo que sucede en una concatenación con el operador `+`:

```java
public String concat(String str) {
    int otherLen = str.length();
    if (otherLen == 0) {
        return this;
    }
    int len = value.length;
    char buf[] = Arrays.copyOf(value, len + otherLen);
    str.getChars(buf, len);
    return new String(buf, true);
}

public static char[] copyOf(char[] original, int newLength) {
    char[] copy = new char[newLength];
    System.arraycopy(original, 0, copy, 0,
            Math.min(original.length, newLength));
    return copy;
}

void getChars(char dst[], int dstBegin) {
    System.arraycopy(value, 0, dst, dstBegin, value.length);
}
```

Al concatenar dos cadenas, **se crea un nuevo array del tamaño de la suma del tamaño de la cadena actual más el tamaño de la cadena a concatenar**. Se copian los valores de ambas cadenas al nuevo array, para finalmente **generar un nuevo objeto de tipo String** y retornarlo.

Como puede notarse, una concatenación de cadenas usando el operador `+` implica muchas operaciones. En notación *big O* su complejidad es de *O(n^2)*.

### Cómo no implementar concatenación de `String`s dentro de un ciclo.

Usando el operador `String` `+` en un ciclo, el código sería como sigue:

```java
/**
 * Cómo no concatenar cadenas dentro de un ciclo. Java 11
 */
public class BadConcatenation {

    public static void main(String[] args) {

        var repetitions = 100_000;
        var base = "Hello world";
        var result = "";

        for (int i = 0; i < repetitions; i++) {
            result = result + base; 
        }

        System.out.println("Result: " + result);
    }
}
```

Si revisamos el tiempo de ejecución que toma concatenar `100_000` veces la cadena `"Hello world!"`, el resultado es el siguiente:

```bash
$ java BadConcatenation.java
Time: 6067 ms
```

## La clase `StringBuilder`

La clase `StringBuilder` es utilizada para crear cadenas mutables (modificables). Es un símil de la clase `StringBuffer` excepto que no es sincronizada.
Para nuestro ejemplo no necesitamos recrear la cadena cada vez, solo ir agregando a la parte final de una sola cadena.

### Cómo sí implementar concatenación de `String`s dentro de un ciclo
Veamos ahora cómo implementar de forma correcta la concatenación de cadenas dentro de un ciclo:

```java
/**
 * Cómo sí concatenar cadenas dentro de un ciclo. Java 11
 */
public class GoodConcatenation {

    public static void main(String[] args) {

        var repetitions = 100_000;
        var base = "Hello world";
        var builder = new StringBuilder();

        for (int i = 0; i < repetitions; i++) {
            builder.append(base); 
        }

        System.out.println("Result: " + builder.toString());
    }
}
```

Si revisamos el tiempo de ejecución que toma concatenar `100_000` veces la cadena `"Hello world!"`, el resultado es el siguiente:

```bash
$ java GoodConcatenation.java
Time: 6 ms
```

## Comparativa de performance
Teóricamente, con los resultados de la dos implementaciones anteriores, podríamos decir que se pueden completar 1000+ concatenaciones con un `StringBuilder` cuando solo se puede completar 1 con el operador `+`. ¡Interesante, no!
En un ejercicio por comparar cuál es el tiempo de ejecución total que toma concatenar la cadena `"Hello world!"` usando el operador `+` vs. utilizar un `StringBuilder`, con un número de concatenaciones igual a `10_000`, `20_000`, `30_000`, `...`, hasta `1_000_000`, los resultados se reflejan en la siguiente gráfica:

![String concatenation creation]({{ site.url }}/img/string_concatenation_comparative.png)

Como puede observarse en la gráfica, utilizar el operador `+` al concatenar la cadena `"Hello world!"`,  incrementa el tiempo total como más implementaciones sean necesarias. Usando este operador, la prueba se detuvo cuando el número de concatenaciones solicitado era cercano a `500_000`; tomó casi 3.5 minutos.

Por otro lado, se observa que al utilizar un `StringBuilder` para esta tarea, el tiempo de respuesta máximo fue de 20 ms (0.0003333 minutos) incluso cuando hubo que concatenar la cadena `1_000_000` de veces.

Nota: Las instancias de `StringBuilder` no son thread-safe. Si se requiere sincronización, entonces se recomienda usar `StringBuffer`. 

## Conclusiones

Por todo lo anterior podemos concluir que, cuando exista la necesidad de realizar concatenación de `String`s dentro de ciclos, y ciclos con muchas iteraciones, hay que decidirse por `StringBuilder` en lugar del operador `+`.

Es importante mencionar que, cuando la concatenación no suceda dentro un ciclo, [es admisible usar el operador `+`](https://dzone.com/articles/string-concatenation-performacne-improvement-in-ja). Además, si los elementos que se concatenan son pocos, la legibilidad del código es mejor con el operador `+` que con un `StringBuilder`. Este es un ejemplo:

```java
//Más legible
var concatenated = "Hello" + " " + "world!";

//Menos legible y más verboso.
var concatenated = new StringBuilder().append("Hello").append(" ").append("world!"); 
```

Pero repito, **esto cuando no sea dentro de un ciclo**.

##Referencias
* https://docs.oracle.com/javase/specs/jls/se8/html/jls-4.html#jls-4.3.3
* https://dzone.com/articles/string-concatenation-performacne-improvement-in-ja
