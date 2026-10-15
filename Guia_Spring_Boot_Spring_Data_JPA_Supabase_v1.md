# Guía de Laboratorio: API REST con Spring Boot, Spring Data JPA y Supabase

> **Documento:** Guía docente y taller práctico de laboratorio  
> **Área:** Programación Web / Desarrollo Backend  
> **Fecha:** Octubre de 2026  
> **Proyecto central:** API REST para gestión de estudiantes conectada directamente a PostgreSQL alojado en Supabase mediante JDBC.

---

## Ficha Técnica del Taller

| Parámetro | Detalle |
| :--- | :--- |
| **Duración** | 2 sesiones de 2 horas (o 1 taller intensivo de 4 horas) |
| **Audiencia** | Estudiantes con conocimientos previos en POO Java, protocolo HTTP y arquitectura REST |
| **Prerrequisitos** | JDK 17+, Maven 3.8+, IDE (IntelliJ IDEA / VS Code / Eclipse), cliente HTTP (Postman, Insomnia o cURL) y cuenta activa en Supabase |
| **Entregable** | API CRUD funcional por capas, conexión JDBC parametrizada por variables de entorno, validaciones Bean Validation y consultas derivadas |
| **Metodología** | Exposición conceptual guiada $\rightarrow$ desarrollo paso a paso $\rightarrow$ experimentación y resolución de retos |

---

## Resultados de Aprendizaje

Al finalizar este laboratorio, el estudiante estará en capacidad de:

1. **Diferenciar** los roles de PostgreSQL, Supabase, JDBC, JPA (Jakarta Persistence), Hibernate y Spring Data JPA dentro del ecosistema de persistencia.
2. **Modelar** entidades mapeadas a esquemas relacionales existentes respetando tipos de datos, restricciones de integridad y estrategias de claves primarias.
3. **Asegurar** la configuración de persistencia desacoplando credenciales sensibles mediante variables de entorno del sistema operativo.
4. **Implementar** una arquitectura en capas (`Controller` $\rightarrow$ `Service` $\rightarrow$ `Repository` $\rightarrow$ `Database`) con control transaccional y manejo centralizado de excepciones HTTP.
5. **Formular** consultas derivadas y JPQL, identificando el impacto de las transacciones y las limitaciones de concurrencia en bases de datos compartidas.

---

## 1. Fundamentos Arquitectónicos: ¿Qué problema resolvemos?

Una aplicación sin persistencia pierde su estado al detener el proceso en la máquina virtual de Java (JVM). Para garantizar durabilidad, recurrimos a un motor de base de datos relacional. 

En esta práctica, **no utilizaremos la API REST/PostgREST ni el SDK de JavaScript de Supabase**. En su lugar, Spring Boot se conectará de forma directa al motor PostgreSQL subyacente mediante el protocolo nativo sobre TCP/IP usando el driver JDBC.

```
┌─────────────┐       HTTP / JSON       ┌────────────────────────┐
│   Cliente   │ ──────────────────────> │  EstudianteController  │
│ (Postman)   │ <────────────────────── │ (Capa Web / Endpoint)  │
└─────────────┘                         └───────────┬────────────┘
                                                    │ Invocación Java
                                                    ▼
                                        ┌────────────────────────┐
                                        │   EstudianteService    │
                                        │  (Lógica del Negocio)  │
                                        └───────────┬────────────┘
                                                    │ Invocación Java
                                                    ▼
                                        ┌────────────────────────┐
                                        │  EstudianteRepository  │
                                        │  (Spring Data Proxy)   │
                                        └───────────┬────────────┘
                                                    │ API JPA
                                                    ▼
                                        ┌────────────────────────┐
                                        │       Hibernate        │
                                        │  (Implementación JPA)  │
                                        └───────────┬────────────┘
                                                    │ Sentencias SQL
                                                    ▼
                                        ┌────────────────────────┐
                                        │      Driver JDBC       │
                                        └───────────┬────────────┘
                                                    │ TCP/IP + TLS (Puerto 5432)
                                                    ▼
                                        ┌────────────────────────┐
                                        │  Supabase (PostgreSQL) │
                                        └────────────────────────┘
```

