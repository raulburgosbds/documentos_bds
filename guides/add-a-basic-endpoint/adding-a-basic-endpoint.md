# Especificación de Implementación: Endpoint de Certificaciones

Este documento define las especificaciones técnicas y el procedimiento detallado para la implementación del recurso de **Certificaciones** en el microservicio `people-center`.

---

## Tabla de Contenidos

1. [Estructura del Proyecto](#estructura-del-proyecto)
2. [Modelo de Datos](#modelo-de-datos)
   - [Diagrama de Relaciones](#diagrama-de-relaciones)
   - [Diagrama Entidad-Relación](#diagrama-entidad-relación)
   - [Cardinalidad y Relaciones](#cardinalidad-y-relaciones)
3. [Procedimiento de Implementación](#procedimiento-de-implementación)
   - [1. Migración de Esquema de Base de Datos](#1-migración-de-esquema-de-base-de-datos)
   - [2. Definición de Entidades JPA](#2-definición-de-entidades-jpa)
     - [Diagrama de Clases](#diagrama-de-clases)
     - [2.1. Entidad CertificationEntity](#21-entidad-certificationentity)
     - [2.2. Entidad PersonCertificationEntity](#22-entidad-personcertificationentity)
     - [2.3. Actualización de PersonEntity](#23-actualización-de-personentity)
   - [3. Definición de Repositorios](#3-definición-de-repositorios)
     - [3.1. CertificationRepository](#31-certificationrepository)
     - [3.2. PersonCertificationRepository](#32-personcertificationrepository)
   - [4. Definición de Modelos de Transferencia de Datos (DTOs)](#4-definición-de-modelos-de-transferencia-de-datos-dtos)
     - [4.1. Certification.java](#41-certificationjava)
     - [4.2. PersonCertification.java](#42-personcertificationjava)
     - [4.3. CreatePersonCertificationRequest.java](#43-createpersoncertificationrequestjava)
   - [5. Actualización de la Librería Compartida](#5-actualización-de-la-librería-compartida)
     - [5.1. Definición de Constantes de Ruta](#51-definición-de-constantes-de-ruta)
     - [5.2. Justificación Arquitectónica](#52-justificación-arquitectónica)
     - [5.3. Ejemplo de Implementación en Clientes](#53-ejemplo-de-implementación-en-clientes)
     - [5.4. Extensiones Potenciales de la Librería](#54-extensiones-potenciales-de-la-librería)
       - [A. Inclusión en DTO Person](#a-inclusión-en-dto-person)
       - [B. Enum CertificationType](#b-enum-certificationtype)
       - [C. Cliente HTTP (PeopleCenterClient)](#c-cliente-http-peoplecenterclient)
       - [Matriz de Decisión: Extensiones Opcionales](#matriz-de-decisión-extensiones-opcionales)
   - [6. Implementación del Mapper](#6-implementación-del-mapper)
   - [7. Implementación de la Capa de Servicio](#7-implementación-de-la-capa-de-servicio)
     - [7.1. Implementación Concreta](#71-implementación-concreta)
   - [8. Implementación del Controlador REST](#8-implementación-del-controlador-rest)
   - [9. Manejo de Excepciones](#9-manejo-de-excepciones)
   - [10. Verificación y Pruebas](#10-verificación-y-pruebas)
     - [10.1. Pruebas Unitarias (Service)](#101-pruebas-unitarias-service)
     - [10.2. Pruebas de Integración (Controller)](#102-pruebas-de-integración-controller)
4. [Conclusión](#conclusión)

---

## Estructura del Proyecto

El proyecto adhiere a la arquitectura en capas estándar de Spring Boot:

```text
microservice/src/main/java/ar/com/bds/people/center/
├── controller/     # Controladores REST
├── service/        # Lógica de negocio
│   └── impl/       # Implementaciones de servicios
├── repository/     # Acceso a datos (JPA)
├── entity/         # Entidades JPA
├── mapper/         # Mappers (MapStruct)
└── exception/      # Excepciones personalizadas
```

---

## Modelo de Datos

### Diagrama de Relaciones

El módulo de certificaciones se integra con el modelo de datos existente de People Center de la siguiente manera:

```mermaid
erDiagram
    people ||--o{ person_certification : "tiene asignadas"
    certification ||--o{ person_certification : "clasifica"

    people {
        bigint id PK
        string type
        datetime created_at
        datetime updated_at
    }

    certification {
        int id PK
        string code UK
        string name
        datetime created_at
        datetime deleted_at
    }

    person_certification {
        int id PK
        bigint person_id FK
        int certification_id FK
        string url
        decimal percentage
        decimal aliquot
        datetime start_date
        datetime end_date
        datetime created_at
        datetime deleted_at
    }
```

### Diagrama Entidad-Relación

![DER People Center](./der_people_center.png)

La entidad `person_certification` actúa como tabla de vinculación con atributos adicionales entre:

- **people**: Entidad principal (mediante `person_id`).
- **certification**: Catálogo maestro (mediante `certification_id`).

### Cardinalidad y Relaciones

- **people (1) → (N) person_certification**: Una persona puede poseer múltiples certificaciones.
- **certification (1) → (N) person_certification**: Un tipo de certificación puede estar asociado a múltiples personas.
- **person_certification**: Representa la instancia concreta de una certificación para una persona específica.

> [!NOTE]
> La tabla `certification` constituye un catálogo maestro que define los tipos de certificaciones disponibles (ej: CERT_IVA, CERT_GANANCIAS), mientras que `person_certification` almacena los datos específicos de la asignación (URL del documento, vigencia, alícuotas, etc.).

---

## Procedimiento de Implementación

### 1. Migración de Esquema de Base de Datos

**Ubicación:** `microservice/src/main/resources/db/migration/`

**Archivo:** `V25__create_certification_tables.sql` (el número de versión debe ser incremental).

**Definición SQL:**

```sql
-- Tabla: certification
CREATE TABLE IF NOT EXISTS `certification`
(
    `id`         INT          NOT NULL AUTO_INCREMENT,
    `code`       VARCHAR(50)  NOT NULL,
    `name`       VARCHAR(100) NOT NULL,
    `created_at` DATETIME     NOT NULL,
    `deleted_at` DATETIME     NULL,
    PRIMARY KEY (`id`),
    UNIQUE KEY `code_unique` (`code`)
);

-- Tabla: person_certification
CREATE TABLE IF NOT EXISTS `person_certification`
(
    `id`               INT            NOT NULL AUTO_INCREMENT,
    `person_id`        BIGINT         NOT NULL,
    `url`              VARCHAR(250)   NOT NULL,
    `certification_id` INT            NOT NULL,
    `percentage`       DECIMAL(5, 4)  NULL,
    `aliquot`          DECIMAL(5, 4)  NULL,
    `start_date`       DATETIME       NULL,
    `end_date`         DATETIME       NULL,
    `created_at`       DATETIME       NOT NULL,
    `deleted_at`       DATETIME       NULL,
    PRIMARY KEY (`id`),
    KEY `person_id_certification_idx` (`person_id`),
    KEY `certification_id_idx` (`certification_id`),
    CONSTRAINT `person_id_certification_fk` FOREIGN KEY (`person_id`) REFERENCES `people` (`id`),
    CONSTRAINT `certification_id_fk` FOREIGN KEY (`certification_id`) REFERENCES `certification` (`id`)
);

-- Datos semilla iniciales (sujetos a definición final)
INSERT INTO `certification` (`code`, `name`, `created_at`)
VALUES ('CERT_IVA', 'Certificación IVA', NOW()),
    ('CERT_GANANCIAS', 'Certificación Ganancias', NOW());
```

> [!NOTE]
> Para este caso, los códigos y nombres de las certificaciones deben alinearse con los requerimientos definidos por el área correspondiente del banco, puesto que actualmente no se tiene definidos se usaron semmillas iniciales aleatorias.

---

### 2. Definición de Entidades JPA

#### Diagrama de Clases

```mermaid
classDiagram
    direction LR
    class PersonEntity {
        +Long id
        +Set~PersonCertificationEntity~ certifications
    }
    class CertificationEntity {
        +Integer id
        +String code
        +String name
    }
    class PersonCertificationEntity {
        +Integer id
        +PersonEntity personId
        +CertificationEntity certification
        +String url
        +BigDecimal percentage
        +BigDecimal aliquot
        +ZonedDateTime startDate
        +ZonedDateTime endDate
    }

    PersonEntity "1" *-- "*" PersonCertificationEntity : OneToMany
    PersonCertificationEntity "*" --> "1" CertificationEntity : ManyToOne
```

#### 2.1. Entidad `CertificationEntity`

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/entity/CertificationEntity.java`

```java
package ar.com.bds.people.center.entity;

import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.Where;

import javax.persistence.*;
import java.time.ZonedDateTime;

@Getter
@Setter
@Entity
@Table(name = "certification")
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Where(clause = "deleted_at IS NULL")
public class CertificationEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "code", nullable = false, unique = true, length = 50)
    private String code;

    @Column(name = "name", nullable = false, length = 100)
    private String name;

    @Column(name = "created_at", updatable = false, nullable = false)
    @CreationTimestamp
    private ZonedDateTime createdAt;

    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;
}
```

#### 2.2. Entidad `PersonCertificationEntity`

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/entity/PersonCertificationEntity.java`

```java
package ar.com.bds.people.center.entity;

import com.fasterxml.jackson.annotation.JsonBackReference;
import lombok.*;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.Where;

import javax.persistence.*;
import java.math.BigDecimal;
import java.time.ZonedDateTime;

@Getter
@Setter
@Entity
@Table(name = "person_certification", indexes = {
    @Index(name = "person_id_certification_idx", columnList = "person_id"),
    @Index(name = "certification_id_idx", columnList = "certification_id")
})
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Where(clause = "deleted_at IS NULL")
public class PersonCertificationEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @JoinColumn(name = "person_id", nullable = false)
    @ManyToOne
    @JsonBackReference
    private PersonEntity personId;

    @Column(name = "url", nullable = false, length = 250)
    private String url;

    @JoinColumn(name = "certification_id", nullable = false)
    @ManyToOne
    private CertificationEntity certification;

    @Column(name = "percentage", precision = 5, scale = 4)
    private BigDecimal percentage;

    @Column(name = "aliquot", precision = 5, scale = 4)
    private BigDecimal aliquot;

    @Column(name = "start_date")
    private ZonedDateTime startDate;

    @Column(name = "end_date")
    private ZonedDateTime endDate;

    @Column(name = "created_at", updatable = false, nullable = false)
    @CreationTimestamp
    private ZonedDateTime createdAt;

    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;
}
```

#### 2.3. Actualización de `PersonEntity`

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/entity/PersonEntity.java`

Se requiere agregar la relación `@OneToMany` en `PersonEntity` para habilitar el acceso a las certificaciones:

```java
@Entity
@Table(name = "people")
public class PersonEntity implements HasCreatedAndUpdated {
    
    public static final String PERSON_ID_COLUMN_NAME = "person_id";
    
    @Id
    private Long id;
    
    // ... otras relaciones existentes ...
    
    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = PERSON_ID_COLUMN_NAME)
    private Set<TaxpayersProfileEntity> taxpayersProfiles;
    
    // Relación con Certificaciones
    @OneToMany(cascade = CascadeType.ALL)
    @JoinColumn(name = PERSON_ID_COLUMN_NAME)
    private Set<PersonCertificationEntity> certifications;
    
    // ... resto de las relaciones ...
}
```

> [!IMPORTANT]
> La inclusión de esta relación en `PersonEntity` es recomendable para mantener la consistencia del modelo de dominio y permitir la navegación bidireccional, aunque el acceso principal se realice a través de los repositorios específicos.

---

### 3. Definición de Repositorios

#### 3.1. `CertificationRepository`

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/repository/CertificationRepository.java`

```java
package ar.com.bds.people.center.repository;

import ar.com.bds.people.center.entity.CertificationEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public interface CertificationRepository extends JpaRepository<CertificationEntity, Integer> {
    Optional<CertificationEntity> findByCode(String code);
}
```

#### 3.2. `PersonCertificationRepository`

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/repository/PersonCertificationRepository.java`

```java
package ar.com.bds.people.center.repository;

import ar.com.bds.people.center.entity.PersonCertificationEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.ZonedDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface PersonCertificationRepository extends JpaRepository<PersonCertificationEntity, Integer> {
    
    Optional<PersonCertificationEntity> findByIdAndPersonId_Id(Integer id, Long personId);
    
    @Query("SELECT pc FROM PersonCertificationEntity pc " +
        "WHERE pc.personId.id = :personId " +
        "AND pc.deletedAt IS NULL " +
        "AND (pc.startDate IS NULL OR pc.startDate <= :now) " +
        "AND (pc.endDate IS NULL OR pc.endDate >= :now)")
    List<PersonCertificationEntity> findValidCertifications(
        @Param("personId") Long personId, 
        @Param("now") ZonedDateTime now
    );
}
```

---

### 4. Definición de Modelos de Transferencia de Datos (DTOs)

> [!IMPORTANT]
> Los DTOs deben definirse en el módulo `library/model` para asegurar su disponibilidad para los clientes del servicio.

**Estructura de ubicación de DTOs:**

```
library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/
├── Certification.java              # DTO de respuesta (catálogo)
├── PersonCertification.java        # DTO de respuesta (recurso principal)
└── requests/
    └── CreatePersonCertificationRequest.java  # DTO de request (creación)
```

> [!NOTE]
> **Convención de ubicación:**
>
> - **DTOs de Respuesta**: Se ubican directamente en `model/` (ej: `Certification.java`, `PersonCertification.java`).
> - **DTOs de Request**: Se ubican en el subdirectorio `model/requests/` (ej: `CreatePersonCertificationRequest.java`).
> - Esta convención facilita la distinción entre modelos de entrada y salida de la API.

**Ubicación base:** `library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/`

#### 4.1. `Certification.java`

```java
package ar.com.bds.lib.peoplecenter.model;

import lombok.*;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Certification {
    private String code;
    private String name;
}
```

#### 4.2. `PersonCertification.java`

```java
package ar.com.bds.lib.peoplecenter.model;

import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.*;

import java.math.BigDecimal;
import java.time.ZonedDateTime;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PersonCertification {
    private Integer id;
    private String url;
    private Certification certification;
    private BigDecimal aliquot;
    private BigDecimal percentage;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime startDate;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime endDate;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime createdAt;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime deletedAt;
}
```

#### 4.3. `CreatePersonCertificationRequest.java`

**Ubicación:** `library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/requests/`

```java
package ar.com.bds.lib.peoplecenter.model.requests;

import com.fasterxml.jackson.annotation.JsonFormat;
import lombok.*;

import javax.validation.constraints.DecimalMax;
import javax.validation.constraints.DecimalMin;
import javax.validation.constraints.NotBlank;
import java.math.BigDecimal;
import java.time.ZonedDateTime;

@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CreatePersonCertificationRequest {
    
    @NotBlank(message = "URL is required")
    private String url;
    
    @NotBlank(message = "Certification code is required")
    private String certificationCode;
    
    @DecimalMin(value = "0.0", inclusive = true, message = "Aliquot must be >= 0")
    @DecimalMax(value = "1.0", inclusive = true, message = "Aliquot must be <= 1")
    private BigDecimal aliquot;
    
    @DecimalMin(value = "0.0", inclusive = true, message = "Percentage must be >= 0")
    @DecimalMax(value = "1.0", inclusive = true, message = "Percentage must be <= 1")
    private BigDecimal percentage;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime startDate;
    
    @JsonFormat(shape = JsonFormat.Shape.STRING, pattern = "yyyy-MM-dd'T'HH:mm:ss.SSSXXX")
    private ZonedDateTime endDate;
}
```

---

### 5. Actualización de la Librería Compartida

Para facilitar el consumo del nuevo endpoint por otros microservicios, se debe actualizar la clase de constantes `PathV2.java` en la `library`.

#### 5.1. Definición de Constantes de Ruta

**Ubicación:** `library/model/src/main/java/ar/com/bds/lib/peoplecenter/api/PathV2.java`

Se debe agregar la constante `CERTIFICATIONS_PATH` respetando el orden alfabético:

```java
package ar.com.bds.lib.peoplecenter.api;

import lombok.experimental.UtilityClass;

@UtilityClass
public final class PathV2 {

    public static final String CUSTOMER_PATH = "/v2/customers";
    private static final String BASE_PATH = "/v2/people";

    public static final String ACCEPTED_TERMS_PATH = BASE_PATH + "/{id}/accepted-terms";
    public static final String ADDRESS_PATH = BASE_PATH + "/{id}/addresses";
    
    // Constante para el recurso de certificaciones
    public static final String CERTIFICATIONS_PATH = BASE_PATH + "/{id}/certifications";
    
    public static final String CHANNELS_PATH = BASE_PATH + "/{id}/channels";
    public static final String CONTACTS_PATH = BASE_PATH + "/{id}/contacts";
    // ... resto de constantes ...
}
```

#### 5.2. Justificación Arquitectónica

El uso de constantes para las rutas proporciona los siguientes beneficios:

1. **Consistencia:** Garantiza que todos los clientes utilicen exactamente la misma ruta definida.
2. **Mantenibilidad:** Centraliza la definición de la ruta, facilitando futuras modificaciones.
3. **Seguridad de Tipos:** Reduce la probabilidad de errores tipográficos en las cadenas de URL.

#### 5.3. Ejemplo de Implementación en Clientes

```java
// Ejemplo en microservicio consumidor
public List<PersonCertification> getPersonCertifications(Long personId) {
    // Uso de la constante centralizada
    String url = baseUrl + PathV2.CERTIFICATIONS_PATH
        .replace("{id}", personId.toString());
    
    return restTemplate.exchange(
        url,
        HttpMethod.GET,
        null,
        new ParameterizedTypeReference<List<PersonCertification>>() {}
    ).getBody();
}
```

#### 5.4. Extensiones Potenciales de la Librería

Dependiendo de los requerimientos específicos de integración, se pueden considerar las siguientes extensiones opcionales:

##### A. Inclusión en DTO `Person`

**Ubicación:** `library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/Person.java`

El DTO `Person` contiene listas de todos los sub-recursos (addresses, contacts, documents, etc.). Se puede agregar la lista de certificaciones para mantener consistencia:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Person implements HasId {

// ... campos existentes ...

@Valid
@Builder.Default
private List<TaxpayersProfile> taxpayersProfiles = new ArrayList<>();

// Agregar esta nueva lista (OPCIONAL)
@Valid
@Builder.Default
private List<PersonCertification> certifications = new ArrayList<>();

@Valid
@Builder.Default
private List<Document> documents = new ArrayList<>();

// ... resto de campos ...
}
```

**Criterios de Implementación:**

```mermaid
flowchart TD
A[Evaluar Necesidad de<br/>Certificaciones en Person] --> B{GET /people incluye<br/>certificaciones?}
B -->|Sí| C[IMPLEMENTAR]
B -->|No| D{Acceso solo vía<br/>/certifications?}
D -->|Sí| E[NO IMPLEMENTAR]
D -->|No| F{Otros servicios<br/>requieren datos completos?}
F -->|Sí| C
F -->|No| G{Mantener consistencia<br/>con sub-recursos?}
G -->|Sí| C
G -->|No| E

style C fill:#164503
style E fill:#D20103
```

**Análisis de Impacto:**

- **Ventaja:** Datos completos disponibles en una sola llamada HTTP
- **Consideración:** Incremento en el tamaño del payload en `GET /people/{id}`

##### B. Enum `CertificationType`

**Ubicación:** `library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/enums/CertificationType.java`

Si los códigos de certificación son fijos y conocidos, se recomienda crear un enum:

```java
package ar.com.bds.lib.peoplecenter.model.enums;

public enum CertificationType {
CERT_IVA("Certificación IVA"),
CERT_GANANCIAS("Certificación Ganancias"),
CERT_INGRESOS_BRUTOS("Certificación Ingresos Brutos"),
CERT_SUSS("Certificación SUSS");

private final String description;

CertificationType(String description) {
    this.description = description;
}

public String getDescription() {
    return description;
}
}
```

**Criterios de Implementación:**

```mermaid
flowchart TD
A[Evaluar Necesidad de<br/>CertificationType Enum] --> B{Códigos son<br/>fijos?}
B -->|Sí| C[IMPLEMENTAR]
B -->|No| D{Códigos dinámicos<br/>desde BD?}
D -->|Sí| E[NO IMPLEMENTAR<br/>Usar String]
D -->|No| F{Requiere validación<br/>estricta?}
F -->|Sí| C
F -->|No| G{Códigos cambian<br/>frecuentemente?}
G -->|Sí| E
G -->|No| C

style C fill:#164503
style E fill:#D20103
```

**Análisis de Impacto:**

- **Ventaja:** Type-safety, autocompletado en IDEs
- **Consideración:** Menor flexibilidad, requiere recompilación para nuevos tipos

##### C. Cliente HTTP (`PeopleCenterClient`)

**Ubicación:** `library/client/src/main/java/ar/com/bds/lib/peoplecenter/client/PeopleCenterClient.java`

Si múltiples microservicios requieren consumir el endpoint, se recomienda agregar métodos al cliente HTTP compartido:

```java
package ar.com.bds.lib.peoplecenter.client;

import ar.com.bds.lib.peoplecenter.api.PathV2;
import ar.com.bds.lib.peoplecenter.model.PersonCertification;
import ar.com.bds.lib.peoplecenter.model.requests.CreatePersonCertificationRequest;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

import java.util.List;

@Component
public class PeopleCenterClient {

private final RestTemplate restTemplate;
private final String baseUrl;

// ... constructor y otros métodos ...

// Métodos para gestión de certificaciones (OPCIONAL)

public Integer createCertification(Long personId, CreatePersonCertificationRequest request) {
    String url = baseUrl + PathV2.CERTIFICATIONS_PATH.replace("{id}", personId.toString());
    return restTemplate.postForObject(url, request, Integer.class);
}

public List<PersonCertification> getCertifications(Long personId) {
    String url = baseUrl + PathV2.CERTIFICATIONS_PATH.replace("{id}", personId.toString());
    return restTemplate.exchange(
        url,
        HttpMethod.GET,
        null,
        new ParameterizedTypeReference<List<PersonCertification>>() {}
    ).getBody();
}

public PersonCertification getCertificationById(Long personId, Integer certificationId) {
    String url = baseUrl + PathV2.CERTIFICATIONS_PATH.replace("{id}", personId.toString()) 
        + "/" + certificationId;
    return restTemplate.getForObject(url, PersonCertification.class);
}

public void deleteCertification(Long personId, Integer certificationId) {
    String url = baseUrl + PathV2.CERTIFICATIONS_PATH.replace("{id}", personId.toString()) 
        + "/" + certificationId;
    restTemplate.delete(url);
}
}
```

**Criterios de Implementación:**

```mermaid
flowchart TD
A[Evaluar Necesidad de<br/>PeopleCenterClient] --> B{Múltiples servicios<br/>consumirán endpoint?}
B -->|Sí| C[IMPLEMENTAR]
B -->|No| D{Solo un servicio<br/>lo usará?}
D -->|Sí| E[NO IMPLEMENTAR<br/>Cliente propio]
D -->|No| F{Estandarizar<br/>consumo de API?}
F -->|Sí| C
F -->|No| G{Equipos requieren<br/>implementaciones específicas?}
G -->|Sí| E
G -->|No| C

style C fill:#164503
style E fill:#D20103
```

**Análisis de Impacto:**

- **Ventaja:** Código reutilizable, reducción de duplicación
- **Consideración:** Dependencia adicional para los servicios consumidores

##### Matriz de Decisión: Extensiones Opcionales

| Extensión | Prioridad | Criterio de Implementación | Ubicación |
|-----------|-----------|----------------------------|-----------|
| **Person.java** | Media | Certificaciones requeridas en `GET /people/{id}` | `model/Person.java` |
| **CertificationType Enum** | Baja | Códigos fijos con validación estricta | `model/enums/CertificationType.java` |
| **Cliente HTTP** | Baja | Consumo por múltiples microservicios | `client/PeopleCenterClient.java` |

> [!NOTE]
> Se recomienda iniciar la implementación sin estas extensiones opcionales, incorporándolas únicamente cuando se identifique una necesidad concreta documentada en los requerimientos del proyecto.

---

### 6. Implementación del Mapper

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/mapper/PersonCertificationMapper.java`

```java
package ar.com.bds.people.center.mapper;

import ar.com.bds.lib.peoplecenter.model.Certification;
import ar.com.bds.lib.peoplecenter.model.PersonCertification;
import ar.com.bds.people.center.entity.CertificationEntity;
import ar.com.bds.people.center.entity.PersonCertificationEntity;
import org.mapstruct.*;

@Mapper(
    componentModel = MappingConstants.ComponentModel.SPRING,
    nullValueCheckStrategy = NullValueCheckStrategy.ALWAYS
)
public interface PersonCertificationMapper {

    @Mapping(target = "certification", source = "certification")
    PersonCertification toDto(PersonCertificationEntity entity);
    
    Certification certificationToDto(CertificationEntity entity);
}
```

---

### 7. Implementación de la Capa de Servicio

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/service/PersonCertificationService.java`

```java
package ar.com.bds.people.center.service;

import ar.com.bds.lib.peoplecenter.model.requests.CreatePersonCertificationRequest;
import ar.com.bds.lib.peoplecenter.model.PersonCertification;

import java.util.List;

public interface PersonCertificationService {
    Integer create(Long personId, CreatePersonCertificationRequest request);
    List<PersonCertification> getValidCertifications(Long personId);
    PersonCertification getById(Long personId, Integer certificationId);
    Integer delete(Long personId, Integer certificationId);
}
```

#### 7.1. Implementación Concreta

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/service/impl/PersonCertificationServiceImpl.java`

```java
package ar.com.bds.people.center.service.impl;

import ar.com.bds.exception.ResourceNotFoundException;
import ar.com.bds.lib.peoplecenter.model.requests.CreatePersonCertificationRequest;
import ar.com.bds.lib.peoplecenter.model.PersonCertification;
import ar.com.bds.people.center.entity.CertificationEntity;
import ar.com.bds.people.center.entity.PersonCertificationEntity;
import ar.com.bds.people.center.entity.PersonEntity;
import ar.com.bds.people.center.mapper.PersonCertificationMapper;
import ar.com.bds.people.center.repository.CertificationRepository;
import ar.com.bds.people.center.repository.PersonCertificationRepository;
import ar.com.bds.people.center.repository.PeopleCenterRepository;
import ar.com.bds.people.center.service.PersonCertificationService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.ZonedDateTime;
import java.util.List;
import java.util.stream.Collectors;

@Service
@Slf4j
@RequiredArgsConstructor
public class PersonCertificationServiceImpl implements PersonCertificationService {

    private final PersonCertificationRepository personCertificationRepository;
    private final CertificationRepository certificationRepository;
    private final PeopleCenterRepository peopleCenterRepository;
    private final PersonCertificationMapper mapper;

    @Override
    @Transactional
    public Integer create(Long personId, CreatePersonCertificationRequest request) {
        log.info("Creating certification for person: {}", personId);

        // Validaciones
        validateRequest(request);

        // Verificar que la persona existe
        PersonEntity person = peopleCenterRepository.findById(personId)
                .orElseThrow(() -> new ResourceNotFoundException("Person not found with id: " + personId));

        // Verificar que el código de certificación existe y está vigente
        CertificationEntity certification = certificationRepository.findByCode(request.getCertificationCode())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Certification not found with code: " + request.getCertificationCode()));

        // Crear la entidad
        PersonCertificationEntity entity = PersonCertificationEntity.builder()
                .personId(person)
                .url(request.getUrl())
                .certification(certification)
                .percentage(request.getPercentage())
                .aliquot(request.getAliquot())
                .startDate(request.getStartDate())
                .endDate(request.getEndDate())
                .build();

        PersonCertificationEntity saved = personCertificationRepository.save(entity);
        log.info("Certification created with id: {}", saved.getId());

        return saved.getId();
    }

    @Override
    public List<PersonCertification> getValidCertifications(Long personId) {
        log.info("Getting valid certifications for person: {}", personId);

        ZonedDateTime now = ZonedDateTime.now();
        List<PersonCertificationEntity> entities = personCertificationRepository
                .findValidCertifications(personId, now);

        return entities.stream()
                .map(mapper::toDto)
                .collect(Collectors.toList());
    }

    @Override
    public PersonCertification getById(Long personId, Integer certificationId) {
        log.info("Getting certification {} for person: {}", certificationId, personId);

        PersonCertificationEntity entity = personCertificationRepository
                .findByIdAndPersonId_Id(certificationId, personId)
                .orElseThrow(() -> new ResourceNotFoundException(
                        String.format("Certification not found with id %d for person %d", certificationId, personId)));

        return mapper.toDto(entity);
    }

    @Override
    @Transactional
    public Integer delete(Long personId, Integer certificationId) {
        log.info("Deleting certification {} for person: {}", certificationId, personId);

        PersonCertificationEntity entity = personCertificationRepository
                .findByIdAndPersonId_Id(certificationId, personId)
                .orElseThrow(() -> new ResourceNotFoundException(
                        String.format("Certification not found with id %d for person %d", certificationId, personId)));

        entity.setDeletedAt(ZonedDateTime.now());
        personCertificationRepository.save(entity);

        log.info("Certification {} deleted successfully", certificationId);
        return entity.getId();
    }

    private void validateRequest(CreatePersonCertificationRequest request) {
        // Solo se permite un dato a la vez en percentage o aliquot
        if (request.getPercentage() != null && request.getAliquot() != null) {
            throw new IllegalArgumentException("Only one of 'percentage' or 'aliquot' can be specified");
        }

        // La fecha desde debe ser menor a la hasta si se especifica
        if (request.getStartDate() != null && request.getEndDate() != null) {
            if (request.getStartDate().isAfter(request.getEndDate())) {
                throw new IllegalArgumentException("Start date must be before end date");
            }
        }
    }
}
```

---

### 8. Implementación del Controlador REST

**Ubicación:** `microservice/src/main/java/ar/com/bds/people/center/controller/PersonCertificationController.java`

El controlador expone los recursos a través de la API REST, delegando la lógica de negocio a la capa de servicio.

```java
package ar.com.bds.people.center.controller;

import ar.com.bds.lib.peoplecenter.api.PathV2;
import ar.com.bds.lib.peoplecenter.model.requests.CreatePersonCertificationRequest;
import ar.com.bds.lib.peoplecenter.model.PersonCertification;
import ar.com.bds.people.center.service.PersonCertificationService;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import javax.validation.Valid;
import java.util.List;

@RestController
@RequiredArgsConstructor
@Tag(name = "Certificaciones", description = "Gestión de certificaciones impositivas y legales de personas")
public class PersonCertificationController {

    private final PersonCertificationService service;

    @Operation(summary = "Crear certificación", description = "Asigna una nueva certificación a una persona existente")
    @PostMapping(PathV2.CERTIFICATIONS_PATH)
    public ResponseEntity<Integer> create(
            @PathVariable Long id,
            @RequestBody @Valid CreatePersonCertificationRequest request) {
        
        Integer certificationId = service.create(id, request);
        return new ResponseEntity<>(certificationId, HttpStatus.CREATED);
    }

    @Operation(summary = "Listar certificaciones válidas", description = "Obtiene las certificaciones vigentes de una persona")
    @GetMapping(PathV2.CERTIFICATIONS_PATH)
    public ResponseEntity<List<PersonCertification>> getValidCertifications(@PathVariable Long id) {
        return ResponseEntity.ok(service.getValidCertifications(id));
    }

    @Operation(summary = "Obtener certificación por ID", description = "Recupera el detalle de una certificación específica")
    @GetMapping(PathV2.CERTIFICATIONS_PATH + "/{certificationId}")
    public ResponseEntity<PersonCertification> getById(
            @PathVariable Long id,
            @PathVariable Integer certificationId) {
        return ResponseEntity.ok(service.getById(id, certificationId));
    }

    @Operation(summary = "Eliminar certificación", description = "Realiza el borrado lógico de una certificación")
    @DeleteMapping(PathV2.CERTIFICATIONS_PATH + "/{certificationId}")
    public ResponseEntity<Void> delete(
            @PathVariable Long id,
            @PathVariable Integer certificationId) {
        service.delete(id, certificationId);
        return ResponseEntity.noContent().build();
    }
}
```

> [!NOTE]
> **Operaciones no implementadas:** Este controlador **no incluye métodos PATCH o PUT** para actualizar certificaciones existentes. Según los requerimientos de la Historia de Usuario, las certificaciones son inmutables una vez creadas. Si se requiere modificar una certificación, se debe eliminar la existente (soft delete) y crear una nueva. Esta decisión de diseño garantiza la trazabilidad completa del historial de certificaciones.

---

### 9. Manejo de Excepciones

El sistema utiliza un manejador global de excepciones (`@ControllerAdvice`). Para este módulo, se deben considerar las siguientes excepciones estándar:

| Excepción | Código HTTP | Escenario |
|-----------|-------------|-----------|
| `ResourceNotFoundException` | 404 Not Found | Persona o Certificación no encontrada. |
| `MethodArgumentNotValidException` | 400 Bad Request | Error en validación de campos (ej: URL vacía, porcentajes inválidos). |
| `DataIntegrityViolationException` | 409 Conflict | Violación de restricciones de base de datos (ej: código duplicado). |

**Ejemplo de respuesta de error:**

```json
{
"timestamp": "2023-10-27T10:30:00.000Z",
"status": 404,
"error": "Not Found",
"message": "Certificación no encontrada con código: CERT_INEXISTENTE",
"path": "/v2/people/123/certifications"
}
```

---

### 10. Verificación y Pruebas

Para garantizar la calidad de la implementación, se requiere la ejecución de las siguientes pruebas. A continuación se detallan las implementaciones de referencia.

#### 10.1. Pruebas Unitarias (Service)

**Ubicación:** `microservice/src/test/java/ar/com/bds/people/center/service/PersonCertificationServiceTest.java`

```java
package ar.com.bds.people.center.service;

import ar.com.bds.exception.ResourceNotFoundException;
import ar.com.bds.lib.peoplecenter.model.PersonCertification;
import ar.com.bds.lib.peoplecenter.model.requests.CreatePersonCertificationRequest;
import ar.com.bds.people.center.entity.CertificationEntity;
import ar.com.bds.people.center.entity.PersonCertificationEntity;
import ar.com.bds.people.center.entity.PersonEntity;
import ar.com.bds.people.center.mapper.PersonCertificationMapper;
import ar.com.bds.people.center.repository.CertificationRepository;
import ar.com.bds.people.center.repository.PeopleCenterRepository;
import ar.com.bds.people.center.repository.PersonCertificationRepository;
import ar.com.bds.people.center.service.impl.PersonCertificationServiceImpl;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import java.math.BigDecimal;
import java.time.ZonedDateTime;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
@DisplayName("PersonCertificationService - Unit Tests")
class PersonCertificationServiceTest {

    @Mock
    private PersonCertificationRepository personCertificationRepository;
    @Mock
    private CertificationRepository certificationRepository;
    @Mock
    private PeopleCenterRepository peopleCenterRepository;
    @Mock
    private PersonCertificationMapper mapper;

    @InjectMocks
    private PersonCertificationServiceImpl service;

    private PersonEntity mockPerson;
    private CertificationEntity mockCertification;
    private PersonCertificationEntity mockPersonCertification;
    private CreatePersonCertificationRequest mockRequest;

    @BeforeEach
    void setUp() {
        // Setup common test data
        mockPerson = new PersonEntity();
        mockPerson.setId(1L);

        mockCertification = new CertificationEntity();
        mockCertification.setId(100);
        mockCertification.setCode("CERT_IVA");
        mockCertification.setName("Certificación IVA");

        mockPersonCertification = new PersonCertificationEntity();
        mockPersonCertification.setId(500);
        mockPersonCertification.setPersonId(mockPerson);
        mockPersonCertification.setCertification(mockCertification);
        mockPersonCertification.setUrl("http://example.com/cert.pdf");
        mockPersonCertification.setAliquot(new BigDecimal("0.21"));

        mockRequest = CreatePersonCertificationRequest.builder()
                .certificationCode("CERT_IVA")
                .url("http://example.com/doc.pdf")
                .aliquot(new BigDecimal("0.21"))
                .build();
    }

    @Test
    @DisplayName("create() - Debe retornar ID cuando los datos son válidos")
    void create_ShouldReturnId_WhenDataIsValid() {
        // Arrange
        when(peopleCenterRepository.findById(1L)).thenReturn(Optional.of(mockPerson));
        when(certificationRepository.findByCode("CERT_IVA")).thenReturn(Optional.of(mockCertification));
        when(personCertificationRepository.save(any(PersonCertificationEntity.class)))
                .thenReturn(mockPersonCertification);

        // Act
        Integer result = service.create(1L, mockRequest);

        // Assert
        assertNotNull(result);
        assertEquals(500, result);
        verify(personCertificationRepository).save(any(PersonCertificationEntity.class));
        verify(peopleCenterRepository).findById(1L);
        verify(certificationRepository).findByCode("CERT_IVA");
    }

    @Test
    @DisplayName("create() - Debe lanzar ResourceNotFoundException cuando la persona no existe")
    void create_ShouldThrowException_WhenPersonNotFound() {
        // Arrange
        when(peopleCenterRepository.findById(999L)).thenReturn(Optional.empty());

        // Act & Assert
        ResourceNotFoundException exception = assertThrows(
                ResourceNotFoundException.class,
                () -> service.create(999L, mockRequest));

        assertTrue(exception.getMessage().contains("Person not found"));
        verify(personCertificationRepository, never()).save(any());
    }

    @Test
    @DisplayName("create() - Debe lanzar ResourceNotFoundException cuando la certificación no existe")
    void create_ShouldThrowException_WhenCertificationNotFound() {
        // Arrange
        when(peopleCenterRepository.findById(1L)).thenReturn(Optional.of(mockPerson));
        when(certificationRepository.findByCode("INVALID_CODE")).thenReturn(Optional.empty());

        CreatePersonCertificationRequest invalidRequest = CreatePersonCertificationRequest.builder()
                .certificationCode("INVALID_CODE")
                .url("http://example.com/doc.pdf")
                .build();

        // Act & Assert
        ResourceNotFoundException exception = assertThrows(
                ResourceNotFoundException.class,
                () -> service.create(1L, invalidRequest));

        assertTrue(exception.getMessage().contains("Certification not found"));
        verify(personCertificationRepository, never()).save(any());
    }

    @Test
    @DisplayName("getValidCertifications() - Debe retornar lista de certificaciones válidas")
    void getValidCertifications_ShouldReturnList_WhenCertificationsExist() {
        // Arrange
        List<PersonCertificationEntity> entities = Arrays.asList(
                mockPersonCertification,
                mockPersonCertification);
        PersonCertification dto = new PersonCertification();
        dto.setId(500);

        when(personCertificationRepository.findValidCertifications(eq(1L), any(ZonedDateTime.class)))
                .thenReturn(entities);
        when(mapper.toDto(any(PersonCertificationEntity.class))).thenReturn(dto);

        // Act
        List<PersonCertification> result = service.getValidCertifications(1L);

        // Assert
        assertNotNull(result);
        assertEquals(2, result.size());
        verify(personCertificationRepository).findValidCertifications(eq(1L), any(ZonedDateTime.class));
        verify(mapper, times(2)).toDto(any(PersonCertificationEntity.class));
    }

    @Test
    @DisplayName("getById() - Debe retornar certificación cuando existe")
    void getById_ShouldReturnCertification_WhenExists() {
        // Arrange
        PersonCertification dto = new PersonCertification();
        dto.setId(500);

        when(personCertificationRepository.findByIdAndPersonId_Id(500, 1L))
                .thenReturn(Optional.of(mockPersonCertification));
        when(mapper.toDto(mockPersonCertification)).thenReturn(dto);

        // Act
        PersonCertification result = service.getById(1L, 500);

        // Assert
        assertNotNull(result);
        assertEquals(500, result.getId());
        verify(personCertificationRepository).findByIdAndPersonId_Id(500, 1L);
    }

    @Test
    @DisplayName("getById() - Debe lanzar ResourceNotFoundException cuando no existe")
    void getById_ShouldThrowException_WhenNotFound() {
        // Arrange
        when(personCertificationRepository.findByIdAndPersonId_Id(999, 1L))
                .thenReturn(Optional.empty());

        // Act & Assert
        ResourceNotFoundException exception = assertThrows(
                ResourceNotFoundException.class,
                () -> service.getById(1L, 999));

        assertTrue(exception.getMessage().contains("Certification not found"));
    }

    @Test
    @DisplayName("delete() - Debe realizar soft delete correctamente")
    void delete_ShouldPerformSoftDelete_WhenCertificationExists() {
        // Arrange
        when(personCertificationRepository.findByIdAndPersonId_Id(500, 1L))
                .thenReturn(Optional.of(mockPersonCertification));
        when(personCertificationRepository.save(any(PersonCertificationEntity.class)))
                .thenReturn(mockPersonCertification);

        // Act
        Integer result = service.delete(1L, 500);

        // Assert
        assertNotNull(result);
        assertEquals(500, result);
        verify(personCertificationRepository).save(argThat(entity -> entity.getDeletedAt() != null));
    }

    @Test
    @DisplayName("delete() - Debe lanzar ResourceNotFoundException cuando no existe")
    void delete_ShouldThrowException_WhenNotFound() {
        // Arrange
        when(personCertificationRepository.findByIdAndPersonId_Id(999, 1L))
                .thenReturn(Optional.empty());

        // Act & Assert
        ResourceNotFoundException exception = assertThrows(
                ResourceNotFoundException.class,
                () -> service.delete(1L, 999));

        assertTrue(exception.getMessage().contains("Certification not found"));
        verify(personCertificationRepository, never()).save(any());
    }
}
```

## Conclusión

La implementación del módulo de Certificaciones debe seguir estrictamente esta especificación para asegurar la consistencia con la arquitectura de `people-center`. Cualquier desviación de este diseño debe ser discutida y aprobada por el equipo de arquitectura.
