# Guía Práctica: Spring Data JPA, Supabase (PostgreSQL) y Migraciones con Flyway

Esta guía documenta la integración de una base de datos **Supabase (PostgreSQL)** en un proyecto **Spring Boot 3** utilizando **Spring Data JPA**, modelado de relaciones (`@OneToMany`, `@ManyToMany`) y control de versiones del esquema con **Flyway**.

---

## 1. Obtención de Credenciales en Supabase

Supabase provee un motor PostgreSQL administrado. Para conectarse mediante JDBC:

1. Ingresa a la consola de Supabase y selecciona tu proyecto.
2. Navega a **Project Settings** > **Database**.
3. En la sección **Connection parameters** / **Connection string** (modo *Direct* en puerto `5432` o *Transaction Pooler* en puerto `6543`), recopila:
   - **Host:** ej. `db.[PROJECT_REF].supabase.co`
   - **Port:** `5432`
   - **Database name:** `postgres` (por defecto)
   - **User:** `postgres`
   - **Password:** Contraseña definida al crear el proyecto

> **Requisito obligatorio:** Las conexiones entrantes a Supabase exigen cifrado SSL. Por ende, la URL de JDBC debe contener el parámetro `?sslmode=require`.

---

## 2. Dependencias del Proyecto (`pom.xml`)

Agrega las dependencias requeridas en el archivo `pom.xml`.

```xml
<dependencies>
    <!-- Spring Data JPA -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- PostgreSQL JDBC Driver -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Spring Boot Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Flyway Core -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>

    <!-- Módulo específico de PostgreSQL para Flyway (Flyway 9+) -->
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-database-postgresql</artifactId>
    </dependency>

    <!-- Lombok -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- Spring Boot Starter Test -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 3. Configuración (`application.yml`)

Ubicado en `src/main/resources/application.yml`. 

> **Importante:** Se define `spring.jpa.hibernate.ddl-auto: validate` para delegar la creación y modificación de tablas enteramente a Flyway, previniendo que Hibernate altere el esquema automáticamente.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${SUPABASE_DB_HOST:db.xxxxxx.supabase.co}:5432/${SUPABASE_DB_NAME:postgres}?sslmode=require
    username: ${SUPABASE_DB_USER:postgres}
    password: ${SUPABASE_DB_PASSWORD:tu_password_aqui}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      idle-timeout: 30000
      max-lifetime: 60000
      connection-timeout: 20000

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
        jdbc:
          lob:
            non_contextual_creation: true

  flyway:
    enabled: true
    baseline-on-migrate: true
    baseline-version: "0"
    locations: classpath:db/migration
    schemas: public
    table: flyway_schema_history
```

---

## 4. Versionado de Base de Datos con Flyway

Crea la carpeta de scripts en: `src/main/resources/db/migration/`.

### Estructura de nombres
La convención de nomenclatura estándar es:
`V<Numero_Version>__<Descripcion_con_guiones_bajos>.sql` (con doble guion bajo `__`).

```text
src/main/resources/db/migration/
├── V1__create_categories_and_products.sql
├── V2__create_tags_and_join_table.sql
└── V3__insert_seed_data.sql
```

### Script `V1__create_categories_and_products.sql`
```sql
CREATE TABLE categories (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    stock INTEGER NOT NULL DEFAULT 0,
    category_id BIGINT,
    created_at TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT fk_products_category
        FOREIGN KEY (category_id) 
        REFERENCES categories(id) 
        ON DELETE RESTRICT
);

CREATE INDEX idx_products_category_id ON products(category_id);
CREATE INDEX idx_products_name ON products(name);
```

### Script `V2__create_tags_and_join_table.sql`
```sql
CREATE TABLE tags (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE
);

CREATE TABLE product_tags (
    product_id BIGINT NOT NULL,
    tag_id BIGINT NOT NULL,
    CONSTRAINT pk_product_tags PRIMARY KEY (product_id, tag_id),
    CONSTRAINT fk_pt_product 
        FOREIGN KEY (product_id) 
        REFERENCES products(id) 
        ON DELETE CASCADE,
    CONSTRAINT fk_pt_tag 
        FOREIGN KEY (tag_id) 
        REFERENCES tags(id) 
        ON DELETE CASCADE
);

CREATE INDEX idx_pt_product_id ON product_tags(product_id);
CREATE INDEX idx_pt_tag_id ON product_tags(tag_id);
```

### Script `V3__insert_seed_data.sql`
```sql
INSERT INTO categories (name) VALUES 
('Hardware'),
('Accesorios');

INSERT INTO tags (name) VALUES 
('Gamer'),
('Oficina'),
('Wireless');

INSERT INTO products (name, price, stock, category_id) VALUES 
('Teclado Mecánico', 79.99, 20, 1),
('Mouse Inalámbrico', 34.50, 45, 2);

INSERT INTO product_tags (product_id, tag_id) VALUES 
(1, 1),
(2, 2),
(2, 3);
```

---

## 5. Modelado de Entidades JPA

### 5.1 Entidad `Category`
```java
package com.ejemplo.supabase.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import jakarta.persistence.*;
import lombok.*;

import java.time.LocalDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "categories")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Category {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String name;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "category", cascade = CascadeType.ALL)
    @JsonIgnoreProperties("category")
    @Builder.Default
    private List<Product> products = new ArrayList<>();

    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
    }

    public void addProduct(Product product) {
        products.add(product);
        product.setCategory(this);
    }

    public void removeProduct(Product product) {
        products.remove(product);
        product.setCategory(null);
    }
}
```

