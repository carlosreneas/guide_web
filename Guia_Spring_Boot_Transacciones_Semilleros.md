# Laboratorio: registro transaccional de semilleros de investigación

Spring Boot · API REST · JSON anidado · Jackson · Flyway · JPA · Bean Validation · @Transactional · MapStruct

Guía del estudiante \| Dos sesiones de 120 minutos \| Nivel intermedio

Caso inspirado en el formato institucional de actualización del semillero SIAWEB (2026-1). Los ejemplos emplean datos ficticios; no se deben publicar datos personales reales del archivo de referencia.

# 1. Propósito y producto final

Construir desde cero una API REST que registre un semillero, su director, líneas de investigación e integrantes en una sola petición. La operación debe ser atómica: si falla una validación o una inserción, no deben persistirse registros parciales.

Al finalizar, cada equipo deberá explicar el recorrido HTTP → JSON → DTO → servicio → entidades JPA → repositorios → base de datos → respuesta JSON.

# 2. Contexto universitario y alcance

El formato SIAWEB contiene información básica, complementaria, director y estudiantes. Para hacer viable la práctica se seleccionan: nombre, sigla, grupo, misión, visión, objetivo general, líneas, director e integrantes. El primer integrante de la lista se considera líder en el formato de referencia; para evitar ambigüedad en la API, esta práctica utiliza el campo booleano esLider y exige exactamente uno.

Reglas: sigla única; correo del director válido; al menos una línea y un integrante; códigos estudiantiles no repetidos dentro del semillero; exactamente un líder; ninguna escritura parcial ante un error. Las relaciones y reglas son decisiones pedagógicas, no una copia literal del sistema institucional.

# 3. Contrato JSON de entrada

```json
{
"nombre": "Semillero de Aplicaciones Web",
"sigla": "SAW-01",
"grupoInvestigacion": "Grupo de Ingeniería de Software",
"mision": "Promover la investigación aplicada",
"vision": "Desarrollar soluciones universitarias",
"objetivoGeneral": "Fortalecer competencias investigativas",
"director": {"nombre": "Docente Ejemplo", "correo": "docente@universidad.edu"},
"lineas": ["Aplicaciones Web", "Sistemas Distribuidos"],
"integrantes": [
{"codigo": "2026001", "nombre": "Ana Ejemplo", "programa": "Ingeniería de Sistemas", "esLider": true},
{"codigo": "2026002", "nombre": "Luis Ejemplo", "programa": "Ingeniería de Sistemas", "esLider": false}
]
}
```

Discusión: director es un objeto, lineas es un arreglo de cadenas e integrantes es un arreglo de objetos. ¿Cuántas clases DTO son necesarias? ¿Qué información no debería recibir directamente una entidad JPA?

# 4. Arquitectura de la aplicación

```text
Cliente HTTP (Postman)

↓ POST /api/semilleros

@RestController → Jackson → SemilleroRequest (DTO)

↓

@Service + @Transactional → validación de reglas → mapeo

↓

JpaRepository → Hibernate/JPA → PostgreSQL

↑

SemilleroResponse (DTO) ← respuesta 201 Created
```

## 4.1 Spring Boot y Spring Web

EXPLICACIÓN CONCEPTUAL. Spring Boot configura y arranca la aplicación; Spring Web expone endpoints HTTP mediante @RestController, @PostMapping y @RequestBody.

EJEMPLO EXPLICADO. @RequestBody indica que el cuerpo JSON debe convertirse a un objeto Java. ResponseEntity permite establecer estado HTTP y cuerpo.

ACTIVIDAD DEL ESTUDIANTE. Identificar en el flujo qué componente recibe la petición y cuál debe contener la lógica de negocio.

RESULTADO ESPERADO. El controlador no realiza operaciones SQL ni decide reglas de integridad.

## 4.2 Jackson, DTO y Java

EXPLICACIÓN CONCEPTUAL. Jackson deserializa JSON en objetos Java. Un DTO representa datos del contrato externo; una Entity representa persistencia. Son responsabilidades distintas.

