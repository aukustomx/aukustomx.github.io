---
title: "Arquitectura Hexagonal y Feature Flags: Construyendo Sistemas Flexibles y Mantenibles"
date: 2026-01-16
author: "Tu Nombre"
tags: ["arquitectura", "hexagonal", "ports-and-adapters", "feature-flags", "solid", "typescript"]
description: "Un caso de estudio práctico sobre cómo combinar Arquitectura Hexagonal, principios SOLID y Feature Flags para construir sistemas desacoplados y ágiles en producción."
---

# Arquitectura Hexagonal y Feature Flags: Construyendo Sistemas Flexibles y Mantenibles

*Un caso de estudio práctico para desacoplar la lógica de negocio y agilizar los lanzamientos.*

## Introducción: El Problema del Acoplamiento

En el fascinate mundo del desarrollo de software, la agilidad es reina. Sin embargo, a menudo nos encontramos atrapados en un ciclo frustrante: un pequeño cambio en la lógica de negocio, como activar o desactivar una funcionalidad, requiere modificar código, pasar por un ciclo completo de CI/CD y realizar un despliegue. Este proceso no solo es lento y costoso, sino también propenso a errores.

Imagina este escenario común en una plataforma de banca digital: tu sistema dispara encuestas de satisfacción a proveedores externos como **QuestionPro** o **Medallia** después de ciertas interacciones del usuario. El negocio necesita habilitar o deshabilitar estas encuestas frecuentemente, a veces solo para una de las cuatro aplicaciones que componen el monorepo.

El enfoque tradicional sería algo así:

```typescript
// Un servicio cualquiera en la aplicación
async function handleTransactionSuccess(transaction) {
  // ... lógica de la transacción ...

  // TODO: Comentar esta línea cuando el negocio pida desactivar encuestas
  await surveyProvider.trigger("transaccion_exitosa");
}
```

Este simple comentario o condicional es la punta del iceberg. Debajo de la superficie se esconde un mar de problemas:

| Problema | Impacto |
|:---------|:--------|
| **Acoplamiento Fuerte** | La lógica de negocio está directamente acoplada al SDK del proveedor de encuestas |
| **Modificaciones Constantes** | Cada cambio requiere la intervención de un desarrollador |
| **Despliegues Riesgosos** | Un cambio trivial exige un despliegue completo, introduciendo el riesgo de regresiones |
| **Falta de Escalabilidad** | Añadir nuevos proveedores convierte el código en un laberinto de condicionales |
| **Testing Complejo** | Probar la lógica de negocio requiere configurar conexiones a servicios externos |

¿Y si pudiéramos cambiar el comportamiento de nuestra aplicación en producción, de forma segura y sin tocar una sola línea de código? Este artículo explora una solución robusta que combina tres conceptos poderosos: la **Arquitectura Hexagonal (Ports & Adapters)**, los **principios SOLID** y una gestión inteligente de **Feature Flags**.

## Arquitectura Hexagonal: Protegiendo el Corazón del Sistema

La Arquitectura Hexagonal, acuñada por Alistair Cockburn en 2005, propone una forma de estructurar el software que pone la lógica de negocio en el centro y la aísla de las dependencias externas como bases de datos, APIs, interfaces de usuario y servicios de terceros.

### El Concepto del Hexágono

Imagina tu aplicación como un hexágono donde cada lado representa un punto de conexión con el mundo exterior. La forma hexagonal es solo una representación visual; lo importante es el concepto de **separación de responsabilidades** mediante capas concéntricas.

**Dentro del Hexágono** vive el **Core Domain**, la lógica de negocio pura y agnóstica a la tecnología. Este núcleo no sabe si hablas con una base de datos PostgreSQL, MongoDB, una API REST o un archivo de texto. Solo conoce conceptos del dominio del negocio.

**En los Vértices del Hexágono** se encuentran los **Puertos (Ports)**. Son interfaces que definen un contrato, especificando *qué* se puede hacer pero no *cómo* se hace. Los puertos actúan como puntos de entrada y salida del hexágono.

