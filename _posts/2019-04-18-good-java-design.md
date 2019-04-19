# Immutability 
Has oído la frase *"haz que funcione, hazlo correcto y que funcione rápido, en ese orden"*. El autor de esta frase puede ser cualquiera, Kent beck o Butler Lampson's o algún otro, no importa. Lo que a nosotros nos interesa es la interpretación de la misma.

Como desarollador, y tal vez de forma inconsiente, cumples solo la primera parte de esta frase: *"haz que funcione"*, y las razones de que así sea pueden ser muchas: estás enfrentándote a un deadline muy próximo, mucha carga de trabajo, desconocimiento de la tecnología utilizada o incluso podrían ser malos hábitos. Esto hasta cierto punto puede entenderse, pero es cierto que tu responsabilidad es avanzar hacia las otras dos premisas conforme ganas experiencia. Si te haces el hábito de escribir buen código, siguiendo principios de diseño, te ahorrarás estrés, tiempo, dinero y quizá malos deseos de aquellos que mantengan tu código en el futuro.

Pero, ¿cómo escribir código Java que cumpla las premisas de la frase? Es este post veremos una de las muchas forma de lograrlo, la Inmutabilidad.

En OOP, la idea principal es abtraer un problema como objetos del mundo real. Los objetos son considerados first-class citizens, con sus propios atributos (propiedades) y métodos (comportamiento). Hay ciertos atributos de los objetos que deben mantenerse ocultos (encapsulamiento) y libres de cambios (invariants), y el mundo exterior no debe (a al menos no debería) conocer cómo son internamente. Cuando digo *libre de cambios* me refiero a que desde su creación y durante el tiempo de vida del objeto, nada ni nadie debe cambiar sus *invariants*, o sea no deben mutar.

En una aplicación Java, un objeto inmutable ofrece varias ventajas:
* Siempre será el mismo durante todo su ciclo de vida.
* Ningun *caller* podrá cambiar nada del objeto.
* Facilita el mantenimiento de los programas.
* Mejora el rendimiento limitando el número de copias del objeto, como es el caso de los `String`s.
* Facilita el debugging ante posibles problemas.
* Previene side-effects.

## ¿Cómo logramos crear objetos inmutables?
Analicemos el siguiente código:

```java
class PersonInfo {
    public String name;
    public LocalDateTime birthday;
}
```

Digamos que en algún punto de la aplicación se crea un objeto de tipo Person. Los atributos del objeto son configurados de la siguiente manera:
```java
    PersonInfo personInfo = new PersonInfo();
    personInfo.name = "Bob"
    personInfo.birthday = LocalDateTime.of(2000, )
```

Este parece ser un inofensivo programa, pero tiene varias deventajas a la vez que corre varios peligros. Supongamos que este objeto es pasado a otro para su procesamiento, por ejemplo, PersonLogging:
```java
    ...
    void logPersonInfo(PersonInfo personInfo) {
	recordPersonInfo(personInfo.name);
	recordPersonInfo(personInfo.birthday);

        //Aprovechamos para poner el sufijo _logged al nombre de la persona
        personInfo.name = personInfo.name + "_logged";
    }
```

Pero, pero, pero, espera. ¿cómo que aprovechamos para cambiar el nombre? El nombre en la información de una persona es algo que no debería variar. Sin embargo aquí ha ocurrido y las *invariants* han variado. Hasta este momento, los objetos de la clase `Person` son vulnerables ante sus *callers*. Sus atributos están expuestos de tal manera que cualquier otro objeto puede manipularlos arbitrariamente. ¿Querrías tú que alguien cambiara tu nombre así porque sí?

Java soporta la inmutabilidad, pero no forza a su utilización. Nosotros al escribir nuestros programas sí podríamos hacerlo. En lo máximo posible debes diseñar tus programas para utilizar objetos inmutables. Joshua Bloch lo aconseja en su libro *Effective Java: "Trata los objetos como inmutables"*.

Para lograr que los atributos de los objetos tipo PersonInfo no cambien durante su ciclo de vida debemos seguir varios pasos:
* Cambiar el modificador de acceso de public a private. De hecho se considera definir los atributos y métodos de una clase como privados, e irlos exponiendo solo si es necesario.
* No exponer ningún método *mutator*
* Exponer los atributos, si se necesita, mediante métodos *accesors*
* Declarar los atributos `como final`.
* Inicializar los atributos con ayuda de Constructores.
* Especial atención a colecciones y parámetros de constructores.

## Versión inmutable de Person


```java
class PersonInfo {
    private final String name;
    private final LocalDateTime birthday;

    PersonalInfo(String name, LocalDateTime birthday) {
        this.name = name;
        this.birthday = birthday;
    }

    public String getName() {
        return name;
    }

    //Solo queremos publicar el día y el mes de nacimiento,
    // no el año y en forma de string "mm/dd"
    public String getBirthday() {
        return birthday.getMonthValue() + "/" + birthday.getDayOfMonth();
    }
}
```
Al marcar como `final` los atributos de la clase, el compilador ayudará a hacer notar que dichos valores no pueden ser modificados una vez que se han definido. El modificador de acceso `private` asegura que los atributos solo son alcanzables por otros elementos dentro de la clase. El constructor ayuda a inicializar los valores de los atributos y el hecho de no tener *mutators* garantiza que los objetos no serán modificados desde el exterior. 

No existe ninguna regla que indique que se deben regresar objetos del mismo tipo que sus atributos. Así que en este caso, ocultamos (encapsulamos) del exterior de que el birthday de una persona es un LocalDateTime y solo exponemos el dato de mes/año como un `String`.

## Colecciones y parámetros en constructores
Las colecciones y los parámetros en los constructores son casos que requieren especial atención y cuidado al implementar objetos inmutables.
Supongamos que implementamos la siguiente clase como inmutable:
```java
class Student {
    private final String name;
    private final List<Course> courses;

    //Constructor initialization

    public List<Course> getCourses() {
        return courses; 
    }
}
```
¿Es realmente inmutable esta clase? Podrías confundirte y asegurar que `courses` es `final` y por lo tanto la lista de cursos a los que un estudiante está inscrito nadie puede modificarla, y que se inicializó en el construtor. Además no hay ningún *mutator* definido. La clase no es completamente inmutable. La razón es que lo que es `final` es la referencia a la lista, de tal manera que courses no podrá apuntar a otra colección. Sin embargo, el contenido de la colección ha quedado expuesto, como se ve en el siguiente código de un *caller*:

```java
   
    //Aquí colocamos una materia extraescolar del alumno
    student.getCourses().add(new Course("Artes marciales"));
   
```

Si fueras tú el estudiante en cuestión, al final del semestre recibirías una nota reprobatoria de un curso que tú ni siquiera registraste. O peor aun, el caller decide eliminar tus cursos con `student.getCourses().clear()`. No apareces en las listas de los profesores.

Para evitar esto, utiliza colecciones inmutables o devuelve una *copia defensiva* de la colección. 

  
## Paralelización
Desde la versión 8 de Java, cuentas ya con un estilo funcional de programación a través del API de Stream y las expresiones Lambda. La inmutabilidad es uno de los puntos escenciales de la programación funcional, donde el objetivo no es la mutación de los objetos sino la transformación de estos.


## Conclusión
El código con muchos cambios en variables es dificil de entender, facil de corromper, difícil de depurar y problemático al paralelizar. 
La inmutabilidad elimina estos problemas de raíz. 
