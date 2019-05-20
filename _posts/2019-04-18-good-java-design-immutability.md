Has oído la frase *"haz que funcione, hazlo correcto y que funcione rápido, en ese orden"*, por Kent beck o Butler Lampson's. 

Como desarrollador, y tal vez de forma inconsciente, podríamos estar cumpliendo solo con la primera parte de esta frase: *"haz que funcione"*, y olvidar las otras dos; y puede ser por muchas razones: un deadline muy próximo, carga de trabajo, desconocimiento de la tecnología utilizada o incluso malos hábitos. Comprensible. No obstante, mi responsabilidad y la tuya es avanzar hacia las otras dos premisas. Si te haces el hábito de escribir código que sigue principios de diseño y buenas prácticas de programación, te ahorrarás, entre muchas otras cosas, estrés, tiempo, dinero y hasta uno que otro insulto remoto de quienes mantengan tu código en el futuro.

Pero, ¿cómo escribir código que cumpla esta frase? En este post veremos una de las muchas prácticas para lograrlo, la **Inmutabilidad**.

En OOP, la idea principal es abstraer un problema como objetos del mundo real. Los objetos son considerados *first-class citizens*, con sus atributos (propiedades) y métodos (comportamiento). Pero, como en el mundo real, hay cierta información de los objetos que deben mantenerse ocultos (encapsulamiento) y libres de cambios (invariants), y el mundo exterior no debe (o al menos no debería) conocerlos. Cuando digo *libres de cambios* me refiero a que desde su creación y durante el ciclo de vida, el objeto no debe mutar.

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

Este parece ser un inocente e inofensivo programa, pero más bien es ingenuo, presenta varias desventajas y al mismo tiempo corre varios peligros en manos de sus *callers*. *Joshua Bloch* recomienda: *minimizar el acceso de clases y miembros de clase* en su libro *Effective Java*, algo que la clase `PersonInfo` claramente no hace. *"Un componente bien diseñado oculta (encapsula) todos los detalles de su implementación y los componentes se comunican entre sí solo a través de sus APIs"*

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
        // a la persona como procesada por el servicio de logging
        personInfo.name = personInfo.name + "_logged";
    }