**Fuera del Hexágono** están los **Adaptadores (Adapters)**. Son las implementaciones concretas de los puertos. Un adaptador podría envolver un cliente de base de datos, una llamada a una API REST, un SDK de un tercero o incluso una interfaz de línea de comandos. Los adaptadores traducen las intenciones del Core en acciones concretas del mundo real.

![Arquitectura Hexagonal Simplificada]({{ site.url }}/img/ports-and-adapters.gif)

### Tipos de Puertos y Adaptadores

La arquitectura distingue entre dos tipos de interacciones:

1. **Driving Ports/Adapters (Lado Primario)**: Inician la interacción con el sistema. Un controlador HTTP que recibe una petición REST es un *driving adapter* que llama a un *driving port* en el Core. También se les conoce como **Primary Ports**.

2. **Driven Ports/Adapters (Lado Secundario)**: Son invocados por el Core cuando necesita algo del exterior. Cuando el Core necesita persistir datos, llama a un *driven port* (ej. `RepositorioUsuarios`), y un *driven adapter* (ej. `PostgresRepositorioUsuarios`) lo implementa. También se les conoce como **Secondary Ports**.

### La Regla de Oro: La Dirección de las Dependencias

El principio fundamental de la Arquitectura Hexagonal es la **Regla de la Dependencia**: todas las dependencias apuntan hacia adentro, hacia el Core Domain. El Core no depende de nada externo; son los detalles externos los que dependen del Core a través de las interfaces que este define.

Esta inversión de dependencias es lo que permite que el Core permanezca estable y protegido de los cambios en la infraestructura. Puedes cambiar de base de datos, de framework web o de proveedor de servicios sin tocar una sola línea de la lógica de negocio.

## Caso de Estudio: Sistema de Control de Encuestas

Apliquemos estos conceptos a nuestro problema real. Necesitamos un sistema que decida si se dispara una encuesta, pero sin saber de dónde viene la decisión (el feature flag) ni a quién se le envía (el proveedor de encuestas).

### Paso 1: Definir el Lenguaje del Dominio

Antes de escribir código, debemos entender el lenguaje del negocio. En nuestro caso, el negocio habla de "features" o "capacidades" que se pueden habilitar o deshabilitar. No habla de "llaves en S3" o "endpoints de REST".

A continuación se observa el diagrama de Arquitectura de la solución siguiendo una arquitectura Hexagonal

![Control de Encuestas en aplicativo con Arquitectura Hexagonal]({{ site.url }}/img/encuestas-feature-flags-hexagonal.png)

Por eso, definimos un `enum` que representa conceptos de dominio:

```typescript
// core/domain/Feature.ts
export enum Feature {
  CONSUMIR_ENCUESTA = 'CONSUMIR_ENCUESTA',
  MOSTRAR_MODULO_NOMINA = 'MOSTRAR_MODULO_NOMINA',
  MOSTRAR_PAGO_CREDITO_AUTO = 'MOSTRAR_PAGO_CREDITO_AUTO',
  MOSTRAR_INVERSIONES = 'MOSTRAR_INVERSIONES',
}
```

Esta es una distinción crucial. **Los Feature Flags se tratan como conceptos de dominio**, no como llaves técnicas. El Core habla en términos de `CONSUMIR_ENCUESTA`, y es responsabilidad de los adaptadores traducir esto a la llave técnica que corresponda en cada contexto (`app1.surveys.enabled`, `surveys-toggle`, etc.).

### Paso 2: Definir los Puertos (Contratos)

Ahora definimos las interfaces que nuestro Core necesita para comunicarse con el exterior. Estos son los **Driven Ports** porque el Core los invocará cuando necesite algo.

```typescript
// core/ports/FlagsService.ts
export interface FlagsService {
  /**
   * Verifica si una feature está habilitada
   * @param feature - Concepto de negocio del dominio
   */
  isEnabled(feature: Feature): Promise<boolean>;
  
  getValue<T>(feature: Feature): Promise<T | null>;
  refresh(): Promise<void>;
}
```