### Matriz de Responsabilidades

| Componente | Rol en la arquitectura |
| :--- | :--- |
| **PostgreSQL** | Motor relacional: almacena tuplas, impone restricciones (`PRIMARY KEY`, `UNIQUE`, `NOT NULL`) y ejecuta planes de consulta SQL. |
| **Supabase** | Plataforma PaaS/BaaS: aprovisiona y gestiona la instancia PostgreSQL en la nube, interfaz web, Session Pooler y parámetros de red. |
| **Driver JDBC** | Controlador de bajo nivel (`org.postgresql:postgresql`): traduce llamadas Java al protocolo de red de PostgreSQL. |
| **Jakarta Persistence (JPA)** | Especificación estándar: conjunto de interfaces y anotaciones (`@Entity`, `@Table`, `@Id`) que definen el ORM. |
| **Hibernate ORM** | Motor ORM: implementa JPA, traduce operaciones sobre objetos en sentencias SQL y gestiona el ciclo de vida de las entidades. |
| **Spring Data JPA** | Capa de abstracción: sintetiza dinámicamente implementaciones de repositorios, reduciendo el código repetitivo a interfaces declarativas. |
| **Spring Boot** | Contenedor y orquestador: inicializa el pool de conexiones (HikariCP), autoconfigura el `EntityManager` e inyecta dependencias. |

---

## 2. Responsabilidades por Capa de Software

* **`Controller` (Controlador REST):** Atiende peticiones HTTP, extrae parámetros de URL o cuerpos JSON, coordina la validación sintáctica (`@Valid`) y emite códigos de estado HTTP representativos (`200`, `201`, `400`, `404`, `409`). **No debe contener lógica de negocio ni sentencias SQL.**
* **`Service` (Capa de Negocio):** Orquesta operaciones, aplica reglas del dominio (como comprobación de unicidad previa o cálculo de estados) y define los límites transaccionales (`@Transactional`).
* **`Repository` (Persistencia):** Interfaz que extiende `JpaRepository<T, ID>`. Durante el inicio de la aplicación, Spring genera un proxy dinámico en memoria que implementa métodos CRUD y consultas derivadas.
* **`Entity` (Entidad de Dominio):** Clase Java anotada con `@Entity` que modela la estructura de una tabla física en PostgreSQL.

> **Pregunta orientadora:** Si eliminamos la interfaz `EstudianteRepository`, ¿quién interactúa con PostgreSQL?  
> *Respuesta:* Sería necesario programar manualmente el mapeo mediante JDBC clásico (`Connection`, `PreparedStatement`, `ResultSet`), `JdbcTemplate` o un framework alternativo como MyBatis.

---

## 3. Preparación de la Base de Datos en Supabase