### 5.2 Entidad `Tag`
```java
package com.ejemplo.supabase.model;

import com.fasterxml.jackson.annotation.JsonIgnore;
import jakarta.persistence.*;
import lombok.*;

import java.util.HashSet;
import java.util.Objects;
import java.util.Set;

@Entity
@Table(name = "tags")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Tag {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String name;

    @ManyToMany(mappedBy = "tags")
    @JsonIgnore
    @Builder.Default
    private Set<Product> products = new HashSet<>();

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Tag tag)) return false;
        return Objects.equals(name, tag.name);
    }

    @Override
    public int hashCode() {
        return Objects.hashCode(name);
    }
}
```

### 5.3 Entidad `Product`
```java
package com.ejemplo.supabase.model;

import com.fasterxml.jackson.annotation.JsonIgnoreProperties;
import jakarta.persistence.*;
import lombok.*;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.HashSet;
import java.util.Set;

@Entity
@Table(name = "products")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(nullable = false)
    private Integer stock;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id")
    @JsonIgnoreProperties("products")
    private Category category;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "product_tags",
        joinColumns = @JoinColumn(name = "product_id"),
        inverseJoinColumns = @JoinColumn(name = "tag_id")
    )
    @Builder.Default
    private Set<Tag> tags = new HashSet<>();

    @PrePersist
    public void prePersist() {
        this.createdAt = LocalDateTime.now();
    }

    public void addTag(Tag tag) {
        tags.add(tag);
        tag.getProducts().add(this);
    }

    public void removeTag(Tag tag) {
        tags.remove(tag);
        tag.getProducts().remove(this);
    }
}
```

---

## 6. Repositorio con Prevención de Consultas $N+1$

En `ProductRepository.java` se implementan consultas derivadas, `@EntityGraph` y JPQL con `JOIN FETCH` para evitar el problema de rendimiento $N+1$ en colecciones perezosas (`LAZY`):

```java
package com.ejemplo.supabase.repository;

import com.ejemplo.supabase.model.Product;
import org.springframework.data.jpa.repository.EntityGraph;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @EntityGraph(attributePaths = {"category", "tags"})
    List<Product> findAll();

    @Query("SELECT DISTINCT p FROM Product p " +
           "LEFT JOIN FETCH p.category " +
           "LEFT JOIN FETCH p.tags " +
           "WHERE p.id = :id")
    Optional<Product> findByIdWithDetails(@Param("id") Long id);

    List<Product> findByCategoryId(Long categoryId);

    List<Product> findByTags_Name(String tagName);
}
```

---

## 7. Capa de Servicio y Controlador

### 7.1 Servicio (`ProductService.java`)
```java
package com.ejemplo.supabase.service;

import com.ejemplo.supabase.model.Category;
import com.ejemplo.supabase.model.Product;
import com.ejemplo.supabase.model.Tag;
import com.ejemplo.supabase.repository.CategoryRepository;
import com.ejemplo.supabase.repository.ProductRepository;
import com.ejemplo.supabase.repository.TagRepository;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.util.List;
import java.util.Set;

@Service
public class ProductService {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;
    private final TagRepository tagRepository;

    public ProductService(ProductRepository productRepository,
                          CategoryRepository categoryRepository,
                          TagRepository tagRepository) {
        this.productRepository = productRepository;
        this.categoryRepository = categoryRepository;
        this.tagRepository = tagRepository;
    }

    @Transactional(readOnly = true)
    public List<Product> getAll() {
        return productRepository.findAll();
    }

    @Transactional(readOnly = true)
    public Product getById(Long id) {
        return productRepository.findByIdWithDetails(id)
                .orElseThrow(() -> new RuntimeException("Producto no encontrado: " + id));
    }

    @Transactional
    public Product create(String name, BigDecimal price, Integer stock, Long categoryId, Set<String> tagNames) {
        Category category = categoryRepository.findById(categoryId)
                .orElseThrow(() -> new RuntimeException("Categoría no encontrada: " + categoryId));

        Product product = Product.builder()
                .name(name)
                .price(price)
                .stock(stock)
                .category(category)
                .build();

        if (tagNames != null) {
            for (String tagName : tagNames) {
                Tag tag = tagRepository.findByName(tagName)
                        .orElseGet(() -> tagRepository.save(Tag.builder().name(tagName).build()));
                product.addTag(tag);
            }
        }

        return productRepository.save(product);
    }
}
```

### 7.2 Controlador REST (`ProductController.java`)
```java
package com.ejemplo.supabase.controller;

import com.ejemplo.supabase.model.Product;
import com.ejemplo.supabase.service.ProductService;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.math.BigDecimal;
import java.util.List;
import java.util.Set;

@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    @GetMapping
    public ResponseEntity<List<Product>> getAll() {
        return ResponseEntity.ok(productService.getAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Product> getById(@PathVariable Long id) {
        return ResponseEntity.ok(productService.getById(id));
    }

    public record CreateProductRequest(String name, BigDecimal price, Integer stock, Long categoryId, Set<String> tags) {}

    @PostMapping
    public ResponseEntity<Product> create(@RequestBody CreateProductRequest req) {
        Product created = productService.create(req.name(), req.price(), req.stock(), req.categoryId(), req.tags());
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

---

## 8. Verificación y Reglas de Producción

1. **Tabla de Historial:** Al ejecutar la aplicación, verifica en el SQL Editor de Supabase:
   ```sql
   SELECT * FROM flyway_schema_history ORDER BY installed_rank;
   ```
2. **Inmutabilidad de Scripts:** Nunca edites un archivo `V*.sql` ya ejecutado en Supabase; creará un fallo de checksum (`Migration checksum mismatch`). Todo cambio nuevo debe crearse como un archivo incremental (`V4__...sql`).
3. **Row Level Security (RLS):** La conexión directa por JDBC mediante el usuario `postgres` opera con permisos de administrador en Supabase, por lo que las políticas de RLS de la API de Supabase son omitidas a nivel de Spring Boot.