```typescript
// core/ports/EncuestasService.ts
export interface EncuestasService {
  /**
   * Dispara un evento de encuesta
   * @param eventName - Nombre del evento de negocio
   * @param payload - Datos del evento
   */
  triggerEvent(eventName: string, payload: EncuestaPayload): Promise<void>;
  
  isAvailable(): Promise<boolean>;
}
```

Observa que estas interfaces son completamente agnósticas a la tecnología. No mencionan AWS, REST, HTTP, JSON o cualquier detalle de implementación. Son contratos puros que describen *qué* se puede hacer.

### Paso 3: Implementar la Lógica de Negocio

Con los puertos definidos, podemos implementar la lógica de negocio en el Core Domain. Este servicio orquesta la decisión de disparar encuestas:

```typescript
// core/domain/EncuestasServiceDomain.ts
export class EncuestasServiceDomain {
  constructor(
    private readonly flagsService: FlagsService,
    private readonly encuestasService: EncuestasService
  ) {}

  async dispararEncuestaSiEstaHabilitada(
    eventName: string,
    payload: EncuestaPayload
  ): Promise<boolean> {
    try {
      // 1. Decisión de negocio basada en feature flag
      const encuestasHabilitadas = await this.flagsService.isEnabled(
        Feature.CONSUMIR_ENCUESTA
      );

      if (!encuestasHabilitadas) {
        console.log('Encuestas deshabilitadas. No se dispara.');
        return false;
      }

      // 2. Verificar disponibilidad del servicio
      const disponible = await this.encuestasService.isAvailable();
      if (!disponible) {
        console.warn('Servicio de encuestas no disponible.');
        return false;
      }

      // 3. Disparar encuesta
      await this.encuestasService.triggerEvent(eventName, payload);
      console.log('Encuesta disparada exitosamente');
      return true;

    } catch (error) {
      console.error('Error al disparar encuesta:', error);
      // Fail-safe: no propagar el error para no afectar el flujo principal
      return false;
    }
  }

  async estanEncuestasHabilitadas(): Promise<boolean> {
    try {
      return await this.flagsService.isEnabled(Feature.CONSUMIR_ENCUESTA);
    } catch (error) {
      console.error('Error al verificar feature flag:', error);
      return false; // Fail-safe
    }
  }
}
```

Observa la belleza de este código. Es limpio, expresivo y no contiene una sola pista sobre AWS S3, QuestionPro, REST o bases de datos. Es **lógica de negocio pura**, protegida dentro de su hexágono. Depende únicamente de abstracciones (`FlagsService` y `EncuestasService`), no de implementaciones concretas.

### Paso 4: Implementar los Adaptadores

Ahora conectamos el Core al mundo real creando clases concretas que implementan nuestros puertos.

#### Adapter para Feature Flags en AWS S3

```typescript
// adapters/flags/AwsS3FlagService.ts
export class AwsS3FlagService implements FlagsService {
  private cache: Map<string, unknown> = new Map();
  private lastRefresh: Date | null = null;
  private readonly cacheTTL: number = 60000; // 60 segundos

  constructor(
    private readonly s3Client: S3Client,
    private readonly bucketName: string,
    private readonly fileName: string,
    private readonly featureKeyMapping: Record<Feature, string>
  ) {}

  async isEnabled(feature: Feature): Promise<boolean> {
    await this.refreshIfNeeded();

    // Mapear Feature del dominio a key técnica
    const technicalKey = this.featureKeyMapping[feature];
    if (!technicalKey) {
      console.warn(`No mapping found for feature: ${feature}`);
      return false;
    }

    const value = this.cache.get(technicalKey);
    return value === true;
  }

  async refresh(): Promise<void> {
    try {
      console.log(`Refreshing flags from S3: s3://${this.bucketName}/${this.fileName}`);

      const response = await this.s3Client.getObject({
        Bucket: this.bucketName,
        Key: this.fileName,
      });

      const data = JSON.parse(await response.Body.transformToString());

      // Actualizar caché
      this.cache.clear();
      Object.entries(data).forEach(([key, value]) => {
        this.cache.set(key, value);
      });

      this.lastRefresh = new Date();
      console.log(`Flags refreshed successfully. Cache size: ${this.cache.size}`);
    } catch (error) {
      console.error(`Error refreshing flags from S3:`, error);
      throw error;
    }
  }

  private async refreshIfNeeded(): Promise<void> {
    const now = Date.now();
    const shouldRefresh = 
      !this.lastRefresh || 
      (now - this.lastRefresh.getTime()) > this.cacheTTL;

    if (shouldRefresh) {
      await this.refresh();
    }
  }

  // ... otros métodos
}
```

Este adaptador se encarga de toda la complejidad de comunicarse con AWS S3: autenticación, manejo de errores, caché, mapeo de llaves técnicas. El Core nunca se entera de estos detalles.

#### Adapter para Encuestas con QuestionPro

```typescript
// adapters/encuestas/QuestionProEncuestasService.ts
export class QuestionProEncuestasService implements EncuestasService {
  private isServiceAvailable: boolean = true;
  private failureCount: number = 0;
  private readonly maxFailures: number = 3;