EJEMPLO EXPLICADO. Un objeto director corresponde a DirectorRequest; cada elemento de integrantes corresponde a IntegranteRequest.

ACTIVIDAD DEL ESTUDIANTE. Diseñar SemilleroRequest, DirectorRequest e IntegranteRequest, con getters/setters o records según versión Java.

RESULTADO ESPERADO. Una petición válida puede deserializarse sin exponer directamente las entidades.

## 4.3 Flyway y esquema

EXPLICACIÓN CONCEPTUAL. Flyway ejecuta migraciones SQL versionadas; JPA no debe modificar el esquema por su cuenta. Hibernate valida la correspondencia entre entidades y tablas.

EJEMPLO EXPLICADO. V1\_\_crear_estructura.sql define tablas y claves; V2\_\_datos_iniciales.sql puede cargar catálogos si se requieren.

ACTIVIDAD DEL ESTUDIANTE. Diseñar migraciones reproducibles y establecer spring.jpa.hibernate.ddl-auto=validate.

RESULTADO ESPERADO. Al arrancar sobre una BD vacía aparecen las tablas y flyway_schema_history; un segundo arranque no duplica estructuras.

## 4.4 JPA, Hibernate y Repository

EXPLICACIÓN CONCEPTUAL. JPA es la especificación de persistencia; Hibernate es una implementación; Spring Data JPA facilita repositorios como JpaRepository.

EJEMPLO EXPLICADO. @Entity, @Id, @GeneratedValue, @ManyToOne y @OneToMany describen entidades y relaciones.

ACTIVIDAD DEL ESTUDIANTE. Crear entidades y repositorios, justificando dónde usar claves foráneas y restricciones UNIQUE.

RESULTADO ESPERADO. El modelo relacional representa director, semillero, líneas e integrantes con integridad referencial.

## 4.5 Bean Validation

EXPLICACIÓN CONCEPTUAL. @Valid activa validaciones declarativas sobre DTO; @NotBlank, @Email y @NotEmpty verifican requisitos básicos. Las reglas entre elementos requieren lógica adicional.

EJEMPLO EXPLICADO. @Valid @RequestBody SemilleroRequest request; @NotEmpty sobre integrantes.

ACTIVIDAD DEL ESTUDIANTE. Rechazar correo inválido, sigla vacía y lista de integrantes vacía; definir respuestas HTTP 400.

RESULTADO ESPERADO. No se intenta persistir un JSON estructuralmente inválido.

## 4.6 Transacciones Spring

EXPLICACIÓN CONCEPTUAL. @Transactional delimita una unidad de trabajo; por defecto, las excepciones no comprobadas provocan rollback. No toda excepción checked lo provoca sin configuración.

EJEMPLO EXPLICADO. Un error al validar el segundo integrante debe impedir que queden director, semillero o primer integrante guardados.

ACTIVIDAD DEL ESTUDIANTE. Implementar la operación en un método público de @Service y comprobar rollback con consultas SQL.

RESULTADO ESPERADO. Tras un fallo, los conteos de tablas permanecen iguales a los anteriores a la petición.

## 4.7 MapStruct

EXPLICACIÓN CONCEPTUAL. MapStruct genera código de conversión entre DTO y Entity en compilación; no reemplaza Jackson ni reglas de negocio.

EJEMPLO EXPLICADO. Un mapper puede convertir DirectorRequest en Director; la asociación al semillero y la unicidad del líder siguen en el servicio.

ACTIVIDAD DEL ESTUDIANTE. Primero implementar conversión manual; luego introducir MapStruct y comparar clases y responsabilidades.

RESULTADO ESPERADO. La API produce la misma respuesta antes y después de la refactorización.

# 5. Preparación del proyecto (sesión 1)

Crear un proyecto Maven desde Spring Initializr con Java 21 y las dependencias Spring Web, Spring Data JPA, Validation, Flyway Migration y PostgreSQL Driver. Usar una versión estable de Spring Boot compatible con las dependencias. MapStruct se añade en la segunda sesión.

