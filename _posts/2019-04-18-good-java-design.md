# Good Java Design. Immutability

Has oído la frase "haz que funcione, hazlo correcto y que funcione rápido, en ese orden", en algún punto de carrera de desarrollo. Diré que el autor de esta frase puede ser cualquiera, Kent beck o Butler Lampson's o algún otro, no importa. Lo que a nosotros nos intesera es la interpretación de la misma.

Como desarollador, y tal vez inconsientemente, cumples solo la primera parte de esta frase: "haz que funcione", y las razones de que así sea pueden ser muchas: estás enfrentándote a un deadline muy próximo, mucha carga de trabajo, desconocimiento de la tecnología utilizada o incluso podrían ser malos hábitos. Esto hasta cierto punto puede entenderse, pero es cierto que tu responsabilidad es avanzar hacia las otras dos premisas conforme ganas experiencia. Si te haces el  hábito de escribir buen código, siguiendo principios de diseño, te ahorrarás tiempo, dinero y quizá malos deseos de aquellos que mantengan tu código en el futuro.

Pero, ¿cómo escribir código Java bien diseñado, que funcione, que sea correcto y además con excelente rendimiento? Es este post veremos una de las forma de lograrlo, la Inmutabilidad.

En OOP, la idea principal es abtraer un problema como objetos del mundo real. Los objetos son considerados first-class citizens, con sus propios atributos (propiedades) y métodos (comportamiento).

Hay ciertos atributos de los objetos que deben mantenerse ocultos (encapsulamiento) y libres de cambios (invariants), y el mundo exterior no debe (a al menos no debería) conocer cómo son en el interior (encapsulamiento).




Sin entrar en temas de especificaciones, los métodos o funciones son módulos auto-contenidos de código, que realizan una tarea específica. Los métodos toman 0 o más parámetros como entrada, ejecutan un procesamiento u operación y pueden o no regresar un resultado. Una vez que un método es implementado, lo podemos utilizar una y otra vez. En Java, particularmente, los métodos están asociados a un Objeto y normalmente representan el comportamiento del objeto.

¿Cómo hacer que un método funcione?
Supongamos que queremos validar que la hora de inicio de un Evento al momento de reegistrarlo sea posterior a la hora del registro.
Si no es así, debemos indicar un error a través de una excepción, por ejemplo.
```java
class EventRequest {
   ...
   private LocalDateTime registered;
   
   public LocalDateTime getRegistered() {
      return registered;
   }
   
   public void setRegistered(LocalDateTime registered) {
      this.registered = registered;
   }
   ...
}
```

```java
...
public class EventRegistration {
  ...
  public void register(EventRequest eventRequest) {
    ...
    LocalDateTime now = LocalDateTime.now();
    if(validateEvent(eventRequest, now)) {
      ...
      throw new RuntimeException();
    }
    ...
  }
  
  private boolean validateEvent(EventRequest eventRequest, LocalDateTime now) {
      return eventRequest.getRegistered().isBefore(now)
}
```
Este parece ser un inofensivo programa, pero tiene varias deventajas a la vez que corre varios peligros.

## No pongas acceso public, protected o package private a un método si no es necesario.

Incluso no definas el método si no se necesita. Al diseñar clases en OOP y en Java, debes asegurar que sus *internals* no sean modificados. Un objeto de tipo EventRequest tiene  una fecha de registro y no debería dejar oportunidad que ningún otro código modifique este hecho. 

Para asegurar esto, la clase debe asegurar que una vez establecida la fecha de registro, esta no puede ser modificada.
```java
class EventRequest {
   ...
   private final LocalDateTime registered;
   
   public EventRequest(LocalDateTime registered) {
     this.registered = registered;
   }
   
   public LocalDateTime getRegistered() {
      return registered;
   }
   
   //Elimina el setter
   ...
}

Con esto aseguramos que 

## Consecuencias de escribir malos métodos
Si te toma horas entender lo que hace una clase, un método o un paquete, este podría ser indicio de un mal diseño. Esto puede traer estrés para ti al momento de mantener el código de aguien más. Así que asegúrate de no producir esto para quienes tomarán tu código para darle mantenimiento, debuguear y/o extender.

# Mutabilidad