  constructor(
    private readonly apiKey: string,
    private readonly surveyId: string,
    private readonly baseUrl: string = 'https://api.questionpro.com'
  ) {}

  async triggerEvent(eventName: string, payload: EncuestaPayload): Promise<void> {
    if (!await this.isAvailable()) {
      throw new Error('Service unavailable (circuit breaker open)');
    }

    try {
      console.log(`Triggering event: ${eventName}`);

      // Transformar payload genérico a formato QuestionPro
      const questionProPayload = this.transformPayload(eventName, payload);

      // Llamar a la API de QuestionPro
      await this.sendToQuestionPro(questionProPayload);

      // Resetear contador de fallos
      this.failureCount = 0;
      this.isServiceAvailable = true;

      console.log(`Event sent successfully: ${eventName}`);
    } catch (error) {
      console.error(`Error sending event:`, error);
      this.handleFailure();
      throw error;
    }
  }

  async isAvailable(): Promise<boolean> {
    // Implementación de circuit breaker
    return this.isServiceAvailable;
  }

  private transformPayload(eventName: string, payload: EncuestaPayload): any {
    return {
      surveyID: this.surveyId,
      responseID: payload.sessionId || this.generateResponseId(),
      customVariables: {
        event_name: eventName,
        user_id: payload.userId,
        timestamp: payload.timestamp?.toISOString() || new Date().toISOString(),
        ...payload.metadata,
      },
      externalReference: payload.userId,
      source: 'web_app',
    };
  }

  // ... otros métodos
}
```

Este adaptador encapsula toda la lógica específica de QuestionPro: formato de payload, manejo de errores, circuit breaker pattern. Si mañana necesitamos cambiar a Medallia, simplemente creamos un nuevo adaptador sin tocar el Core.

### Paso 5: Ensamblar con Inyección de Dependencias

El último paso es decirle a nuestra aplicación qué implementaciones concretas usar. Esto se hace en la capa más externa, la de configuración:

```typescript
// config/DependencyInjection.ts
export class DependencyContainer {
  private static instance: DependencyContainer;
  private flagsService?: FlagsService;
  private encuestasService?: EncuestasService;
  private encuestasServiceDomain?: EncuestasServiceDomain;

  private constructor(private config: AppConfig) {}

  static getInstance(config?: AppConfig): DependencyContainer {
    if (!DependencyContainer.instance) {
      if (!config) {
        throw new Error('Config required on first initialization');
      }
      DependencyContainer.instance = new DependencyContainer(config);
    }
    return DependencyContainer.instance;
  }

  getEncuestasServiceDomain(): EncuestasServiceDomain {
    if (!this.encuestasServiceDomain) {
      const flagsService = this.getFlagsService();
      const encuestasService = this.getEncuestasService();
      this.encuestasServiceDomain = new EncuestasServiceDomain(
        flagsService,
        encuestasService
      );
    }
    return this.encuestasServiceDomain;
  }

