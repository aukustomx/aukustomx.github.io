# Immutability 
Has oído la frase *"haz que funcione, hazlo correcto y que funcione rápido, en ese orden"*. ¿Quién la dijo? Kent beck o Butler Lampson's o algún otro, no importa. Lo que nos interesa es el fondo de la misma.

Como desarollador, y tal vez de forma inconsiente, podríamos estar cumpliendo solo la primera parte de esta frase: *"haz que funcione"*, y las razones pueden ser muchas: un deadline muy próximo, carga de trabajo, desconocimiento de la tecnología utilizada o incluso malos hábitos. Comprensible. No obstate, mi responsabilidad y la tuya es avanzar hacia las otras dos premisas. Si te haces el hábito de escribir código que sigue principios de diseño, te ahorrarás estrés, tiempo, dinero y quizá uno que otro mal deseo de quienes mantengan tu código en el futuro.

Pero, ¿cómo escribir código que cumpla esta frase? Es este post veremos una de las muchas prácticas para lograrlo, la **Inmutabilidad**.

En OOP, la idea principal es abtraer un problema como si de objetos del mundo real se tratase. En OOP, los objetos son considerados first-class citizens, con sus propios atributos (propiedades) y métodos (comportamiento). Pero hay ciertos atributos de los objetos que deben mantenerse ocultos (encapsulamiento) y libres de cambios (invariants), y el mundo exterior no debe (o al menos no debería) conocer cómo son internamente. Cuando digo *libre de cambios* me refiero a que desde su creación y durante el tiempo de vida del objeto, estos *invariants* nada ni nadie debe cambiarlos, o sea, no deben mutar.

## ¿Cómo logramos crear objetos inmutables?
Analicemos el siguiente código:

```java
class PersonInfo {
    public String name;
    public LocalDateTime birthday;
}
```

Y digamos que en algún punto de la aplicación un cliente o *caller* crea un objeto de tipo `PersonInfo` de la siguiente manera:
```java
class PersonService {
    ...
    void registrationProcess() {
        PersonInfo personInfo = new PersonInfo();
        personInfo.name = "Bob"
        personInfo.birthday = LocalDateTime.of(2000, 1, 1, 0, 0);

    //...more code
```

Este parece ser un inocente e inofensivo programa, pero tiene varias desventajas a la vez que corre varios peligros. Joshua Bloch recomienda *minimizar el acceso de clases y miembros de clase* en su libro *Effective Java*, algo que la clase `PersonInfo` claramente no hace. Ocultar los detalles de la implementación de nuestro componente nos traería solo beneficios. *"Un componente bien diseñado oculta (encapsula) todos los detalles de su implementación y los componentes se comunican entre sí solo a través de sus APIs"*

Para demostrar las debilidades y peligros que corren los objetos de la clase `PersonInfo` vamos a suponer que nuestro recién creado objeto es pasado a otro para procesar, por ejemplo, un registro en una bitácora:
```java
class PersonService {

    LoggingService loggingService;

    void registrationProcess() {
        ...
        //una vez registrado, guardamos operación en bitácora
        loggingService.logRegistration(personInfo);
    }


class LoggingService {
     
    void logRegistration(PersonInfo personInfo) {
	recordPersonInfo(personInfo.name);
	recordPersonInfo(personInfo.birthday);

        //Aprovechamos para poner el sufijo _logged para marcar
        // a la persona como procesada
        personInfo.name = personInfo.name + "_logged";
    }
```

A ver a ver, un momento, ¿cómo que: aprovechamos para poner el sufijo...? El nombre en un objeto `PersonInfo` no debería cambiar. Sin embargo aquí ha ocurrido y sus *invariants*, pues... han variado. `PersonInfo` es un código endeble y vulnerable ante sus *callers*. Sus atributos están expuestos de tal manera que cualquier otro objeto puede manipularlos arbitrariamente. ¿Querrías tú que alguien cambiara tu nombre así porque sí?

Podemos remediar el problema con encapsulamiento y mejor aún implementando **inmutabilidad**. En este sentido, otras dos de las muchas recomendaciones de Joshua Bloch son: *"en clases públicas, utiliza métodos accesors, no campos públicos"* y *"Minimiza la mutabilidad"*. Si bien Java soporta inmutabilidad en los objetos, no nos forza a su utilización. Nosotros al escribir nuestros programas sí podríamos hacerlo, y veremos cómo conseguirlo. Vamos por partes, primero...

## ¿Qué es una clase inmutable?
Una clase inmutables es simplemente aquella cuyas instancias no pueden ser modificadas, una vez que su información ha sido definida. No habrá ninguna modificación a la misma durante el ciclo de vida de dichos objetos. En Java existen estos varios ejemplos de clases inmutables, tal es el caso de los `String`s.

## ¿Cómo diseñar una clase inmutable?
Para lograr que los atributos de los objetos tipo `PersonInfo` no cambien durante su ciclo de vida debemos seguir varios pasos:
* Cambiar el modificador de acceso de `public` a `private`. Recuerda, minimiza el acceso.
* Declarar los atributos como `final`.
* No exponer ningún *mutator*.
* Exponer los atributos, si se necesita, solo con *accesors*.
* Inicializar los atributos con ayuda de Constructores.
* Poner especial atención a colecciones y parámetros de constructores.