```

A ver a ver, un momento, ¿cómo que: aprovechamos para poner el sufijo...? El nombre en un objeto `PersonInfo` no debería cambiar. Sin embargo aquí ha ocurrido y los *invariants* del objeto, pues... han variado. Los atributos de `PersonInfo` están expuestos de tal manera que cualquier otro objeto puede manipularlos arbitrariamente. ¿Querrías tú que alguien cambiara tu nombre de esta manera? Creo que no.

Podemos remediar el problema con encapsulamiento y mejor aún implementando **inmutabilidad**. En este sentido, otra vez Joshua Bloch recomienda: *"en clases públicas, utiliza métodos accesors, no uses campos públicos"* y *"Minimiza la mutabilidad"*. Si bien Java soporta inmutabilidad de clases, no nos forza a su utilización, pero nosotros, al escribir nuestros programas, sí podríamos hacerlo, y veremos cómo conseguirlo. Vamos por partes, primero...

## ¿Qué es una clase inmutable?
Una clase inmutable es simplemente aquella cuyas instancias no pueden ser modificadas una vez que su información ha sido definida. No habrá ninguna modificación a la misma durante su ciclo de vida. Un ejemplo de clase inmutable en Java es la clase `String`. Ahora veamos...

## ¿Cómo diseñar una clase inmutable?
Para que los objetos de tipo `PersonInfo` pasen de ser ingenuos desprevenidos a bien protegidos debemos seguir varios pasos:
* Declarar la clase como `final`
* Cambiar el modificador de acceso de `public` a `private`. Recuerda, minimiza el acceso.
* Declarar cada atributos como `final`.
* No exponer ningún *mutator*.
* Exponer los atributos, si se necesita, solo con *accesors*.
* Inicializar los atributos con ayuda de Constructores.
* Poner especial atención a colecciones y parámetros de constructores.

## Versión inmutable de PersonInfo
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
Analicemos los pasos anteriores:
Si marcamos la clase como final, evitamos que a través de la herencia, una subclase consiga acceso a los atributos de la clase padre y modifique sus valores.
 
Cuando se cambia el modificador de acceso de los atributos de ~~`public`~~ a `private`, se proteje el estado interno de los objetos. 

Al marcar como `final` a los atributos de una clase, se asegura que los atributos del objeto no pueden ser modificados una vez que se han definido. En el caso de la clase `PersonInfo`, un constructor ayuda a inicializar los valores de sus atributos. 

**No implementes métodos que modifiquen el estado del objeto**. Como se puede ver, ahora la clase `PersonInfo` no tiene *mutators*. Con este ajuste, evitamos cambios al estado del objeto.

Si fuera necesario, la clase expone información solo usando *accesors*, que son considerados como su *API*.

Si lo notaste, el *accesor* `getBirthday()` devuelve un `String` y no un `LocalDateTime`. No existe ninguna regla que indique que se deben regresar los atributos de un objeto usando el mismo tipo de dato de dichos atributos. Así que en este caso, encapsulamos el birthday de una persona, exponiendo el dato de mes/año como un `String`. En realidad el `birthday` podría ser un `LocalDateTime` o un `timestamp` en tipo de dato `long`. Los *clientes* no lo sabrían.

## Colecciones y parámetros en constructores
Si la clase que estamos definiendo tiene referencias a objetos mutables (`Collection`s, `StringBuilder`s, por ejemplo) que fueron recibidos como parámetros en constructores o que están expuestos a través de *accesors*, tienes que asegurarte que es el objeto el que tiene **acceso exclusivo** a esto atributos. Esto es, el cliente que construye la instancia o pide por el atributo no debe ser capaz de modificar dicho objeto. ¿Cómo lograrlo? **No inicialices un campo con referencias a objetos provistas por los clientes** o **no regreses un campo mutable desde un accesor**. Una técnica para asegurar el acceso exclusivo es generar *copias defensivas* tanto de parámetros mutables recibidos en constructores, como de atributos mutables en *accesors* y en métodos `readObject` (en caso de serialización).

Supongamos que implementamos la siguiente clase como inmutable:
```java
final class Student {
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
¿Es realmente inmutable la clase Student? Sin analizar con detenimiento podrías asegurar que como `courses` es `final`, como se inicializó en el construtor y además como no hay ningún *mutator* definido, la lista de cursos a los que un estudiante está inscrito no puede modificarse. 
Pero aunque lo parezca, la clase `Student` no es completamente inmutable. La palabra reservada `final` en un atributo de clase garantiza que una **referencia**, en este caso `courses`, nunca apunte a otro objeto o tenga otro valor una vez que ha sido definida (por ejemplo, `courses = new ArrayList<>()` dispararía un error de compilación). Si bien la referencia se ha protegido, lo ha quedado desprotegido aquí es el objeto al que apunta, la lista de cursos como tal, perdiendo así la exclusividad en el acceso a la misma. Esto se ve en el siguiente código de un malicioso *caller*:

```java
class Teacher {

    /**
     * Como profesor tengo que completar mi número de alumnos para
     * mi no muy apreciado curso y así recibir mi bono. Fácil, agregaré
     * mi curso al primer inocente de mi lista de estudiantes.
    public void addACourseToANaiveStudent() { 
        allMyStudents().get(0).getCourses().add(new Course("Project Management"));
        log.info("Venga mi bono");
   }
   ...
```

Si fueras tú el estudiante en cuestión y al final del semestre recibieras una nota reprobatoria de un curso que ni siquiera registraste, o peor aun, que algún *caller* decida eliminar tus cursos con un `student.getCourses().clear()` y con eso ya no aparecieras en las listas de los profesores, ¿qué dirías? ¿crees que tus *internals* estaban suficientemente protegidos? Como comenté, la técnica de hacer *copias defensivas* en constructores o accesors puede librarte de estos peligros:

```java
    public List<Courses> getCourses() {
        return Collections.unmodifiableList(courses);
    }
```

## Paralelización
Otro beneficio de los objetos inmutables es su uso en la paralelización. Desde Java 8 ya contamos con un estilo funcional de programación (FP) *out-of-the-box* a través del API de Stream y las expresiones Lambda (repito, es un *estilo funcional* porque Java no es y creo que nunca será un lenguaje funcional puro) y de mucho más atrás tenemos programacion `multi-thread`; ¿y por qué los objetos inmutables encajan perfecto aquí?, porque se aseguran `thread-safety`. No importa cuántos clientes accedan de forma simultánea a nuestros objetos inmutables, sus atributos expuestos son de solo lectura y no cambian.

## ¿Desventajas?
La única desventaja real de utilizar objetos immutables sería que si después de haber sido concebidos como inmutables, ahora sea necesario mutar el estado de estos objetos (sus atributos), habrá que crear nuevos objetos y/o hacer una refactorización del código que los utiliza. Crear objetos puede resultar costoso (no siempre cierto para objetos inmutables debido a las optimizaciones de la JVM y GC), especialmente para aquellos que son bastante grandes y de los cuales solo *mutaría* una pequeña cantidad de sus atributos. La concatenación de `String`s (que son inmutables), por ejemplo, en un proceso de miles o millones de cadenas sería ineficiente. Usar un `StringBuilder` (mutable), es más adecuado en este escenario. 

La clave radica en evaluar la naturaleza del dominio de negocio de los objetos; validar si estos deben o no mutar. Por ejemplo, para una aplicación de control de peso, el atributo `peso` de un objeto `Persona` no debe ser inmutable, no aplica, puesto que una persona sube o baja de peso. Así que, hay que usar la inmutabilidad donde sea pertinente y no en todos lados. Pero entre menos mutable sea nuestro modelo, más ventajas obtendremos.

Para el caso de la creación de objetos de clases que tienen muchos atributos, inicializarlos a través de un constructor puede caer en el antipattern *Telescoping Constructor* o *constructor telescópico*, pero esto más que una desventaja, es un problema de legibilidad y mantenimiento del código, además de no ser exclusivo de objetos inmutables. Para solucionarlo, el patrón "Builder" es la solución *de facto*.

## Conclusión
Implementar código endeble y vulnerable que no considera buenas prácticas de OOP, puede traer consecuencias al momento de ser expuesto a clientes *potencialmente* maliciosos. Este tipo de código da libertad a que agentes externos manipulen el estado de nuestros objetos o que al depurarlo, para encontrar en qué punto alguien le agregó o quitó algo, resulte en un verdadero dolor de cabeza.

Sin embargo, el código que tiene presente buenas prácticas, en este caso la inmutabilidad, trae muchos beneficios, como los siguientes: 
* Los objetos son los mismos durante todo su ciclo de vida
* Se mantiene oculto el detalle de la implementación
* Solo expone lo que es necesario
* Es fácil de diseñar e implementar
* Es menos propenso a errores
* Es más seguro
* Fácil de probar
* Tiene bajo acoplamiento
* Es thread-safe
* Puede mejorar el rendimiento, limitando el número de copias del objeto, como en el caso de los `String`s
* Fácil de depurar
* Previene side-effects
* Perfecto para paralelizar
 
Así que te invito a que, en la medida de lo posible, y donde aplique la técnica de la inmutabilidad, la implementes.

Happy codding!!!