  private getFlagsService(): FlagsService {
    if (!this.flagsService) {
      // Crear el servicio según la configuración
      if (this.config.flagsProvider === 'aws-s3') {
        const s3Client = new S3Client({ region: 'us-east-1' });
        this.flagsService = AwsS3FlagServiceFactory.createForApplication(
          s3Client,
          'feature-flags-bucket',
          this.config.applicationId
        );
      } else if (this.config.flagsProvider === 'banca-digital') {
        this.flagsService = BancaDigitalFlagServiceFactory.createFromEnv(
          this.config.applicationId
        );
      }
    }
    return this.flagsService!;
  }

  private getEncuestasService(): EncuestasService {
    if (!this.encuestasService) {
      if (this.config.encuestasProvider === 'questionpro') {
        this.encuestasService = QuestionProEncuestasServiceFactory.createFromEnv();
      } else if (this.config.encuestasProvider === 'medallia') {
        this.encuestasService = MedalliaEncuestasServiceFactory.createFromEnv();
      }
    }
    return this.encuestasService!;
  }
}
```

Este es el único lugar que conoce todas las piezas. Si mañana el negocio decide migrar de S3 a LaunchDarkly para los feature flags, solo necesitamos crear un `LaunchDarklyFlagService` y cambiar una línea en este archivo de configuración. El Core, con su valiosa lógica de negocio, permanece intacto.

### Uso en la Aplicación

Finalmente, usar el sistema en la aplicación es trivial:

```typescript
// src/index.ts
import { DependencyContainer, AppConfigFactory } from './config/DependencyInjection';

// Inicializar
const config = AppConfigFactory.createFromEnv();
const container = DependencyContainer.getInstance(config);
const encuestasService = container.getEncuestasServiceDomain();

// Usar en cualquier parte de la aplicación
async function handleTransactionSuccess(userId: string, transactionData: any) {
  // Procesar la transacción
  const transaction = await processTransaction(transactionData);
  
  // Disparar encuesta (controlado por feature flag)
  await encuestasService.dispararEncuestaSiEstaHabilitada(
    'transaccion_exitosa',
    {
      userId,
      sessionId: transaction.id,
      timestamp: new Date(),
      metadata: { amount: transaction.amount },
    }
  );
  
  return transaction;
}
```

## Principios SOLID: Los Pilares de la Arquitectura

Esta arquitectura no surge por arte de magia. Se apoya firmemente en los principios SOLID, que guían el diseño de software robusto y mantenible.

### Single Responsibility Principle (SRP)

Cada clase tiene una única responsabilidad, una única razón para cambiar. `EncuestasServiceDomain` solo decide si disparar encuestas. `AwsS3FlagService` solo se encarga de leer flags desde S3. `QuestionProEncuestasService` solo conoce cómo comunicarse con QuestionPro.

Si necesitamos cambiar la lógica de caché de los flags, solo modificamos `AwsS3FlagService`. Si cambia el formato de payload de QuestionPro, solo tocamos `QuestionProEncuestasService`. La lógica de negocio permanece intacta.

### Open/Closed Principle (OCP)

Nuestro sistema está **abierto a la extensión** pero **cerrado a la modificación**. Podemos añadir nuevos proveedores de flags (LaunchDarkly, Unleash, Split) o nuevos proveedores de encuestas (Medallia, SurveyMonkey) creando nuevos adaptadores, sin necesidad de modificar el Core Domain ni los adaptadores existentes.

```typescript
// Añadir un nuevo proveedor es tan simple como:
export class LaunchDarklyFlagService implements FlagsService {
  // Implementación específica de LaunchDarkly
}