## Versión inmutable de Person
```java
final class PersonInfo {
    private final String name;
    private final LocalDateTime birthday;

    PersonInfo(String name, LocalDateTime birthday) {
        this.name = name;
        this.birthday = birthday;
    }

    public String getName() {
        return name;
    }

    //Solo queremos publicar el día y el mes de nacimiento,
    // en la forma "mm/dd", sin el año.
    public String getBirthday() {
        return birthday.getMonthValue() + "/" + birthday.getDayOfMonth();
    }
}
```
Analicemos lo que hicimos: 
Cuando se cambia el modificador de acceso de `public` a `private`, se proteje el estado interno de los objetos. Los *callers* no saben el detalle de la implementación. 

Al marcar como `final` a los atributos de una clase, se asegura que los atributos del objeto no pueden ser modificados una vez que se han definido. En el caso de la clase `PersonInfo`, un constructor ayuda a inicializar los valores de sus atributos. 

**No implementes métodos que modifiquen el estado del objeto**. Como se puede ver, ahora la clase `PersonInfo` no tiene *mutators*. Con este ajuste, quien decide qué si y qué no puede mutar, es el objeto en sí mismo.

Si fuera necesario, la clase define qué información se da a conocer usando *accesors*. Es en la clase que se establece cómo los *callers* interactúan con las instancia a través de su *API*.

Si lo notaste, el *accesor* `getBirthday()` devuelve un `String` y no un `LocalDateTime`. No existe ninguna regla que indique que se deben regresar objetos del mismo tipo que sus atributos. Así que en este caso, encapsulamos el birthday de una persona, exponiendo el dato de mes/año como un `String`.

## Colecciones y parámetros en constructores
Si la clase que estamos definiendo tiene referencias a objetos mutables (`Collection`s, `StringBuilder`s, por ejemplo) que fueron recibidos como parámetros en constructores, tienes que asegurarte que tienes **acceso exclusivo** a estos componentes, es decir, el cliente que construye la instancia no debe poder modificar dichos objetos. ¿Cómo lograrlo? No inicialices un campo con referencias a objetos provistas por los *callers* o no regreses un campo mutable desde un accesor. Una técnica a utilizar es generar `copias defensivas` en constructores, accesors y métodos readObject (en caso de serialización).

Supongamos que implementamos la siguiente clase como inmutable:
```java
class Student {
    private final String name;
    private final List<Course> courses;

    //Constructor initialization
    ...

    public String getName() {
        return name;
    }

    public List<Course> getCourses() {
        return courses; 
    }
}
```
¿Es realmente inmutable la clase Student? Podrías confundirte y asegurar que como `courses` es `final`, como se inicializó en el construtor y además no hay ningún *mutator* definido, la lista de cursos a los que un estudiante está inscrito no puede modificarse. Pero aunque lo parezca, la clase no es completamente inmutable. `final` se refiere a que **la referencia** `courses` no apuntará a otra colección, por ejemplo, `courses = new ArrayList<>()` una vez que ha sido definido. Pero lo que sí ha quedado desprotegido es el contenido de la lista de cursos, perdiendo la exclusividad en el acceso a la misma, como se ve en el siguiente y malicioso código de un *caller*:

```java
    //Soy un profesor que tengo que completar mi número de alumnos para
    // mi no muy apreciado curso y así recibir mi bono. A ver, a ver, agregaré
    // mi curso a este inocente chico.
    student.getCourses().add(new Course("Project Management"));
   
```

Si fueras tú el estudiante en cuestión y al final del semestre recibieras una nota reprobatoria de un curso que ni siquiera registraste, o peor aun, que algún *caller* decida eliminar tus cursos con un `student.getCourses().clear()` y con eso ya no aparecieras en las listas de los profesores, ¿qué dirías? ¿crees que tus internals estaban suficientemente protegidos? Como comenté, la técnica de hacer *copia defensiva* en constructores o accesors puede librarte de estos peligros:

```java
    public List<Courses> getCourses() {
        return Collections.unmodifiableList(courses);
    }
```

## Paralelización
Desde Java 8 ya contamos con estilo funcional de programación (FP) *out-of-the-box* a través del API de Stream y las expresiones Lambda (repito, el estilo funcional porque Java no es y creo que nunca será un lenguaje funcional puro). Aunado a la FP, desde hace mucho tenemos un entorno de programacion `multi-thread` para procesamiento paralelo; los objetos inmutables encajan perfecto aquí, porque se aseguran `thread-safe`. No importa cuántos clientes accedan de forma simultánea a nuestros objetos inmutables, sus atributos expuestos son de solo lectura y no cambian.

## Conclusión
Implementar código endeble y vulnerable que no considera buenas prácticas de OOP, puede traer consecuencias al momento de exponerlo a clientes *potencialmente* maliciosos. Este tipo de código da libertad a que agentes externos manipulen el estado de nuestros objetos. 

Pero por otro lado, el código que tiene presente prácticas, incluídos el encapsulamiento y la inmutabilidad, trae muchos beneficios. A continuación un resumen de estos: 
* Objetos que son los mismos durante todo su ciclo de vida
* Se mantiene oculto el detalle de la implementación
* Solo expone lo que es necesario
* Fácil de diseñar e implementar
* Menos propenso a errores
* Más seguro
* Fácil de probar
* Tiene bajo acoplamiento
* Es Thread-Safe
* Mejora el rendimiento limitando el número de copias del objeto
* Fácil de depurar
* Previene side-effects
 
Happy codding!!!