### Paso 1: Configurar Proyecto en Supabase
1. Inicia sesión en [supabase.com](https://supabase.com/).
2. Crea un nuevo proyecto. Asigna un nombre descriptivo y define una **contraseña de base de datos segura**. Conserva esta contraseña fuera del repositorio de código.

### Paso 2: Ejecutar el Script DDL
Abre el **SQL Editor** en el panel de Supabase y ejecuta el siguiente script DDL:

```sql
CREATE TABLE IF NOT EXISTS public.estudiantes (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    nombre VARCHAR(80) NOT NULL,
    apellido VARCHAR(80) NOT NULL,
    correo VARCHAR(160) NOT NULL UNIQUE,
    programa VARCHAR(100) NOT NULL
);
```

* `GENERATED BY DEFAULT AS IDENTITY`: Garantiza que PostgreSQL administre el autoincremento a través de una secuencia interna estándar SQL.
* `UNIQUE` en `correo`: Salvaguarda la integridad relacional incluso ante condiciones de carrera en peticiones concurrentes.

### Paso 3: Identificar Parámetros de Conexión
1. Dirígete a **Connect** en la barra superior o a **Project Settings** > **Database**.
2. Selecciona la pestaña **Session pooler** (puerto `5432`). Este modo es el indicado para backends Java que mantienen pools de conexiones de larga duración (como HikariCP) en redes IPv4.
3. Observa la estructura del usuario: habitualmente toma la forma `postgres.[PROJECT_REF]` y no solo `postgres`.

> **ADVERTENCIA:** Las claves de API `anon` y `service_role` pertenecen a la capa HTTP de Supabase (PostgREST/Kong). **No son credenciales válidas para JDBC**. Para la conexión JDBC únicamente se utilizan el host, puerto, usuario y contraseña de PostgreSQL.

---

## 4. Configuración del Proyecto Spring Boot

Genera el proyecto en [start.spring.io](https://start.spring.io/) con los siguientes metadatos:

* **Project:** Maven
* **Language:** Java
* **Spring Boot:** Versión 3.x estable
* **Group:** `com.ejemplo`
* **Artifact:** `estudiantes-api`
* **Packaging:** Jar
* **Java:** 17 o 21

### Dependencias en el `pom.xml`

```xml
<dependencies>
    <!-- Capa Web REST -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Persistencia ORM -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- Bean Validation (Jakarta Validation) -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Driver JDBC de PostgreSQL -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Suite de Pruebas -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 5. Estructura de Paquetes Recomendada

La clase principal decorada con `@SpringBootApplication` debe ubicarse en la raíz del paquete (`com.ejemplo.estudiantes`) para habilitar el escaneo automático de componentes:

```text
src/main/java/com/ejemplo/estudiantes/
├── EstudiantesApiApplication.java
├── controller/
│   └── EstudianteController.java
├── exception/
│   ├── ApiExceptionHandler.java
│   └── RecursoNoEncontradoException.java
├── model/
│   └── Estudiante.java
├── repository/
│   └── EstudianteRepository.java
└── service/
    └── EstudianteService.java

src/main/resources/
└── application.properties
```

---

## 6. Configuración Segura y Variables de Entorno

### Archivo `src/main/resources/application.properties`

```properties
spring.application.name=estudiantes-api
server.port=8080

# Conexión JDBC desacoplada mediante variables de entorno
spring.datasource.url=${DB_URL}
spring.datasource.username=${DB_USER}
spring.datasource.password=${DB_PASSWORD}
spring.datasource.driver-class-name=org.postgresql.Driver

# Configuración del Pool de Conexiones (HikariCP)
# Se reduce para evitar saturar el límite de conexiones de instancias gratuitas
spring.datasource.hikari.maximum-pool-size=3
spring.datasource.hikari.minimum-idle=1

# Hibernate / JPA
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

> **Principio de seguridad:** `ddl-auto=validate` delega el control del esquema a scripts manuales o herramientas de migración (Flyway/Liquibase). Hibernate solo comprobará que los campos de la clase `Estudiante` coincidan estrictamente con las columnas de PostgreSQL al iniciar.

### Configuración de Variables en Entornos de Desarrollo

Antes de ejecutar la aplicación, exporta las variables en tu terminal:

**En Linux / macOS (Bash o Zsh):**
```bash
export DB_URL="jdbc:postgresql://aws-0-sa-east-1.pooler.supabase.com:5432/postgres?sslmode=require"
export DB_USER="postgres.xxxxxxxxx"
export DB_PASSWORD="MiPasswordSeguro123"
```

**En Windows (PowerShell):**
```powershell
$env:DB_URL="jdbc:postgresql://aws-0-sa-east-1.pooler.supabase.com:5432/postgres?sslmode=require"
$env:DB_USER="postgres.xxxxxxxxx"
$env:DB_PASSWORD="MiPasswordSeguro123"
```

---

## 7. Modelado de la Entidad: `Estudiante.java`

Ubicación: `com.ejemplo.estudiantes.model.Estudiante`

```java
package com.ejemplo.estudiantes.model;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.GenerationType;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;

@Entity
@Table(name = "estudiantes", schema = "public")
public class Estudiante {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "El nombre es obligatorio")
    @Size(max = 80, message = "El nombre no puede exceder 80 caracteres")
    @Column(name = "nombre", nullable = false, length = 80)
    private String nombre;

    @NotBlank(message = "El apellido es obligatorio")
    @Size(max = 80, message = "El apellido no puede exceder 80 caracteres")
    @Column(name = "apellido", nullable = false, length = 80)
    private String apellido;

    @NotBlank(message = "El correo es obligatorio")
    @Email(message = "Formato de correo electrónico inválido")
    @Size(max = 160, message = "El correo no puede exceder 160 caracteres")
    @Column(name = "correo", nullable = false, unique = true, length = 160)
    private String correo;

    @NotBlank(message = "El programa académico es obligatorio")
    @Size(max = 100, message = "El programa no puede exceder 100 caracteres")
    @Column(name = "programa", nullable = false, length = 100)
    private String programa;

    // Constructor sin argumentos requerido por la especificación JPA
    public Estudiante() {
    }

    public Estudiante(Long id, String nombre, String apellido, String correo, String programa) {
        this.id = id;
        this.nombre = nombre;
        this.apellido = apellido;
        this.correo = correo;
        this.programa = programa;
    }

    // Getters y Setters
    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getNombre() {
        return nombre;
    }

    public void setNombre(String nombre) {
        this.nombre = nombre;
    }

    public String getApellido() {
        return apellido;
    }

    public void setApellido(String apellido) {
        this.apellido = apellido;
    }

    public String getCorreo() {
        return correo;
    }

    public void setCorreo(String correo) {
        this.correo = correo;
    }

    public String getPrograma() {
        return programa;
    }

    public void setPrograma(String programa) {
        this.programa = programa;
    }
}
```

---

## 8. Persistencia Declarativa: `EstudianteRepository.java`

Ubicación: `com.ejemplo.estudiantes.repository.EstudianteRepository`

```java
package com.ejemplo.estudiantes.repository;

import com.ejemplo.estudiantes.model.Estudiante;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface EstudianteRepository extends JpaRepository<Estudiante, Long> {

    // Derived Query: Comprueba existencia ignorando mayúsculas/minúsculas
    boolean existsByCorreoIgnoreCase(String correo);

    // Derived Query: Búsqueda por subcadena insensible a mayúsculas
    List<Estudiante> findByProgramaContainingIgnoreCase(String programa);

    // Consulta personalizada equivalente mediante JPQL
    @Query("SELECT e FROM Estudiante e WHERE LOWER(e.programa) = LOWER(:programa)")
    List<Estudiante> buscarPorProgramaExacto(@Param("programa") String programa);
}
```

---

## 9. Capa de Servicios: `EstudianteService.java`

Ubicación: `com.ejemplo.estudiantes.service.EstudianteService`

```java
package com.ejemplo.estudiantes.service;

import com.ejemplo.estudiantes.exception.RecursoNoEncontradoException;
import com.ejemplo.estudiantes.model.Estudiante;
import com.ejemplo.estudiantes.repository.EstudianteRepository;
import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
public class EstudianteService {

    private final EstudianteRepository repo;

    // Inyección de dependencias mediante constructor
    public EstudianteService(EstudianteRepository repo) {
        this.repo = repo;
    }

    @Transactional(readOnly = true)
    public List<Estudiante> listar() {
        return repo.findAll();
    }

    @Transactional(readOnly = true)
    public Estudiante consultar(Long id) {
        return repo.findById(id)
                .orElseThrow(() -> new RecursoNoEncontradoException("Estudiante no encontrado con ID: " + id));
    }

    @Transactional
    public Estudiante crear(Estudiante nuevo) {
        nuevo.setId(null); // Asegura que la base de datos asigne el identificador
        if (repo.existsByCorreoIgnoreCase(nuevo.getCorreo())) {
            throw new DataIntegrityViolationException("Ya existe un estudiante registrado con el correo: " + nuevo.getCorreo());
        }
        return repo.saveAndFlush(nuevo);
    }

    @Transactional
    public Estudiante actualizar(Long id, Estudiante datos) {
        Estudiante actual = consultar(id);
        
        // Si el correo cambia, validar unicidad
        if (!actual.getCorreo().equalsIgnoreCase(datos.getCorreo()) 
                && repo.existsByCorreoIgnoreCase(datos.getCorreo())) {
            throw new DataIntegrityViolationException("El correo ya está en uso por otro registro");
        }

        actual.setNombre(datos.getNombre());
        actual.setApellido(datos.getApellido());
        actual.setCorreo(datos.getCorreo());
        actual.setPrograma(datos.getPrograma());

        return repo.saveAndFlush(actual);
    }

    @Transactional
    public void eliminar(Long id) {
        Estudiante actual = consultar(id);
        repo.delete(actual);
    }

    @Transactional(readOnly = true)
    public List<Estudiante> buscarPorPrograma(String programa) {
        return repo.findByProgramaContainingIgnoreCase(programa);
    }
}
```

---

## 10. Gestión Centralizada de Excepciones

### Excepción Personalizada
Ubicación: `com.ejemplo.estudiantes.exception.RecursoNoEncontradoException`

```java
package com.ejemplo.estudiantes.exception;

public class RecursoNoEncontradoException extends RuntimeException {
    public RecursoNoEncontradoException(String mensaje) {
        super(mensaje);
    }
}
```

### Manejador Global con `@RestControllerAdvice`
Ubicación: `com.ejemplo.estudiantes.exception.ApiExceptionHandler`

```java
package com.ejemplo.estudiantes.exception;

import org.springframework.dao.DataIntegrityViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class ApiExceptionHandler {

    @ExceptionHandler(RecursoNoEncontradoException.class)
    public ResponseEntity<Map<String, Object>> manejarNoEncontrado(RecursoNoEncontradoException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of(
                "timestamp", LocalDateTime.now(),
                "status", HttpStatus.NOT_FOUND.value(),
                "error", "Recurso no encontrado",
                "message", ex.getMessage()
        ));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<Map<String, Object>> manejarConflictoIntegridad(DataIntegrityViolationException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT).body(Map.of(
                "timestamp", LocalDateTime.now(),
                "status", HttpStatus.CONFLICT.value(),
                "error", "Conflicto de integridad de datos",
                "message", ex.getMessage()
        ));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> manejarErroresValidacion(MethodArgumentNotValidException ex) {
        Map<String, String> erroresPorCampo = new HashMap<>();
        for (FieldError error : ex.getBindingResult().getFieldErrors()) {
            erroresPorCampo.put(error.getField(), error.getDefaultMessage());
        }

        return ResponseEntity.badRequest().body(Map.of(
                "timestamp", LocalDateTime.now(),
                "status", HttpStatus.BAD_REQUEST.value(),
                "error", "Validación fallida",
                "detalles", erroresPorCampo
        ));
    }
}
```

---

## 11. Endpoints REST: `EstudianteController.java`

Ubicación: `com.ejemplo.estudiantes.controller.EstudianteController`

```java
package com.ejemplo.estudiantes.controller;

import com.ejemplo.estudiantes.model.Estudiante;
import com.ejemplo.estudiantes.service.EstudianteService;
import jakarta.validation.Valid;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/estudiantes")
public class EstudianteController {

    private final EstudianteService service;

    public EstudianteController(EstudianteService service) {
        this.service = service;
    }

    @GetMapping
    public List<Estudiante> listar() {
        return service.listar();
    }

    @GetMapping("/{id}")
    public Estudiante consultar(@PathVariable Long id) {
        return service.consultar(id);
    }

    @PostMapping
    public ResponseEntity<Estudiante> crear(@Valid @RequestBody Estudiante estudiante) {
        Estudiante nuevo = service.crear(estudiante);
        return ResponseEntity.status(HttpStatus.CREATED).body(nuevo);
    }

    @PutMapping("/{id}")
    public Estudiante actualizar(@PathVariable Long id, @Valid @RequestBody Estudiante estudiante) {
        return service.actualizar(id, estudiante);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void eliminar(@PathVariable Long id) {
        service.eliminar(id);
    }

    @GetMapping("/buscar")
    public List<Estudiante> buscar(@RequestParam String programa) {
        return service.buscarPorPrograma(programa);
    }
}
```

---

## 12. Protocolo de Pruebas y Validación

### Ejecución de la Aplicación

```bash
# En sistemas basados en Unix (Linux / macOS):
./mvnw spring-boot:run

# En Windows:
.\mvnw.cmd spring-boot:run
```

### Matriz de Casos de Prueba con Postman / cURL

| Paso | Método HTTP | Ruta de Endpoint | Payload de Entrada (JSON) | Código HTTP Esperado |
| :---: | :--- | :--- | :--- | :---: |
| **1** | `POST` | `/api/estudiantes` | Ver Ejemplo 1 | `201 Created` |
| **2** | `GET` | `/api/estudiantes` | *(Vacío)* | `200 OK` |
| **3** | `GET` | `/api/estudiantes/1` | *(Vacío)* | `200 OK` |
| **4** | `GET` | `/api/estudiantes/buscar?programa=Sistemas` | *(Vacío)* | `200 OK` |
| **5** | `PUT` | `/api/estudiantes/1` | Ver Ejemplo 2 | `200 OK` |
| **6** | `POST` | `/api/estudiantes` (Mismo correo) | Ver Ejemplo 1 | `409 Conflict` |
| **7** | `POST` | `/api/estudiantes` (Sin nombre / Correo no válido) | Ver Ejemplo 3 | `400 Bad Request` |
| **8** | `DELETE` | `/api/estudiantes/1` | *(Vacío)* | `204 No Content` |
| **9** | `GET` | `/api/estudiantes/1` | *(Vacío)* | `404 Not Found` |

#### Payload Ejemplo 1: Creación Válida (`POST`)
```json
{
  "nombre": "Ana",
  "apellido": "Pérez",
  "correo": "ana.perez@universidad.edu",
  "programa": "Ingeniería de Sistemas"
}
```

#### Payload Ejemplo 2: Actualización (`PUT`)
```json
{
  "nombre": "Ana Sofía",
  "apellido": "Pérez Gómez",
  "correo": "ana.perez@universidad.edu",
  "programa": "Ingeniería Industrial"
}
```

#### Payload Ejemplo 3: Petición Inválida para Comprobar Errores (`POST`)
```json
{
  "nombre": "",
  "apellido": "Pérez",
  "correo": "correo-sin-formato-valido",
  "programa": ""
}
```

---

## 13. Diagnóstico de Problemas Comunes (*Troubleshooting*)

```
┌──────────────────────────────────────┬───────────────────────────────────────────┬────────────────────────────────────────────────────────┐
│ Síntoma / Mensaje                    │ Causa Probable                            │ Solución Sugerida                                      │
├──────────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Connection refused / Timeout         │ Firewall escolar/empresarial o puerto 5432│ Usar Session Pooler de Supabase o habilitar salida TCP │
│                                      │ bloqueado. Proyecto en estado 'Paused'.   │ en puerto 5432. Reanudar proyecto en el dashboard.     │
├──────────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Network is unreachable (db.xxx)      │ La red solo soporta IPv4 y se intentó     │ Cambiar el host por el host IPv4 del Session Pooler    │
│                                      │ conectar directo al host IPv6.            │ provisto en el panel Connect de Supabase.              │
├──────────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ password authentication failed       │ Usuario sin sufijo de proyecto            │ Verificar que el usuario tenga el formato              │
│                                      │ (postgres.<REF>) o contraseña errónea.    │ `postgres.[PROJECT_REF]` y redefinir la variable.      │
├──────────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ SchemaManagementException:           │ La tabla no existe o fue creada           │ Ejecutar el DDL en el esquema public en el SQL Editor  │
│ Table [estudiantes] not found        │ en un schema diferente de 'public'.       │ de Supabase con el usuario correcto.                   │
├──────────────────────────────────────┼───────────────────────────────────────────┼────────────────────────────────────────────────────────┤
│ Port 8080 was already in use         │ Otro proceso Java o servicio ocupa el     │ Definir `server.port=8081` en application.properties   │
│                                      │ puerto de red local.                      │ o terminar el proceso bloqueado.                       │
└──────────────────────────────────────┴───────────────────────────────────────────┴────────────────────────────────────────────────────────┘
```

---

## 14. Taller Práctico y Retos Evaluables

* **Actividad 1 — Reproducción y Registro:** Desplegar el proyecto, ejecutar las 9 pruebas de la matriz y capturar el ID generado en la respuesta HTTP junto con la fila persistida en el Table Editor de Supabase.
* **Actividad 2 — Evolución de Esquema:**
  1. Ejecutar en Supabase: `ALTER TABLE public.estudiantes ADD COLUMN telefono VARCHAR(30);`.
  2. Incorporar el atributo correspondiente con validaciones en `Estudiante.java`.
  3. Evaluar qué ocurre si no se actualiza la entidad antes de reiniciar la aplicación con `ddl-auto=validate`.
* **Actividad 3 — Consultas Personalizadas:** Implementar en el controlador y repositorio la búsqueda por apellido con paginación usando la abstracción `Pageable` y el retorno `Page<Estudiante>`.
* **Actividad 4 — Análisis de Concurrencia:** Ejecutar dos peticiones `POST` concurrentes con el mismo correo y analizar cómo interactúan la validación Java `existsByCorreoIgnoreCase` frente al índice `UNIQUE` de PostgreSQL.

---

## 15. Rúbrica de Evaluación (100 Puntos)

| Criterio | Descripción | Puntaje Máximo |
| :--- | :--- | :---: |
| **Arquitectura por Capas** | Desacoplamiento correcto entre `Controller`, `Service`, `Repository` y `Entity` | **20 pts** |
| **Gestión de Credenciales** | Conexión funcional a Supabase sin registrar contraseñas en código fuente | **20 pts** |
| **Operaciones CRUD y HTTP** | Implementación completa de métodos y códigos de estado REST estándar | **25 pts** |
| **Consultas Derivadas / JPQL** | Métodos declarativos funcionales para búsquedas específicas | **15 pts** |
| **Validaciones y Excepciones** | Integración de `@Valid`, mensajes de error descriptivos y captura con Advice | **10 pts** |
| **Sustentación y Evidencias** | Demostración con Postman y consulta equivalente en el SQL Editor de Supabase | **10 pts** |

---

## 16. Fuentes y Referencias Oficiales

* [Spring Boot Official Documentation — Data Access](https://docs.spring.io/spring-boot/reference/data/sql.html)
* [Spring Data JPA Reference Documentation](https://docs.spring.io/spring-data/jpa/reference/)
* [Supabase Docs — Connecting to PostgreSQL](https://supabase.com/docs/guides/database/connecting-to-postgres)
* [PostgreSQL 16 Documentation — Constraints](https://www.postgresql.org/docs/current/ddl-constraints.html)