// Y registrarlo en el container:
if (config.flagsProvider === 'launchdarkly') {
  this.flagsService = new LaunchDarklyFlagService(...);
}
```

### Liskov Substitution Principle (LSP)

Podemos sustituir una implementación de `FlagsService` (como `AwsS3FlagService`) por otra (como `BancaDigitalFlagService`) sin que el sistema se rompa, porque ambas respetan el mismo contrato definido por la interfaz.

El Core Domain no sabe ni le importa cuál implementación está usando. Solo sabe que puede llamar a `isEnabled()` y recibirá un booleano. Esta intercambiabilidad es fundamental para la flexibilidad del sistema.

### Interface Segregation Principle (ISP)

Tenemos puertos pequeños y cohesivos (`FlagsService`, `EncuestasService`) en lugar de una única interfaz monolítica como `IExternalServices` con docenas de métodos. El Core solo depende de lo que realmente necesita.

Si en el futuro necesitamos añadir funcionalidad de notificaciones, creamos un nuevo puerto `NotificacionesService` en lugar de contaminar los existentes.

### Dependency Inversion Principle (DIP)

Este es el corazón de la arquitectura. El Core Domain (un módulo de alto nivel) no depende de los adaptadores (módulos de bajo nivel). Ambos dependen de abstracciones (las interfaces de los puertos).

```typescript
// ❌ Mal: Core depende de implementación concreta
import { AwsS3FlagService } from '../adapters/flags/AwsS3FlagService';

class EncuestasServiceDomain {
  constructor(private flagsService: AwsS3FlagService) {} // Acoplamiento fuerte
}

// ✅ Bien: Core depende de abstracción
import { FlagsService } from '../ports/FlagsService';

class EncuestasServiceDomain {
  constructor(private flagsService: FlagsService) {} // Desacoplado
}
```

La flecha de la dependencia se ha invertido. En lugar de que el Core apunte a los detalles de implementación, los detalles apuntan al Core a través de las interfaces que este define.

## Feature Flags: Agilidad en Producción

Los Feature Flags (también conocidos como Feature Toggles) son una técnica que permite activar o desactivar funcionalidades en tiempo de ejecución sin necesidad de desplegar código nuevo. Son el mecanismo que nos da la agilidad que buscábamos al inicio del artículo.

### Tipos de Feature Flags

Martin Fowler identifica varios tipos de feature flags según su propósito:

| Tipo | Propósito | Longevidad | Ejemplo |
|:-----|:----------|:-----------|:--------|
| **Release Toggles** | Desplegar código incompleto sin activarlo | Temporal (días/semanas) | Nueva feature en desarrollo |
| **Experiment Toggles** | A/B testing y experimentos | Temporal (semanas/meses) | Probar dos versiones de UI |
| **Ops Toggles** | Control operacional en producción | Temporal (horas/días) | Deshabilitar feature bajo carga |
| **Permission Toggles** | Control de acceso por usuario/rol | Permanente | Features premium |

En nuestro caso, usamos **Ops Toggles** para controlar el disparo de encuestas. El negocio puede desactivarlas instantáneamente si detecta un problema con el proveedor externo, sin necesidad de un despliegue de emergencia.

### Feature Flags como Conceptos de Dominio

Una decisión arquitectónica clave en nuestra implementación es tratar los feature flags como **conceptos de dominio**, no como llaves técnicas. El Core habla de `Feature.CONSUMIR_ENCUESTA`, no de `"app1.surveys.enabled"` o `"enable-surveys-toggle"`.

Esta abstracción tiene múltiples beneficios:

1. **Type Safety**: TypeScript valida en tiempo de compilación que solo usemos features válidas.
2. **Refactoring Seguro**: Renombrar una feature es un refactor automático en el IDE.
3. **Documentación Implícita**: El enum sirve como catálogo de todas las features del sistema.
4. **Estabilidad del Core**: Cambios en las convenciones de nombres técnicas no afectan al Core.

El mapeo de `Feature` a llaves técnicas ocurre en los adaptadores, permitiendo que cada aplicación del monorepo tenga su propia convención:

```typescript
// App1 usa prefijo "app1."
const app1Mapping = {
  [Feature.CONSUMIR_ENCUESTA]: 'app1.surveys.enabled',
  [Feature.MOSTRAR_MODULO_NOMINA]: 'app1.payroll.enabled',
};