```text
src/main/java/edu/ejemplo/semilleros/

controller/ dto/request/ dto/response/

entity/ repository/ service/ exception/ mapper/

src/main/resources/

application.properties

db/migration/V1__crear_estructura.sql
```

Configuración orientativa (reemplazar credenciales locales; nunca subir secretos al repositorio):

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/semilleros_db
spring.datasource.username=\${DB_USER}
spring.datasource.password=\${DB_PASSWORD}
spring.jpa.hibernate.ddl-auto=validate
spring.flyway.enabled=true
spring.jpa.open-in-view=false
```

# 6. Modelo relacional y migración

Modelo sugerido: DIRECTORES (id, nombre, correo UNIQUE); SEMILLEROS (id, nombre, sigla UNIQUE, grupo_investigacion, mision, vision, objetivo_general, director_id FK); LINEAS (id, nombre); SEMILLERO_LINEAS (semillero_id FK, linea_id FK, PK compuesta); INTEGRANTES (id, semillero_id FK, codigo, nombre, programa, es_lider, UNIQUE(semillero_id,codigo)).

Para esta práctica, el director se crea junto con el semillero; en una versión institucional se definiría reutilización de docentes. La regla «un líder» se valida en el servicio y puede reforzarse con un índice parcial específico de PostgreSQL para evitar más de uno.

```sql
CREATE TABLE directores (
id BIGSERIAL PRIMARY KEY,
nombre VARCHAR(160) NOT NULL,
correo VARCHAR(180) NOT NULL UNIQUE
);
CREATE TABLE semilleros (
id BIGSERIAL PRIMARY KEY,
nombre VARCHAR(180) NOT NULL,
sigla VARCHAR(30) NOT NULL UNIQUE,
grupo_investigacion VARCHAR(180) NOT NULL,
mision TEXT, vision TEXT, objetivo_general TEXT,
director_id BIGINT NOT NULL REFERENCES directores(id)
);
```

Reto: completar las tablas lineas, semillero_lineas e integrantes; verificar claves primarias, foráneas y restricciones. Evitar modificar una migración ya aplicada: crear una nueva versión.

# 7. DTO, endpoint y servicio

```java
public record DirectorRequest(
@NotBlank String nombre,
@NotBlank @Email String correo) {}
public record IntegranteRequest(
@NotBlank String codigo,
@NotBlank String nombre,
@NotBlank String programa,
boolean esLider) {}
```

Reto: definir SemilleroRequest con @Valid en director y en cada integrante; usar @NotEmpty en las colecciones. Agregar SemilleroResponse con id, sigla, totalIntegrantes y estado.

```java
@RestController
@RequestMapping("/api/semilleros")
public class SemilleroController {
private final SemilleroService service;
public SemilleroController(SemilleroService service) {
this.service = service;
}
@PostMapping
public ResponseEntity<SemilleroResponse> crear(
@Valid @RequestBody SemilleroRequest request) {
return ResponseEntity.status(201).body(service.crear(request));
}
}
```

Algoritmo del servicio: 1) validar sigla no existente; 2) verificar exactamente un líder y códigos no repetidos; 3) guardar director; 4) guardar semillero; 5) resolver o crear líneas; 6) asociar integrantes; 7) retornar DTO de respuesta. La lógica debe residir en un método @Transactional. Los alumnos deciden orden de persistencia y manejo de relaciones.

# 8. Pruebas guiadas y evidencias

| Caso | Entrada                           | Resultado esperado                            |
|------|-----------------------------------|-----------------------------------------------|
| T1   | JSON válido                       | 201; se guardan todos los registros asociados |
| T2   | Sigla duplicada                   | 409; no hay nuevas filas                      |
| T3   | Sin integrantes                   | 400; ninguna escritura                        |
| T4   | Dos líderes                       | 400; ninguna escritura                        |
| T5   | Código repetido                   | 400; ninguna escritura                        |
| T6   | Correo inválido                   | 400; validación del DTO                       |
| T7   | Fallo después de guardar director | Rollback; no queda director huérfano          |

Para T7 se puede provocar temporalmente una excepción RuntimeException dentro del servicio, después de persistir el director. Retirar esa simulación al terminar. Capturar la excepción sin relanzarla puede impedir el rollback esperado.

Evidencias: capturas de Postman con códigos HTTP, consulta SQL antes/después de T7, estructura de paquetes y tabla flyway_schema_history.

# 9. Sesión 2: refactorización con mappers

Paso A. Localizar código repetido de tipo setNombre/getNombre. Paso B. Extraer un mapper manual. Paso C. Añadir MapStruct y su procesador de anotaciones al build Maven. Paso D. Sustituir conversiones simples sin mover al mapper consultas, validaciones de negocio ni transacciones.

@Mapper(componentModel = "spring")

public interface DirectorMapper {

Director toEntity(DirectorRequest request);

}

Analizar: Jackson transforma JSON ↔ DTO; MapStruct transforma DTO ↔ Entity; Hibernate persiste Entity ↔ tablas. Comprobar que el código generado por MapStruct se produce al compilar y que los siete escenarios continúan pasando.

# 10. Cronograma y entregables

Sesión 1: 15 min contexto/JSON; 20 min proyecto; 25 min Flyway; 25 min entidades; 25 min DTO/controlador; 10 min verificación. Sesión 2: 35 min servicio transaccional; 25 min pruebas; 25 min mapper manual; 25 min MapStruct; 10 min cierre.

Entregar: repositorio Maven, migraciones SQL, colección de peticiones o instrucciones curl, README de ejecución, evidencias de rollback y una comparación breve de conversión manual frente a MapStruct.

# 11. Rúbrica (100 puntos)

• Proyecto y configuración reproducible: 10

• Migraciones Flyway e integridad relacional: 15

• Entidades y repositorios JPA: 15

• DTO, JSON y validación: 15

• Servicio y reglas de negocio: 20

• Atomicidad y prueba de rollback: 15

• MapStruct y comparación argumentada: 5

• Evidencias y README: 5

# 12. Preguntas para discusión

• ¿Por qué no es recomendable recibir directamente una Entity en @RequestBody?

• ¿Qué diferencia hay entre deserializar JSON y mapear DTO a Entity?

• ¿Qué responsabilidad corresponde a Flyway y cuál a Hibernate?

• ¿Qué sucede si la última inserción falla después de guardar tres entidades?

• ¿Cómo evitar que dos solicitudes simultáneas registren la misma sigla?

• ¿Qué regla debe validarse en el servicio y cuál conviene reforzar en la base de datos?

Referencia del caso: archivo institucional «FORMATO ACTUALIZACION SEMILLERO 2026-1_SIAWEB». Ejemplos y modelo adaptados para fines académicos.

# 13. Documentación y pruebas de la API con Swagger / OpenAPI

Objetivo: documentar y probar los endpoints REST del sistema de semilleros desde el navegador, sin depender exclusivamente de Postman. Swagger UI es la interfaz interactiva; OpenAPI es la especificación que describe el contrato de la API; springdoc-openapi genera esa descripción a partir de Spring Boot.

## 13.1 Dependencia springdoc-openapi

Explicación conceptual. Springdoc inspecciona los controladores de Spring MVC y expone un documento OpenAPI. Para Spring Boot 3 se utiliza una versión compatible de springdoc 2.x; para Spring Boot 4, una versión compatible de springdoc 3.x. Verificar la matriz de compatibilidad antes de fijar la versión. No confundir la biblioteca springdoc con Swagger UI.

Ejemplo de código:

\<dependency\>

\<groupId\>org.springdoc\</groupId\>

\<artifactId\>springdoc-openapi-starter-webmvc-ui\</artifactId\>

\<version\>VERSION_COMPATIBLE_CON_SPRING_BOOT\</version\>

\</dependency\>

Actividad del estudiante. Incorporar la dependencia al pom.xml, elegir una versión compatible con el Spring Boot del proyecto y ejecutar mvn clean spring-boot:run.

Resultado esperado. Abrir http://localhost:8080/swagger-ui/index.html y comprobar que http://localhost:8080/v3/api-docs devuelve JSON.

## 13.2 Anotaciones para describir el endpoint

Explicación conceptual. @Tag agrupa operaciones; @Operation describe una acción; @ApiResponse documenta respuestas HTTP. @Schema aporta descripciones y ejemplos de campos. Estas anotaciones enriquecen la documentación, pero no sustituyen las validaciones ni la implementación.

Ejemplo de código:

@Tag(name = "Semilleros", description = "Registro de semilleros")

@RestController

@RequestMapping("/api/semilleros")

public class SemilleroController {

@Operation(summary = "Registrar semillero",

description = "Crea un semillero con sus integrantes")

@ApiResponses({

@ApiResponse(responseCode = "201", description = "Creado"),

@ApiResponse(responseCode = "400", description = "Datos inválidos"),

@ApiResponse(responseCode = "409", description = "Conflicto")

})

@PostMapping

public ResponseEntity\<SemilleroResponse\> crear(

@Valid @RequestBody SemilleroRequest request) {

SemilleroResponse resultado = servicio.registrar(request);

return ResponseEntity.status(201).body(resultado);

}

}

Actividad del estudiante. Adaptar el ejemplo a los nombres reales de DTO, servicio y controlador del proyecto; agregar anotaciones @Schema en al menos tres propiedades del DTO.

Resultado esperado. Swagger UI muestra la operación POST, el esquema del JSON de entrada y los códigos de respuesta previstos.

## 13.3 Probar un JSON anidado desde Swagger UI

Explicación conceptual. La opción Try it out permite enviar solicitudes reales a la API. Swagger UI no simula la base de datos: al pulsar Execute se invoca el endpoint y se ejecuta la transacción.

Ejemplo de código:

POST /api/semilleros

Content-Type: application/json

{

"nombre": "Semillero SIAWEB",

"integrantes": \[

{"documento": "10001", "nombre": "Ana", "lider": true},

{"documento": "10002", "nombre": "Luis", "lider": false}

\]

}

Actividad del estudiante. Usar el JSON contractual completo de la sección 3 (el ejemplo reducido de arriba es solo ilustrativo). Ejecutar una solicitud válida y otra con un integrante inválido; observar códigos y respuestas.

Resultado esperado. La petición válida retorna 201 con el DTO de respuesta; la inválida retorna el código de error implementado y no deja registros parciales.

## 13.4 Swagger y validaciones

Explicación conceptual. Las restricciones de Jakarta Bean Validation (@NotBlank, @NotNull, @Size, @Valid) pueden reflejarse en el esquema de entrada. @Valid debe aplicarse a los objetos anidados cuando se requiera validación en cascada.

Ejemplo de código:

public class SemilleroRequest {

@NotBlank

@Schema(description = "Nombre del semillero",

example = "SIAWEB")

private String nombre;

@NotNull

@Valid

private List\<IntegranteRequest\> integrantes;

}

Actividad del estudiante. Revisar las validaciones del modelo real y documentar campos obligatorios, ejemplos y restricciones. Comparar la documentación con el comportamiento efectivo de la API.

Resultado esperado. El esquema OpenAPI representa correctamente los campos requeridos y los errores de validación se manejan de forma coherente.

## 13.5 Evidencias y preguntas de reflexión

- Captura de Swagger UI mostrando el endpoint POST /api/semilleros.

- Captura de la ejecución exitosa con respuesta HTTP 201.

- Captura de una solicitud inválida y explicación del rollback comprobado en la base de datos.

- Adjuntar el JSON generado en /v3/api-docs o una captura de su contenido.

- Responder: ¿en qué se diferencian OpenAPI, Swagger UI, Jackson y MapStruct?

- Responder: ¿por qué documentar un código 409 no hace que Spring lo devuelva automáticamente?

Nota para el docente: este módulo es una ampliación de la práctica; puede desarrollarse al final de la segunda sesión o como actividad complementaria. Swagger UI no sustituye las pruebas automatizadas ni garantiza que la transacción sea correcta.
