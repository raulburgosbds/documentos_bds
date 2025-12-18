# Guía de Estándares de Desarrollo - People Center

**Versión**: 2.0  
**Fecha**: 2024-12-15  
**Propósito**: Establecer estándares comunes para el desarrollo coherente entre todos los miembros del equipo

---

##  Tabla de Contenidos

### PARTE 1: FUNDAMENTOS
1. [Principios y Arquitectura](#1-principios-y-arquitectura)
   - Principios Básicos.
   - Flujo de una Request y Separación de Responsabilidades (SRP)

### PARTE 2: LIBRARY (Contratos)
2. [Library - DTOs y Contratos](#2-library---dtos-y-contratos)
   - DTOs
   - Requests
   - Enums
   - Deserializers
   - PathV2
   - Person.java
   - CHANGELOG.md

### PARTE 3: MICROSERVICE (Implementación)
3. [Implementación en Microservice](#3-implementación-en-microservice)
   - Entities y Repositories
   - Mappers
   - Validators
   - Services
   - Controllers (REST API)

### PARTE 4: PATRONES AVANZADOS
4. [Patrones de Implementación](#4-patrones-de-implementación)
   - Strategy Pattern (Collision)
   - Soft-Delete con @Where
   - @Transactional
   - Manejo de Excepciones

### PARTE 5: CALIDAD
5. [Testing y Documentación](#5-testing-y-documentación)
   - Tests de Service
   - Tests de Controller
   - JavaDoc
   - Swagger/OpenAPI

### PARTE 6: ENTREGA
6. [Git y Versionado](#6-git-y-versionado)
   - Conventional Commits
   - Branching Strategy
   - Pull Requests

### PARTE 7: REFERENCIA
7. [Checklist de Revisión](#7-checklist-de-revisión)

---

## 1. Principios y Arquitectura

### 1.1. Principios Básicos.

#### Fail-Fast

**Validar temprano, fallar rápido:**

```java
//  CORRECTO
public Long create(Long personId, CreateRequest request) {
    // 1. Validar PRIMERO
    validator.validateRequest(request);
    
    // 2. Buscar dependencias
    PersonEntity person = repository.findById(personId)
        .orElseThrow(() -> new NotFoundException(...));
    
    // 3. Lógica de negocio
}

//  INCORRECTO - Validar después de tocar la DB
public Long create(Long personId, CreateRequest request) {
    PersonEntity person = repository.findById(personId).orElse(null);
    validator.validateRequest(request); // Muy tarde
}
```

#### Usar Excepciones, No retornar Nulls

**Nunca retornar `null` en métodos públicos:**

```java
//  CORRECTO
public PersonEntity findById(Long id) {
    return repository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Person not found with id: " + id));
}

//  INCORRECTO
public PersonEntity findById(Long id) {
    return repository.findById(id).orElse(null); // NO hacer esto
}
```

---

### 1.2. Arquitectura de Capas

#### Flujo de una Request y Separación de Responsabilidades (SRP)

```
1. HTTP Request
   ↓
2. Controller (@RestController)
   - Validación básica (@Valid)
   - Orquestación
   - NO icluir lógica de negocio
   ↓
3. Validator (si aplica)
   - Reglas de negocio complejas
   ↓
4. Service (@Service)
   - Lógica de negocio
   - Coordinación
   ↓
5. Strategy/Detector (si aplica)
   - Detección de colisiones
   - Estrategia de persistencia
   ↓
6. Repository (@Repository)
   - Acceso a datos
   - Queries
   ↓
7. Mapper
    - Transformación DTO ↔ Entity
   ↓
8. Database
   ↓
9. HTTP Response
```

#### Responsabilidades por Capa

| Capa | Responsabilidad | NO Debe Hacer |
|------|-----------------|---------------|
| **Controller** | - Orquestar request/response<br>- Validación básica (@Valid)<br>- Mapeo HTTP status | - Lógica de negocio<br>- Acceso directo a Repository<br>- Manejo de transacciones |
| **Validator** | - Validaciones complejas<br>- Reglas de negocio<br>- Logging de validaciones | - Persistir datos<br>- Lógica de negocio<br>- Acceso a Repository |
| **Service** | - Lógica de negocio<br>- Coordinación de capas<br>- Transacciones (@Transactional) | - Validación de requests<br>- Mapeo HTTP<br>- Retornar null |
| **Strategy** | - Estrategias de persistencia<br>- Aplicar reglas según modo | - Acceso directo a DB<br>- Validaciones de negocio |
| **Detector** | - Detectar colisiones<br>- Comparar entidades | - Persistir datos<br>- Acceso a DB |
| **Repository** | - Queries a DB<br>- Acceso a datos | - Lógica de negocio<br>- Validaciones |

---

## 2. Library - DTOs y Contratos

### Visión General

La **library** es un módulo compartido que contiene DTOs, Enums, Interfaces, Rutas API, Clientes API y Requests.

**Ubicación**: `C:\repos\bds\people-center\library\`

### Estructura de Directorios

```
library/
├── CHANGELOG.md                           // Documentación de cambios (OBLIGATORIO antes de PR)
├── model/
│   └── src/main/java/ar/com/bds/lib/peoplecenter/
│       ├── api/
│       │   └── PathV2.java                // Rutas de API
│       ├── model/
│       │   ├── Person.java                // DTO principal
│       │   ├── PersonCertification.java   // DTOs de entidades
│       │   ├── deserializer/              // Deserializers personalizados
│       │   │   └── StrictZonedDateTimeDeserializer.java
│       │   ├── enums/                     // Enums del proyecto
│       │   ├── interfaces/                // Interfaces compartidas
│       │   └── requests/                  // Requests (Create, Update, Patch)
```

---

### 2.1. DTOs (Data Transfer Objects)

**Ubicación**: `library/model/.../model/`

#### DTOs Públicos (APIs de Negocio)

**Reglas para DTOs Públicos**:
- 4 anotaciones: `@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`
- `Long` para IDs (no `Integer`)
- `ZonedDateTime` para fechas (no `LocalDateTime`)
- Bean Validation con mensajes descriptivos
- NO incluir `deletedAt` (los consumidores solo ven registros activos)

**Estructura Básica**:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PersonCertification {
    private Long id;  // Long, no Integer
    
    @NotNull(message = "Certification code is required")
    private String certificationCode;
    
    @NotNull(message = "Start date is required")
    private ZonedDateTime startDate;  // ZonedDateTime, no LocalDateTime
    
    private ZonedDateTime endDate;
    private ZonedDateTime createdAt;
    
    // NO incluir deletedAt - solo para registros activos
}
```

#### DTOs de Administración (Opcionales)
()
**Cuándo usar**: Endpoints de auditoría, historial o administración que necesitan mostrar el estado de los registros incluyendo por ejemplo el valor de Deleted At.

**Reglas para DTOs de Auditoría**:
- SÍ incluir `deletedAt` (los consumidores ven registros activos, inactivos y borrados)
- Incluir método `isActive()` para verificación si corresponde
- Incluir método `getStatus()` para visualización del estado literal
- Usar sufijo `Audit`, `Admin` o `History` en el nombre

**Estructura**:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PersonCertificationAudit {
    private Long id;
    private String certificationCode;
    private ZonedDateTime startDate;
    private ZonedDateTime endDate;
    private ZonedDateTime createdAt;
    private ZonedDateTime deletedAt;  // SÍ incluir para auditoría
    
    /**
     * Verifica si el registro está activo.
     * @return true si no está eliminado
     */
    public boolean isActive() {
        return deletedAt == null;
    }
    
    /**
     * Obtiene el estado del registro.
     * @return "ACTIVE" o "DELETED"
     */
    public String getStatus() {
        return deletedAt == null ? "ACTIVE" : "DELETED";
    }
}
```


#### Tabla de Decisión: ¿Incluir registros con deletedAt != Null?

| Tipo de DTO | Incluir deletedAt | Razón | Ejemplo de Uso |
|-------------|-------------------|-------|----------------|
| **DTO Público** | NO | Los consumidores solo necesitan registros activos | `GET /v2/people/{id}/certifications` |
| **DTO de Auditoría** | SÍ | Administradores necesitan ver estado completo | `GET /v2/people/{id}/certifications/history` |
| **DTO de Admin** | SÍ | Dashboards administrativos requieren estado | `GET /admin/certifications` |

#### Ejemplo Completo de los DTOs en un Service

```java
@Service
public class PersonCertificationServiceImpl {
    
    // Endpoint público - retorna DTO sin deletedAt
    @Transactional(readOnly = true)
    public List<PersonCertification> getValidCertifications(Long personId) {
        return repository.findValidCertifications(personId, ZonedDateTime.now())
            .stream()
            .map(mapper::toDto)  // PersonCertification (sin deletedAt)
            .collect(Collectors.toList());
    }
    
    // Endpoint de auditoría - retorna DTO con deletedAt
    @Transactional(readOnly = true)
    public List<PersonCertificationAudit> getHistory(Long personId) {
        return repository.findAllByPersonIdIncludingDeleted(personId)
            .stream()
            .map(mapper::toAuditDto)  // PersonCertificationAudit (con deletedAt)
            .collect(Collectors.toList());
    }
}
```

#### Crear Mapper para Ambos Tipos

```java
@Mapper(componentModel = "spring")
public interface PersonCertificationMapper {
    
    // Para DTOs públicos (sin deletedAt)
    PersonCertification toDto(PersonCertificationEntity entity);
    
    // Para DTOs de auditoría (con deletedAt)
    PersonCertificationAudit toAuditDto(PersonCertificationEntity entity);
}
```

---

### 2.2. Requests

**Naming Convention**:
- `Create{Entity}Request` - Para crear
- `Update{Entity}Request` - Para actualizar completo
- `Patch{Entity}Request` - Para actualizar parcial

**Ubicación**: `library/model/.../requests/`

**Estructura**:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class CreatePersonCertificationRequest {
    
    @NotBlank(message = "Certification code is required")
    @Size(max = 50, message = "Code must not exceed 50 characters")
    private String certificationCode;
    
    @NotNull(message = "Start date is required")
    private ZonedDateTime startDate;
    
    private ZonedDateTime endDate;
    
}
```

---

### 2.3. Enums

**Reglas**:
-  Usar `@JsonValue` para serialización
-  Usar `@JsonCreator` para deserialización
-  Implementar metodo `fromValue()` con validación
-  Nombres en UPPER_CASE

**Ubicación**: `library/model/.../enums/`

**Estructura**:

```java
package ar.com.bds.lib.peoplecenter.model.enums;

import com.fasterxml.jackson.annotation.JsonCreator;
import com.fasterxml.jackson.annotation.JsonValue;
import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * Persistence mode for collision handling.
 * 
 * - DEFAULT: Strict mode - rejects if conflicts detected
 * - FORCE: Override mode - soft-deletes conflicts and proceeds
 */
@Getter
@AllArgsConstructor
public enum PersistenceMode {
    
    /**
     * DEFAULT mode: Strict validation.
     * Rejects persistence if any collision is detected.
     */
    DEFAULT("DEFAULT"),
    
    /**
     * FORCE mode: Override conflicts.
     * Soft-deletes conflicting records and proceeds with save.
     */
    FORCE("FORCE");
    
    /**
     * String value for JSON serialization.
     */
    @JsonValue
    private final String value;
    
    /**
     * Converts a string value to the corresponding enum.
     * Used by Jackson for JSON deserialization.
     * 
     * @param value String value from JSON (e.g., "DEFAULT", "FORCE")
     * @return Corresponding PersistenceMode enum
     * @throws IllegalArgumentException if value is invalid
     */
    @JsonCreator
    public static PersistenceMode fromValue(String value) {
        // Null/empty validation
        if (value == null || value.trim().isEmpty()) {
            throw new IllegalArgumentException(
                "PersistenceMode value cannot be null or empty. " +
                "Valid values are: DEFAULT, FORCE"
            );
        }
        
        // Find matching enum (case-insensitive)
        for (PersistenceMode mode : PersistenceMode.values()) {
            if (mode.value.equalsIgnoreCase(value)) {
                return mode;
            }
        }
        
        // No match found - throw descriptive error
        throw new IllegalArgumentException(
            String.format(
                "Invalid PersistenceMode: '%s'. Valid values are: DEFAULT, FORCE",
                value
            )
        );
    }
}
```

---

### 2.4. Deserializers - Validación de Fechas

**Ubicación**: `library/model/.../deserializer/`

**Problema que Resuelve**:
-  Evita timestamps numéricos (`1234567890`)
-  Evita fechas sin hora (`"2024-01-15"`)
-  Evita strings arbitrarios (`"invalid"`)
-  Solo acepta ISO-8601 completo (`"2024-01-15T10:00:00Z"`)

**Reglas**:
-  Usar en **todos** los campos `ZonedDateTime` de Requests
-  Aplicar con `@JsonDeserialize(using = StrictZonedDateTimeDeserializer.class)`
-  NO usar en DTOs (solo en Requests)

**Implementación**:

```java
public class StrictZonedDateTimeDeserializer extends JsonDeserializer<ZonedDateTime> {
    
    @Override
    public ZonedDateTime deserialize(JsonParser parser, DeserializationContext context) 
            throws IOException {
        if (parser.getCurrentToken() != JsonToken.VALUE_STRING) {
            throw new JsonMappingException(parser,
                    "Date-time value must be a string in ISO-8601 format");
        }
        
        try {
            return ZonedDateTime.parse(parser.getText());
        } catch (DateTimeParseException e) {
            throw new JsonMappingException(parser,
                    "Invalid date-time format. Expected ISO-8601", e);
        }
    }
}
```

**Uso en Requests**:

```java
@Data
@Builder
public class CreatePersonCertificationRequest {
    
    @NotNull(message = "Start date is required")
    @JsonDeserialize(using = StrictZonedDateTimeDeserializer.class)  // ← CLAVE
    private ZonedDateTime startDate;
    
    @JsonDeserialize(using = StrictZonedDateTimeDeserializer.class)
    private ZonedDateTime endDate;
}
```

---

### 2.5 Person.java - DTO Principal

**¿Qué es y Por Qué Existe?**
Person es el DTO raíz que representa una persona en el sistema. Actúa como un agregado que contiene todos los datos relacionados con una persona: información básica, certificaciones, contactos, documentos, etc. Este patrón permite obtener toda la información de una persona en una sola respuesta de API.

**Beneficios principales:**

- Agregado completo: Una sola llamada a la API retorna toda la información de la persona
- Validación en cascada: @Valid asegura que todos los objetos anidados sean válidos
- Inicialización segura: @Builder.Default evita NullPointerException en listas
- Tipo seguro: Implementa HasId para operaciones genéricas
- Inmutabilidad opcional: Puede usarse con @Builder para crear objetos inmutables

**Reglas:**

- Usar @Valid para validación en cascada
- Usar @Builder.Default con new ArrayList<>()
- Mantener orden alfabético
- Inicializar siempre (evita NPE)

**Ubicación**: `library/model/.../model/Person.java`

**Estructura**:

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Person implements HasId {
    
    private Long id;
    private String denomination;
    private PersonType type;
    
    @Valid
    @Builder.Default
    private List<PersonCertification> certifications = new ArrayList<>();
    
    @Valid
    @Builder.Default
    private List<Contact> contacts = new ArrayList<>();
    
    // ... otras listas en orden alfabético
}
```
---

### 2.6. PathV2 - Rutas de API

**¿Qué es y Por Qué Existe?**
PathV2 es una clase utilitaria que centraliza todas las rutas (endpoints) de la API REST. Actúa como un contrato compartido entre la library y el microservice, garantizando que ambos usen exactamente las mismas URLs.

**Beneficios principales:**

- Single Source of Truth: Una sola definición de cada ruta, compartida entre servidor y cliente
- Refactoring seguro: Cambiar una ruta en un solo lugar actualiza automáticamente controller y client
- Prevención de typos: Errores detectados en compile-time, no en runtime (404)
- Documentación viva: Un vistazo rápido muestra toda la API disponible
- Versionado fácil: Cambiar de v2 a v3 modificando una sola línea

**Reglas**:
-  Mantener orden alfabético
-  Usar kebab-case en URLs (`/tax-activities`, no `/taxActivities`)
-  Naming: `{ENTITY}_PATH`
-  Usar `BASE_PATH` para construir

**Ubicación**: `library/model/.../api/PathV2.java`

**Estructura**:

```java
@UtilityClass
public final class PathV2 {
    
    private static final String BASE_PATH = "/v2/people";
    
    public static final String CERTIFICATIONS_PATH = BASE_PATH + "/{id}/certifications";
    public static final String CHANNELS_PATH = BASE_PATH + "/{id}/channels";
    public static final String CONTACTS_PATH = BASE_PATH + "/{id}/contacts";
    // ... en orden alfabético
}
```

**Reglas**:
-  Usar `@Valid` para validación en cascada
-  Usar `@Builder.Default` con `new ArrayList<>()`
-  Mantener orden alfabético
-  Inicializar siempre (evita NPE)

---

### 2.7. CHANGELOG.md - OBLIGATORIO Antes de PR

¿Qué es y Por Qué Existe?
CHANGELOG.md
 es un archivo obligatorio en la library que documenta todos los cambios realizados en cada versión. Actúa como un historial de cambios que permite a los consumidores de la library entender qué se agregó, modificó o eliminó en cada release.

Beneficios principales:

Transparencia: Los consumidores saben exactamente qué cambió en cada versión
Trazabilidad: Fácil identificar cuándo se introdujo una feature o fix

**Ubicación**: `library/CHANGELOG.md`

**Estructura**:

```markdown
## ChangeLog

*Version 0.11.0-SNAPSHOT-CON27-2 - Released 2025-12-16*
- **Added**
  - TaxActivity Model
  - CreateTaxActivityRequest Model
  - TaxActivityType Enum
  - TAX_ACTIVITIES_PATH in PathV2
  - taxActivities list in Person Model

*Version 0.10.5-SNAPSHOT-CON27-1 - Released 2025-12-05*
- **Added**
  - Certification Model
  - PersonCertification Model
```
    
---

#### Entity

**Reglas**:
-  Implementar interfaces `HasId`, `HasDeletedAt`
-  `@CreationTimestamp` para `createdAt`
-  `@JsonBackReference` en relaciones bidireccionales

**Ubicación**: `microservice/src/main/java/ar/com/bds/people/center/entity/`

**Estructura**:

```java
@Entity
@Table(name = "person_certification")
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Where(clause = "deleted_at IS NULL")  // Soft-delete
public class PersonCertificationEntity implements HasId, HasDeletedAt {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "person_id", nullable = false)
    @JsonBackReference
    private PersonEntity personId;
    
    @Column(name = "certification_code", nullable = false, length = 50)
    private String certificationCode;
    
    @Column(name = "start_date", nullable = false)
    private ZonedDateTime startDate;
    
    @Column(name = "end_date")
    private ZonedDateTime endDate;
    
    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private ZonedDateTime createdAt;
    
    @Override
    public boolean isDeleted() {
        return deletedAt != null;
    }
}
```

#### Repository

**Reglas**:
-  Si es posible usar metodo JPA con preferencia a usar Query nativa. 
-  Query nativa documentada 
-  Usar `@Param` en queries nativas

**Ubicación**: `microservice/src/main/java/ar/com/bds/people/center/repository/`

**Estructura**:

```java
@Repository
public interface PersonCertificationRepository extends JpaRepository<PersonCertificationEntity, Long> {
    
    // Método JPA - solo activos (respeta @Where)
    Optional<PersonCertificationEntity> findByIdAndPersonId_Id(Long id, Long personId);
    
    /**
     * Query nativa para incluir eliminados (bypasea @Where).
     * Usado para: DELETE idempotente, auditoría, recuperación.
     */
    @Query(value = "SELECT * FROM person_certification WHERE id = :id AND person_id = :personId", 
           nativeQuery = true)
    Optional<PersonCertificationEntity> findByIdAndPersonIdIncludingDeleted(
        @Param("id") Long id,
        @Param("personId") Long personId);
    
    /**
     * Find valid certifications (not deleted, started, not expired).
     */
    @Query("SELECT pc FROM PersonCertificationEntity pc " +
           "WHERE pc.personId.id = :personId " +
           "AND pc.deletedAt IS NULL " +
           "AND (pc.startDate IS NULL OR pc.startDate <= :now) " +
           "AND (pc.endDate IS NULL OR pc.endDate >= :now)")
    List<PersonCertificationEntity> findValidCertifications(
        @Param("personId") Long personId,
        @Param("now") ZonedDateTime now);
}
```

---

### 3.2. Mappers

**Reglas**:
-  `@Mapper(componentModel = "spring")`
-  Campos autogenerados con `ignore = true`
-  Conversiones de tipo documentadas

**Ubicación**: `microservice/src/main/java/ar/com/bds/people/center/mapper/`

**Estructura**:

```java
@Mapper(componentModel = "spring")
public interface PersonCertificationMapper {
    
    @Mapping(target = "id", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "deletedAt", ignore = true)
    PersonCertificationEntity toEntity(CreatePersonCertificationRequest request);
    
    PersonCertification toDto(PersonCertificationEntity entity);
    
    List<PersonCertification> toDtoList(List<PersonCertificationEntity> entities);
}
```

---

### 3.3. Validators

**Reglas**:
-  Validaciones de reglas de negocio complejas
-  Mensajes de error claros
-  Logging con nivel `warn`

**Ubicación**: `microservice/src/main/java/ar/com/bds/people/center/validator/`

**Estructura**:

```java
@Component
@Slf4j
public class CertificationValidator {
    
    /**
     * Validates mutually exclusive fields (percentage vs aliquot).
     */
    public void validateMutuallyExclusiveFields(CreatePersonCertificationRequest request) {
        boolean hasPercentage = request.getPercentage() != null;
        boolean hasAliquot = request.getAliquot() != null;
        
        if (hasPercentage && hasAliquot) {
            log.warn("Validation failed: both percentage and aliquot provided");
            throw new IllegalArgumentException(
                "Percentage and aliquot are mutually exclusive. Provide only one.");
        }
    }
    
    /**
     * Validates date range (startDate < endDate).
     */
    public void validateDateRange(ZonedDateTime startDate, ZonedDateTime endDate) {
        if (endDate != null && startDate.isAfter(endDate)) {
            log.warn("Validation failed: startDate {} is after endDate {}", startDate, endDate);
            throw new IllegalArgumentException(
                "Start date must be before end date");
        }
    }
}
```

---

### 3.4. Services

**Reglas**:
-  `@Transactional` en métodos que escriben
-  `@Transactional(readOnly = true)` en consultas
-  Validaciones antes de persistir
-  Logging en inicio/fin de operaciones
-  Excepciones con mensajes descriptivos
-  NO retornar `null` en métodos públicos
-  Crear constructor explícito cuando hay `@Qualifier` (No usar constructor de Loombok en este caso)
-  No permitir el borrado logico de un mismo registro mas de una vez, lanzar IllegalStateException

**Ubicación**: `microservice/src/main/java/ar/com/bds/people/center/service/`

**Interface**:

```java
public interface PersonCertificationService {
    
    /**
     * Create a new certification for a person.
     * 
     * <p><b>Business Rules:</b></p>
     * <ul>
     * <li>Validates mutually exclusive fields (percentage vs aliquot)</li>
     * <li>Validates date range (startDate < endDate)</li>
     * <li>Detects and handles collisions based on PersistenceMode</li>
     * </ul>
     * 
     * @param personId Person ID
     * @param request Certification data
     * @param mode Collision handling strategy
     * @return ID of the created certification
     */
    Long create(Long personId, CreatePersonCertificationRequest request, PersistenceMode mode);
    
    PersonCertification getById(Long personId, Long certificationId);
    
    List<PersonCertification> getValidCertifications(Long personId);
    
    Long delete(Long personId, Long certificationId);
}
```

**Implementation**:

```java
@Service
@Slf4j
public class PersonCertificationServiceImpl implements PersonCertificationService {
    
    private final PersonCertificationRepository repository;
    private final PersonCertificationMapper mapper;
    private final CertificationValidator validator;
    private final PersistenceStrategy<PersonCertificationEntity> defaultStrategy;
    private final PersistenceStrategy<PersonCertificationEntity> forceStrategy;
    private final CollisionDetector<PersonCertificationEntity> detector;
    
    // Constructor explícito (NO @RequiredArgsConstructor cuando hay @Qualifier)
    public PersonCertificationServiceImpl(
            PersonCertificationRepository repository,
            PersonCertificationMapper mapper,
            CertificationValidator validator,
            @Qualifier("defaultPersistenceStrategy") 
            PersistenceStrategy<PersonCertificationEntity> defaultStrategy,
            @Qualifier("forcePersistenceStrategy") 
            PersistenceStrategy<PersonCertificationEntity> forceStrategy,
            CollisionDetector<PersonCertificationEntity> detector) {
        this.repository = repository;
        this.mapper = mapper;
        this.validator = validator;
        this.defaultStrategy = defaultStrategy;
        this.forceStrategy = forceStrategy;
        this.detector = detector;
    }
    
    @Override
    @Transactional
    public Long create(Long personId, CreatePersonCertificationRequest request, PersistenceMode mode) {
        log.info("Creating certification for person: {}, mode: {}", personId, mode);
        
        // 1. Validaciones
        validator.validateMutuallyExclusiveFields(request);
        validator.validateDateRange(request.getStartDate(), request.getEndDate());
        
        // 2. Buscar persona
        PersonEntity person = personRepository.findById(personId)
            .orElseThrow(() -> new ResourceNotFoundException("Person not found: " + personId));
        
        // 3. Mapear a entity
        PersonCertificationEntity newEntity = mapper.toEntity(request);
        newEntity.setPersonId(person);
        
        // 4. Buscar existentes
        List<PersonCertificationEntity> existing = repository.findByPersonId_Id(personId);
        
        // 5. Aplicar estrategia
        PersistenceStrategy<PersonCertificationEntity> strategy = 
            mode == PersistenceMode.FORCE ? forceStrategy : defaultStrategy;
        
        Long id = strategy.apply(newEntity, existing, repository::save, detector);
        
        log.info("Certification created with id: {}", id);
        return id;
    }
    
    @Override
    @Transactional(readOnly = true)
    public PersonCertification getById(Long personId, Long certificationId) {
        log.info("Getting certification {} for person: {}", certificationId, personId);
        
        return repository.findByIdAndPersonIdIncludingDeleted(certificationId, personId)
            .map(mapper::toDto)
            .orElseThrow(() -> new ResourceNotFoundException(
                String.format("Certification %d not found for person %d", 
                    certificationId, personId)));
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<PersonCertification> getValidCertifications(Long personId) {
        log.info("Getting valid certifications for person: {}", personId);
        
        return repository.findValidCertifications(personId, ZonedDateTime.now())
            .stream()
            .map(mapper::toDto)
            .collect(Collectors.toList());
    }
    
    @Override
    @Transactional
    public Long delete(Long personId, Long certificationId) {
            log.info("Deleting certification {} for person: {}", certificationId, personId);

            PersonCertificationEntity entity = personCertificationRepository
                            .findByIdAndPersonId_Id(certificationId, personId)
                            .orElseThrow(() -> new ResourceNotFoundException(
                                            String.format("Certification not found with id %d for person %d",
                                            certificationId,                                   personId)));

            // Verificar si ya está eliminada
            if (entity.getDeletedAt() != null) {
                    log.warn("Certification with id {} for person {} is already deleted", certificationId,
                    personId);
                    throw new IllegalStateException(
                                 String.format("Certification with id %d for person %d is already deleted", certificationId, personId));
            }

            entity.setDeletedAt(ZonedDateTime.now());
            personCertificationRepository.save(entity);

            log.info("Certification {} deleted successfully", certificationId);
            return entity.getId();
        }
}
```

---

### 3.5. Controllers (REST API)

**Reglas - MediaType**:

| Método HTTP | consumes | produces | Ejemplo |
|-------------|----------|----------|---------|
| **POST** |  Sí |  Sí | Crear entidad |
| **PUT** |  Sí |  Sí | Actualizar completo |
| **PATCH** |  Sí |  Sí | Actualizar parcial |
| **GET** |  No |  Sí | Obtener entidad |
| **DELETE** |  No |  Opcional | Eliminar |

**Reglas Generales**:
-  Solo orquestación/enrutamiento, sin lógica de negocio
-  `@Valid` en request bodies
-  Documentación Swagger completa
-  HTTP status codes correctos (201 para POST, 200 para GET/DELETE, etc.)
-  Usar `MediaType.APPLICATION_JSON_VALUE` en metodos que escriben y en los que leen.
-  NO hace checks de null/Evitar vaidaciones en el controller

**Ubicación**: `microservice/src/main/java/ar/com/bds/people/center/controller/`

**Estructura**:

```java
@RestController
@RequestMapping("/v2/people/{personId}/certifications")
@Tag(name = "People Center - Certifications")
@RequiredArgsConstructor
public class CertificationsController {
    
    private final PersonCertificationService service;
    
    @PostMapping(
        consumes = MediaType.APPLICATION_JSON_VALUE,
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    @Operation(summary = "Create a new certification")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "201", description = "Created"),
        @ApiResponse(responseCode = "400", description = "Invalid request"),
        @ApiResponse(responseCode = "404", description = "Person not found"),
        @ApiResponse(responseCode = "409", description = "Conflict detected")
    })
    public ResponseEntity<Long> create(
        @PathVariable Long personId,
        @RequestParam(name = "mode", defaultValue = "DEFAULT") PersistenceMode mode,
        @Valid @RequestBody CreatePersonCertificationRequest request) {
        
        Long id = service.create(personId, request, mode);
        return ResponseEntity.status(HttpStatus.CREATED).body(id);
    }
    
    @GetMapping(produces = MediaType.APPLICATION_JSON_VALUE)
    @Operation(summary = "Get all valid certifications")
    @ApiResponse(responseCode = "200", description = "Success")
    public ResponseEntity<List<PersonCertification>> getAll(@PathVariable Long personId) {
        List<PersonCertification> certifications = service.getValidCertifications(personId);
        return ResponseEntity.ok(certifications);
    }
    
    @GetMapping(
        value = "/{id}",
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    @Operation(summary = "Get certification by ID")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Success"),
        @ApiResponse(responseCode = "404", description = "Not found")
    })
    public ResponseEntity<PersonCertification> getById(
        @PathVariable Long personId,
        @PathVariable Long id) {
        
        PersonCertification certification = service.getById(personId, id);
        return ResponseEntity.ok(certification);
    }
    
    @DeleteMapping(
        value = "/{id}",
        produces = MediaType.APPLICATION_JSON_VALUE
    )
    @Operation(summary = "Delete certification (soft delete)")
    @ApiResponses(value = {
        @ApiResponse(responseCode = "200", description = "Deleted"),
        @ApiResponse(responseCode = "404", description = "Not found")
    })
    public ResponseEntity<Long> delete(
        @PathVariable Long personId,
        @PathVariable Long id) {
        
        Long deletedId = service.delete(personId, id);
        return ResponseEntity.ok(deletedId);
    }
}
```
---

## 4. Patrones de Implementación

### 4.1. Strategy Pattern (Collision)

#### Visión General

El **Strategy Pattern** permite manejar colisiones de datos con dos estrategias:
- **DEFAULT**: Rechaza si hay conflictos (lanza `ConflictException`)
- **FORCE**: Soft-delete de conflictos y procede

> **Aencion** Referirse al documento especifico de [implementacion de la estrategia](https://github.com/raulburgosbds/documentos_bds/blob/main/guides/guia-implementacion-estrategia-force-default/guia-implementacion-estrategia-force-default.md) para una guia detallada de su implementacion.

### 4.2. @Transactional

**¿Qué es y Por Qué Existe?**
@Transactional es una anotación de Spring que gestiona automáticamente las transacciones de base de datos. Garantiza que un conjunto de operaciones se ejecute de forma atómica (todo o nada), manteniendo la consistencia de los datos incluso si ocurre un error.

**Beneficios principales:**

- Atomicidad: Todas las operaciones se completan o ninguna (rollback automático en caso de error)
- Consistencia: Los datos quedan en estado válido incluso si falla una operación
- Aislamiento: Las transacciones concurrentes no interfieren entre sí
- Simplicidad: Spring maneja commit/rollback automáticamente, sin código boilerplate
- Propagación: Control sobre cómo se comportan transacciones anidadas

#### Tabla de Uso

| Operación | Anotación | Razón |
|-----------|-----------|-------|
| **CREATE** | `@Transactional` | Escribe en DB |
| **UPDATE** | `@Transactional` | Escribe en DB |
| **DELETE** | `@Transactional` | Escribe en DB (soft-delete) |
| **GET** | `@Transactional(readOnly = true)` | Solo lectura, optimización |
| **LIST** | `@Transactional(readOnly = true)` | Solo lectura, optimización |


#### Ejemplos:

```java
@Service
public class MyServiceImpl {
    
    //  CORRECTO - CREATE, UPDATE, DELETE
    @Transactional
    public Long create(...) { }
    
    @Transactional
    public void update(...) { }
    
    @Transactional
    public Long delete(...) { }
    
    //  CORRECTO - GET, LIST (readOnly = true)
    @Transactional(readOnly = true)
    public MyEntity getById(...) { }
    
    @Transactional(readOnly = true)
    public List<MyEntity> getAll(...) { }
}
```

#### Errores Comunes

```java
//  Error 1: @Transactional en Controller
@RestController
public class MyController {
    @Transactional  // NO! Va en Service
    public ResponseEntity<Long> create(...) { }
}

//  CORRECTO - @Transactional en Service
@Service
public class MyServiceImpl {
    @Transactional
    public Long create(...) { }
}

//  Error 2: @Transactional en método privado
@Service
public class MyServiceImpl {
    @Transactional  // NO! No funciona en privados
    private void helper() { }
}

//  CORRECTO - @Transactional en método público
@Service
public class MyServiceImpl {
    @Transactional
    public void publicMethod() { }
}

//  Error 3: Llamada interna (self-invocation)
@Service
public class MyServiceImpl {
    public void method1() {
        this.method2();  // NO! Bypasea @Transactional
    }
    
    @Transactional
    public void method2() { }
}

//  CORRECTO - Inyectar el mismo servicio o refactorizar
@Service
public class MyServiceImpl {
    private final MyServiceImpl self;
    
    public void method1() {
        self.method2();  // OK! Respeta @Transactional
    }
    
    @Transactional
    public void method2() { }
}
```

---

### 4.4. Manejo de Excepciones

#### Reglas

-  Loggear ANTES de lanzar excepción
-  Mensajes descriptivos y específicos
-  Incluir IDs relevantes en el mensaje
-  Usar excepción apropiada según contexto
-  NO usar excepciones genéricas (`Exception`, `RuntimeException`)

#### Excepciones del Proyecto

| Excepción | HTTP Status | Cuándo Usar | Dónde Lanzar |
|-----------|-------------|-------------|--------------|
| **ResourceNotFoundException** | 404 | Entity no encontrada | Service |
| **ConflictException** | 409 | Colisión en DEFAULT mode | Strategy |
| **IllegalArgumentException** | 400 | Validaciones de negocio | Validator, Service |
| **IllegalStateException** | 400 | Eliminar reigistro ya eliminado | Service |
| **TypeNotFoundException** | 404 | Tipo de catálogo no encontrado | Service |
| **DataBaseViolationException** | 500 | Violación de constraint DB | Repository |

#### Ejemplos

**ResourceNotFoundException**:

```java
@Service
public class PersonCertificationServiceImpl {
    
    @Transactional(readOnly = true)
    public PersonCertification getById(Long personId, Long certificationId) {
        log.info("Getting certification {} for person: {}", certificationId, personId);
        
        return repository.findByIdAndPersonIdIncludingDeleted(certificationId, personId)
            .map(mapper::toDto)
            .orElseThrow(() -> {
                log.error("Certification {} not found for person {}", certificationId, personId);
                return new ResourceNotFoundException(
                    String.format("Certification %d not found for person %d", 
                        certificationId, personId));
            });
    }
}
```

**ConflictException**:

```java
@Component
public class DefaultPersistenceStrategy<T> implements PersistenceStrategy<T> {
    
    @Override
    public Long apply(T newEntity, Collection<T> existing, 
                     Consumer<T> saver, CollisionDetector<T> detector) {
        Collection<T> conflicts = detector.detectCollisions(newEntity, existing);
        
        if (!conflicts.isEmpty()) {
            List<Long> conflictingIds = conflicts.stream()
                .map(HasId::getId)
                .collect(Collectors.toList());
            
            log.warn("Collision detected. Conflicting IDs: {}", conflictingIds);
            
            throw new ConflictException(
                "Collision detected. Use FORCE mode to override.",
                conflictingIds
            );
        }
        
        saver.accept(newEntity);
        return newEntity.getId();
    }
}
```

**IllegalArgumentException**:

```java
@Component
public class CertificationValidator {
    
    public void validateMutuallyExclusiveFields(CreatePersonCertificationRequest request) {
        boolean hasPercentage = request.getPercentage() != null;
        boolean hasAliquot = request.getAliquot() != null;
        
        if (hasPercentage && hasAliquot) {
            log.warn("Validation failed: both percentage and aliquot provided");
            throw new IllegalArgumentException(
                "Percentage and aliquot are mutually exclusive. Provide only one.");
        }
    }
}
```

---

## 5. Testing y Documentación

### 5.1. Tests de Service

**Reglas**:
-  NO usar `@InjectMocks` cuando hay `@Qualifier`
-  Instanciar service manualmente en `@BeforeEach`
-  Usar `@DisplayName` descriptivos
-  Patrón Arrange-Act-Assert

**Estructura**:

```java
@ExtendWith(MockitoExtension.class)
@DisplayName("PersonCertificationService - Unit Tests")
class PersonCertificationServiceTest {
    
    @Mock
    private PersonCertificationRepository repository;
    @Mock
    private PersistenceStrategy<PersonCertificationEntity> defaultStrategy;
    @Mock
    private PersistenceStrategy<PersonCertificationEntity> forceStrategy;
    
    private PersonCertificationServiceImpl service;
    
    @BeforeEach
    void setUp() {
        // Instanciación manual cuando hay @Qualifier
        service = new PersonCertificationServiceImpl(
            repository,
            mapper,
            validator,
            defaultStrategy,
            forceStrategy,
            detector
        );
    }
    
    @Test
    @DisplayName("create() - Debe retornar ID cuando los datos son válidos")
    void create_ShouldReturnId_WhenDataIsValid() {
        // Arrange
        when(repository.findById(1L)).thenReturn(Optional.of(mockPerson));
        when(defaultStrategy.apply(any(), any(), any(), any())).thenReturn(500L);
        
        // Act
        Long result = service.create(1L, mockRequest, PersistenceMode.DEFAULT);
        
        // Assert
        assertNotNull(result);
        assertEquals(500L, result);
        verify(defaultStrategy).apply(any(), any(), any(), any());
    }
}
```

---

### 5.2. Tests de Controller

**Estructura**:

```java
@WebMvcTest(controllers = CertificationsController.class)
@ActiveProfiles("test")
class CertificationsControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private PersonCertificationService service;
    
    @Test
    @DisplayName("POST /certifications - Should return 201 when request is valid")
    void createCertification_ShouldReturn201_WhenValid() throws Exception {
        // Arrange
        CreateRequest request = CreateRequest.builder()
            .code("CERT_IVA")
            .url("http://example.com/cert.pdf")
            .build();
        
        when(service.create(anyLong(), any(), any())).thenReturn(100L);
        
        // Act & Assert
        mockMvc.perform(post("/v2/people/1/certifications")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(content().string("100"));
    }
}
```

---


### 5.3. Swagger/OpenAPI

**¿Qué es y Por Qué Existe?**
Swagger (ahora OpenAPI) es una herramienta que genera documentación interactiva de la API REST automáticamente desde el código. Permite a los desarrolladores y QA explorar, probar y entender los endpoints sin necesidad de leer código o documentación externa.

**Beneficios principales:**

- Documentación automática: Se genera desde las anotaciones en el código
- Interfaz interactiva: Permite probar endpoints directamente desde el navegador
- Siempre actualizada: Si el código cambia, la documentación también
- Contrato de API: Define claramente qué espera y retorna cada endpoint
- Facilita integración: Los consumidores ven ejemplos y pueden probar la API
- Generación de clientes: Puede generar código cliente automáticamente

**Anotaciones obligatorias:**

- @Tag en cada controller (agrupar endpoints)
- @Operation en cada endpoint (describir qué hace)
- @ApiResponses en cada endpoint (documentar códigos HTTP)
- @Parameter en parámetros (path, query, header)
- @Schema en campos de DTOs (requests, responses)

**Buenas prácticas:**

- Describir códigos de respuesta específicos del contexto
- Proporcionar ejemplos realistas
- Documentar validaciones (min, max, required, etc.)
- Usar descripciones claras y concisas
- Mencionar reglas de negocio relevantes
- Explicar casos edge (null, valores especiales)

**Acceso:**

- Swagger UI: http://localhost:8080/swagger-ui.html
- OpenAPI JSON: http://localhost:8080/api-docs

**Estructura**:

```java
@PostMapping
@Operation(summary = "Create a new certification for a person")
@ApiResponses(value = {
    @ApiResponse(responseCode = "201", description = "Certification created successfully"),
    @ApiResponse(responseCode = "400", description = "Invalid request data"),
    @ApiResponse(responseCode = "404", description = "Person not found"),
    @ApiResponse(responseCode = "409", description = "Conflict detected (DEFAULT mode)"),
    @ApiResponse(responseCode = "500", description = "Internal server error")
})
public ResponseEntity<Long> createCertification(
    @PathVariable Long personId,
    @Parameter(description = "Persistence mode: DEFAULT or FORCE") 
    @RequestParam(defaultValue = "DEFAULT") PersistenceMode mode,
    @Valid @RequestBody CreateRequest request) {
    // ...
}
```

---

## 6. Git y Versionado

### 6.1. Conventional Commits // TODO revisar toda esta seccion

**Formato**:

```bash
<type>(<scope>): <description>

# Tipos
feat:     Nueva funcionalidad
fix:      Corrección de bug
refactor: Refactorización sin cambio funcional
docs:     Cambios en documentación
test:     Agregar o modificar tests
chore:    Tareas de mantenimiento

# Ejemplos
feat(certifications): implement PersonCertificationService
fix(certifications): correct date overlap detection logic
refactor(strategy): extract collision detection to separate class
docs(readme): update setup instructions
test(service): add tests for DEFAULT mode collision handling
```

---

### 6.2. Branching Strategy // TODO revisar toda esta seccion


```
main
  ↓
develop
  ↓
feature/TICKET-123-description
  ↓
(PR review)
  ↓
develop
  ↓
release/v1.8.12
  ↓
main (tag v1.8.12)
```

**Reglas**:
-  Crear branch antes de empezar a trabajar
-  Nombre: `feature/TICKET-123-short-description`
-  Commits pequeños y frecuentes
-  PR con descripción clara de cambios

---

### 6.3. Pull Requests

**Checklist antes de PR**:

# Checklist Antes de Pull Request (PR)

## 1. Library - Contratos y Modelos

- [ ] Nuevas rutas agregadas a PathV2

- [ ] Lista agregada en orden alfabético a Person.java
- [ ] CHANGELOG.md Actualizado con todos los cambios
- [ ] Version actualizada en POM.XML
---

## 2. Microservice - Implementación

- [ ] Version de libray actualizada en POM.XML de microservice
- [ ] Version de microservice actualizada en POM.XML de microservice

- [ ] NO usar @where() en entities
- [ ] NO extender `JpaRepositoryWithTypeOfManagement` (sistema legacy) en controllers

- [ ] `@Slf4j` para logging en services
- [ ] `@Transactional` en métodos del service que escriben
- [ ] `@Transactional(readOnly = true)` en métodos del service de solo lectura
- [ ] Log al inicio y final de métodos del servide, nivel INFO para operaciones importantes
- [ ] Selección de estrategia basada en `PersistenceMode` si aplica en el service

- [ ] Documentacion de swagger completa
- [ ] `consumes/produces = MediaType.APPLICATION_JSON_VALUE` en POST/PUT/PATCH/GET

- [ ] Tests Unitarios
- [ ] Tests de integracion (controller test)
---

## 5. Git y Versionado

### Commits
- [ ] **Conventional Commits**
  - [ ] Formato: `type(scope): message`
  - [ ] Types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`
  - [ ] Scope: módulo afectado (`certifications`, `library`, etc.)
  - [ ] Mensaje descriptivo y conciso

- [ ] **Commits atómicos**
  - [ ] Cada commit representa un cambio lógico completo
  - [ ] No mezclar cambios no relacionados

### Branch
- [ ] **Naming**
  - [ ] Formato: `feature/TICKET-123-description` o `fix/TICKET-123-description`
  - [ ] Descripción en kebab-case

- [ ] **Actualizada**
  - [ ] Merge/rebase con `develop` o `main` antes del PR
  - [ ] Sin conflictos

### Pull Request
- [ ] **Título descriptivo**
  - [ ] Formato: `[TICKET-123] Description of changes`
  - [ ] Describe qué se hizo, no cómo

- [ ] **Descripción completa**
  - [ ] Qué se cambió y por qué
  - [ ] Cómo probar los cambios
  - [ ] Breaking changes mencionados
  - [ ] Screenshots si aplica (UI changes)

---

## 6. Build y Compilación

- [ ] **Build exitoso**
  - [ ] `mvn clean compile` sin errores
  - [ ] `mvn clean install` sin errores

- [ ] **Tests pasan**
  - [ ] `mvn test` sin fallos
  - [ ] Cobertura de código aceptable (>80% en código nuevo)



---

## 7. Revisión Final

- [ ] **Código revisado**
  - [ ] Leí todo el código que estoy subiendo
  - [ ] Eliminé código de debug/testing
  - [ ] Eliminé console.logs, prints, etc.

- [ ] **Documentación actualizada**
  - [ ] README.md actualizado si es necesario
  - [ ] CHANGELOG.md actualizado (OBLIGATORIO en library)
  - [ ] Documentación técnica actualizada si es necesario



---

##  Checklist Rápido (Mínimo Obligatorio)

### Library
- [ ] CHANGELOG.md actualizado
- [ ] DTOs con validaciones correctas
- [ ] Enums con `@JsonValue` y `@JsonCreator`
- [ ] PathV2 actualizado (si aplica)

### Microservice
- [ ] Entities con soft-delete correcto
- [ ] Repositories con filtrado explícito
- [ ] Services con `@Transactional`
- [ ] Controllers con Swagger completo
- [ ] Tests unitarios para código nuevo

### General
- [ ] Build exitoso
- [ ] Tests pasan
- [ ] Commits con Conventional Commits
- [ ] PR con descripción completa

---

##  Antes de Hacer Click en "Create Pull Request"

**Pregunte:**
1. ¿Actualicé el CHANGELOG.md? (si es library)
2. ¿Todos los tests pasan?
3. ¿El build compila sin errores?
4. ¿Documenté con Swagger todos los endpoints nuevos?
5. ¿Revisé todo el código que estoy subiendo?

**Si responde SÍ a todo, está listo para crear el PR!** 

---

## Acuerdos del Equipo

### Compromisos

1. **Revisión de código obligatoria** antes de merge
2. **Tests obligatorios** para nueva funcionalidad
3. **Documentación actualizada** con cada cambio
4. **Seguir estos estándares** en todo nuevo desarrollo
5. **Comunicar desviaciones** y justificarlas

### Proceso de Revisión de Pares

1. Crear PR con descripción clara
2. Asignar revisor
3. Revisor verifica checklist
4. Discutir cambios necesarios
5. Aprobar y merge si depende del equipo

---

**Versión**: 2.0  
**Última actualización**: 2024-12-17  
**Próxima revisión**: Trimestral o cuando sea necesario

---

**Este documento es un acuerdo vivo del equipo. Cualquier cambio debe ser discutido y aprobado por todos los miembros.**