// App2 usa prefijo "app2-"
const app2Mapping = {
  [Feature.CONSUMIR_ENCUESTA]: 'app2-surveys-toggle',
  [Feature.MOSTRAR_MODULO_NOMINA]: 'app2-payroll-toggle',
};
```

## Ventajas de esta Arquitectura

Al adoptar este enfoque, hemos desbloqueado un nuevo nivel de agilidad y robustez. Las ventajas son tangibles y medibles:

### 1. Desacoplamiento Total

La lógica de negocio está completamente aislada de la infraestructura. Podemos cambiar de proveedor de base de datos, de servicio de cloud, de framework web o de API de terceros con un impacto mínimo. El Core permanece estable.

### 2. Testabilidad Máxima

Podemos probar el `EncuestasServiceDomain` en total aislamiento, inyectando *mocks* de los servicios de flags y encuestas. No se necesitan conexiones de red, bases de datos ni servicios externos para verificar la lógica de negocio.

```typescript
// Test unitario del Core Domain
describe('EncuestasServiceDomain', () => {
  it('debe disparar encuesta cuando está habilitada', async () => {
    // Arrange: Crear mocks
    const mockFlagsService: FlagsService = {
      async isEnabled() { return true; },
      async getValue() { return null; },
      async refresh() {},
    };
    
    const mockEncuestasService: EncuestasService = {
      triggerEventCalls: [],
      async triggerEvent(eventName, payload) {
        this.triggerEventCalls.push({ eventName, payload });
      },
      async isAvailable() { return true; },
    };
    
    const service = new EncuestasServiceDomain(mockFlagsService, mockEncuestasService);
    
    // Act
    const result = await service.dispararEncuestaSiEstaHabilitada('test_event', {});
    
    // Assert
    expect(result).toBe(true);
    expect(mockEncuestasService.triggerEventCalls).toHaveLength(1);
  });
});
```

### 3. Flexibilidad y Extensibilidad

Añadir nuevas funcionalidades o proveedores se convierte en un ejercicio de crear un nuevo adaptador, sin tocar el código existente y probado. Esto reduce drásticamente el riesgo de regresiones.

### 4. Lanzamientos Dinámicos sin Despliegues

Gracias a los Feature Flags, el negocio puede activar o desactivar funcionalidades en tiempo real. Algunos escenarios posibles:

- **Rollout Gradual**: Activar una feature para el 10% de los usuarios, luego 50%, luego 100%.
- **Kill Switch**: Desactivar inmediatamente una feature problemática sin esperar un despliegue.
- **Configuración por Aplicación**: Activar encuestas solo en App1 y App3, pero no en App2 y App4.
- **Horarios**: Deshabilitar encuestas fuera del horario de atención al cliente.

Todo esto modificando un archivo JSON en S3 o una entrada en una API, sin necesidad de un solo despliegue.

### 5. Desarrollo en Paralelo

Diferentes equipos pueden trabajar en diferentes adaptadores simultáneamente. El equipo de infraestructura puede estar construyendo un nuevo servicio de flags mientras el equipo de producto trabaja en la lógica del Core, ambos basándose en el mismo contrato (la interfaz del puerto).

### 6. Resiliencia y Fail-Safe

Los adaptadores pueden implementar patrones de resiliencia (circuit breaker, retry, fallback) sin contaminar el Core. Si el servicio de flags falla, podemos retornar un valor por defecto seguro. Si el proveedor de encuestas está caído, el sistema continúa funcionando sin afectar la experiencia del usuario.

### 7. Portabilidad

El Core Domain es portable entre diferentes plataformas y entornos. Podríamos ejecutarlo en Node.js, en un navegador con Deno, o incluso portarlo a otro lenguaje con relativa facilidad, porque no tiene dependencias de infraestructura.

## Comparación: Antes y Después

Para visualizar el impacto de esta arquitectura, comparemos el enfoque tradicional con el hexagonal:

| Aspecto | Arquitectura Tradicional | Arquitectura Hexagonal |
|:--------|:-------------------------|:-----------------------|
| **Acoplamiento** | Lógica de negocio acoplada a SDKs y APIs | Lógica de negocio aislada, depende solo de interfaces |
| **Cambio de Proveedor** | Requiere modificar múltiples archivos, alto riesgo | Crear nuevo adapter, bajo riesgo |
| **Testing** | Requiere mocks complejos o conexiones reales | Tests unitarios simples con mocks de interfaces |
| **Cambios de Comportamiento** | Requiere despliegue completo | Cambio de configuración en tiempo real |
| **Escalabilidad del Código** | Crece en complejidad con cada proveedor | Crece en cantidad de adapters, no en complejidad |
| **Tiempo de Desarrollo** | Inicial: rápido, Mantenimiento: lento | Inicial: moderado, Mantenimiento: rápido |
| **Riesgo de Regresiones** | Alto (cambios afectan múltiples capas) | Bajo (cambios aislados en adapters) |

## Lecciones Aprendidas y Mejores Prácticas

Al implementar esta arquitectura en un proyecto real, surgen algunas lecciones importantes:

### 1. Respetar el Hexágono

La tentación de "hacer trampa" y llamar directamente a un adapter desde la capa de presentación es real, especialmente bajo presión de tiempo. Resiste esa tentación. Cada bypass del Core erosiona la arquitectura.

### 2. Mantener el Core Limpio

El Core Domain no debe contener lógica de logging detallado, métricas, tracing o manejo de conexiones. Estas son responsabilidades de los adapters. El Core solo debe contener lógica de negocio pura.

### 3. Usar Factories para Adapters

Crear factories para los adapters facilita su configuración y testing. Una factory puede leer variables de entorno, construir dependencias y retornar una instancia lista para usar.

### 4. Documentar el Mapeo de Features

El mapeo de `Feature` enum a llaves técnicas debe estar bien documentado y centralizado. Considera usar un archivo de configuración compartido (JSON schema) que valide la estructura.

### 5. Implementar Observabilidad en Adapters

Añade logging estructurado, métricas y tracing en los adapters para tener visibilidad sobre qué está pasando en producción. Herramientas como OpenTelemetry son ideales para esto.

### 6. Establecer Guardrails Automatizados

Usa herramientas de análisis estático (ESLint, ArchUnit) para validar automáticamente que el Core no importe nada de los adapters. Esto previene violaciones accidentales de la arquitectura.

```json
// .eslintrc.json
{
  "rules": {
    "no-restricted-imports": [
      "error",
      {
        "patterns": [
          {
            "group": ["**/adapters/**"],
            "message": "Core Domain must not import from adapters"
          }
        ]
      }
    ]
  }
}
```

## Conclusión: Más Allá del Código, una Mentalidad

La combinación de Arquitectura Hexagonal, principios SOLID y una estrategia de Feature Flags bien definida es más que un simple patrón técnico; es una **mentalidad** que prioriza la flexibilidad, la resiliencia y la agilidad.

Al proteger nuestra lógica de negocio del torbellino de los detalles de implementación, construimos sistemas que no solo son más fáciles de mantener y testear, sino que también entregan más valor al negocio al permitir una adaptación rápida a las necesidades cambiantes del mercado.

En un mundo donde el cambio es la única constante, esta arquitectura nos da la capacidad de evolucionar sin miedo. Podemos experimentar, pivotar y escalar con confianza, sabiendo que nuestro Core Domain —el corazón de nuestro sistema— está protegido y estable.

La próxima vez que te enfrentes a un `if` que controla una funcionalidad, pregúntate: ¿Estoy atando mi lógica de negocio a un detalle de implementación? ¿O estoy construyendo un sistema robusto, listo para evolucionar?

La respuesta podría estar en un hexágono.


## Referencias

1. Cockburn, Alistair. "[Hexagonal architecture](https://alistair.cockburn.us/hexagonal-architecture/)". Alistair Cockburn's website.
2. Martin, Robert C. "[The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)". The Clean Code Blog.
3. Fowler, Martin. "[Feature Toggles (aka Feature Flags)](https://martinfowler.com/articles/feature-toggles.html)". Martin Fowler's website.
4. Evans, Eric. "[Domain-Driven Design: Tackling Complexity in the Heart of Software](https://www.amazon.com.mx/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215)". Addison-Wesley, 2004.


**Tags**: #arquitectura #hexagonal #ports-and-adapters #feature-flags #solid #typescript #software-engineering #clean-architecture #domain-driven-design

