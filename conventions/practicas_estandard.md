# Guía de Estándares de Desarrollo - People Center

**Versión**: 2.0  
**Fecha**: 2024-12-15  
**Propósito**: Establecer estándares comunes para el desarrollo coherente entre todos los miembros del equipo

---

##  Tabla de Contenidos

### PARTE 1: FUNDAMENTOS
1. [Principios y Arquitectura](#1-principios-y-arquitectura)
   - Principios SOLID
   - Arquitectura de Capas
   - Flujo de Request

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

### 1.1. Principios SOLID

#### Separación de Responsabilidades (SRP)

Cada clase debe tener **una única responsabilidad**:

```
Controller  → Orquestación de requests HTTP (NO lógica de negocio)
Service     → Lógica de negocio y coordinación
Validator   → Reglas de negocio complejas
Repository  → Acceso a datos
Mapper      → Transformación DTO ↔ Entity
```

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

#### Excepciones, No Nulls

**Nunca retornar `null` en métodos públicos:**

```java
//  CORRECTO
public PersonEntity findById(Long id) {
    return repository.findById(id)
        .orElseThrow(() -> new NotFoundException("Person not found with id: " + id));
}

//  INCORRECTO
public PersonEntity findById(Long id) {
    return repository.findById(id).orElse(null); // NO!
}
```

---

### 1.2. Arquitectura de Capas

#### Flujo de una Request

```
1. HTTP Request
   ↓
2. Controller (@RestController)
   - Validación básica (@Valid)
   - Orquestación
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
7. Database
   ↓
8. HTTP Response
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

La **library** es un módulo compartido que contiene DTOs, Enums, Interfaces, Rutas API y Requests.

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

**Reglas para DTOs Públicos**:
- 4 anotaciones: `@Data`, `@Builder`, `@NoArgsConstructor`, `@AllArgsConstructor`
- `Long` para IDs (no `Integer`)
- `ZonedDateTime` para fechas (no `LocalDateTime`)
- Bean Validation con mensajes descriptivos
- NO incluir `deletedAt` (los consumidores solo ven registros activos)

#### DTOs de Auditoría/Administración

**Cuándo usar**: Endpoints de auditoría, historial o administración que necesitan mostrar el estado de los registros.

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

**Reglas para DTOs de Auditoría**:
- SÍ incluir `deletedAt`
- Incluir método `isActive()` para verificación
- Incluir método `getStatus()` para visualización
- Usar sufijo `Audit` o `Admin` en el nombre

#### Tabla de Decisión: ¿Incluir deletedAt?

| Tipo de DTO | Incluir deletedAt | Razón | Ejemplo de Uso |
|-------------|-------------------|-------|----------------|
| **DTO Público** | NO | Los consumidores solo necesitan registros activos | `GET /v2/people/{id}/certifications` |
| **DTO de Auditoría** | SÍ | Administradores necesitan ver estado completo | `GET /v2/people/{id}/certifications/history` |
| **DTO de Admin** | SÍ | Dashboards administrativos requieren estado | `GET /admin/certifications` |

#### Ejemplo Completo en Service

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

#### Mapper para Ambos Tipos

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
    
    // NO incluir id ni createdAt (se generan en backend)
}
```

**Naming Convention**:
- `Create{Entity}Request` - Para crear
- `Update{Entity}Request` - Para actualizar completo
- `Patch{Entity}Request` - Para actualizar parcial

---

### 2.3. Enums

**Ubicación**: `library/model/.../enums/`

**Estructura**:

```java
@Getter
@AllArgsConstructor
public enum PersistenceMode {
    
    DEFAULT("DEFAULT"),
    FORCE("FORCE");
    
    @JsonValue
    private final String value;
    
    @JsonCreator
    public static PersistenceMode fromValue(String value) {
        for (PersistenceMode mode : PersistenceMode.values()) {
            if (mode.value.equalsIgnoreCase(value)) {
                return mode;
            }
        }
        throw new IllegalArgumentException("Invalid PersistenceMode: " + value);
    }
}
```

**Reglas**:
-  Usar `@JsonValue` para serialización
-  Usar `@JsonCreator` para deserialización
-  Implementar `fromValue()` con validación
-  Nombres en UPPER_CASE

---

### 2.4. Deserializers - Validación de Fechas

**Ubicación**: `library/model/.../deserializer/`

**Problema que Resuelve**:
-  Evita timestamps numéricos (`1234567890`)
-  Evita fechas sin hora (`"2024-01-15"`)
-  Evita strings arbitrarios (`"invalid"`)
-  Solo acepta ISO-8601 completo (`"2024-01-15T10:00:00Z"`)

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

**Reglas**:
-  Usar en **todos** los campos `ZonedDateTime` de Requests
-  Aplicar con `@JsonDeserialize(using = StrictZonedDateTimeDeserializer.class)`
-  NO usar en DTOs (solo en Requests)

---

### 2.5. PathV2 - Rutas de API

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
-  Mantener orden alfabético
-  Usar kebab-case en URLs (`/tax-activities`, no `/taxActivities`)
-  Naming: `{ENTITY}_PATH`
-  Usar `BASE_PATH` para construir

---

### 2.6. Person.java - DTO Principal

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

**Reglas**:
-  Usar `@Valid` para validación en cascada
-  Usar `@Builder.Default` con `new ArrayList<>()`
-  Mantener orden alfabético
-  Inicializar siempre (evita NPE)

---

### 2.7. CHANGELOG.md - OBLIGATORIO Antes de PR

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

**Categorías**:
- **Added** - Nuevos DTOs, Enums, Requests, Paths
- **Changed** - Modificaciones a modelos existentes
- **Fixed** - Correcciones de bugs
- **Removed** - Eliminación de código

**Workflow Antes de PR**:
1. Identificar cambios en la library
2. Determinar nueva versión (X.Y.Z)
3. Actualizar `library/CHANGELOG.md` (al INICIO)
4. Actualizar `library/pom.xml` con nueva versión

**Reglas de Oro**:
-  Actualizar SIEMPRE antes de PR
-  Agregar al INICIO del archivo (no al final)
-  Incrementar versión correctamente
-  Sincronizar con pom.xml

---

### Workflow Completo: Agregar Nueva Entidad en Library

#### Paso 1: Crear DTO
```java
// library/model/.../model/TaxActivity.java
@Data @NoArgsConstructor @AllArgsConstructor @Builder
public class TaxActivity {
    private Long id;
    @NotNull(message = "Activity code is required")
    private String activityCode;
    private ZonedDateTime startDate;
}
```

#### Paso 2: Crear Request
```java
// library/model/.../requests/CreateTaxActivityRequest.java
@Data @Builder
public class CreateTaxActivityRequest {
    @NotBlank(message = "Activity code is required")
    private String activityCode;
    @NotNull(message = "Start date is required")
    @JsonDeserialize(using = StrictZonedDateTimeDeserializer.class)
    private ZonedDateTime startDate;
}
```

#### Paso 3: Crear Enum (si es necesario)
```java
// library/model/.../enums/TaxActivityType.java
@Getter @AllArgsConstructor
public enum TaxActivityType {
    PRIMARY("PRIMARY"), SECONDARY("SECONDARY");
    @JsonValue private final String value;
    @JsonCreator
    public static TaxActivityType fromValue(String value) { /* ... */ }
}
```

#### Paso 4: Agregar Ruta en PathV2
```java
public static final String TAX_ACTIVITIES_PATH = BASE_PATH + "/{id}/tax-activities";
```

#### Paso 5: Agregar Lista en Person.java
```java
@Valid
@Builder.Default
private List<TaxActivity> taxActivities = new ArrayList<>();
```

#### Paso 6: Actualizar CHANGELOG.md
```markdown
*Version 0.11.0-SNAPSHOT-CON27-2 - Released 2025-12-16*
- **Added**
  - TaxActivity Model
  - CreateTaxActivityRequest Model
  - TaxActivityType Enum
  - TAX_ACTIVITIES_PATH in PathV2
  - taxActivities list in Person Model
```

#### Paso 7: Actualizar pom.xml
```xml
<version>0.11.0-SNAPSHOT</version>
```

---

## 3. Implementación en Microservice

### 3.1. Entities y Repositories

#### Entity

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

**Reglas**:
-  Implementar interfaces `HasId`, `HasDeletedAt`
-  Usar `@Where(clause = "deleted_at IS NULL")` para soft-delete
-  `@CreationTimestamp` para `createdAt`
-  `@JsonBackReference` en relaciones bidireccionales

#### Repository

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

**Reglas**:
-  Dos métodos: JPA (activos) + Native (incluye eliminados)
-  Query nativa documentada con JavaDoc
-  Usar `@Param` en queries

---

### 3.2. Mappers

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

**Reglas**:
-  `@Mapper(componentModel = "spring")`
-  Campos autogenerados con `ignore = true`
-  Conversiones de tipo documentadas

---

### 3.3. Validators

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

**Reglas**:
-  Validaciones de reglas de negocio complejas
-  Mensajes de error claros
-  Logging con nivel `warn`

---

### 3.4. Services

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
        
        PersonCertificationEntity entity = repository
            .findByIdAndPersonIdIncludingDeleted(certificationId, personId)
            .orElseThrow(() -> new ResourceNotFoundException(
                String.format("Certification %d not found for person %d", 
                    certificationId, personId)));
        
        entity.setDeletedAt(ZonedDateTime.now());
        repository.save(entity);
        
        log.info("Certification {} soft deleted", certificationId);
        return certificationId;
    }
}
```

**Reglas**:
-  Interface con JavaDoc detallado
-  `@Transactional` en métodos que escriben
-  `@Transactional(readOnly = true)` en consultas
-  Validaciones antes de persistir
-  Logging en inicio/fin de operaciones
-  Excepciones con mensajes descriptivos
-  NO retorna `null` en métodos públicos
-  Constructor explícito cuando hay `@Qualifier`

---

### 3.5. Controllers (REST API)

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

**Reglas - MediaType**:

| Método HTTP | consumes | produces | Ejemplo |
|-------------|----------|----------|---------|
| **POST** |  Sí |  Sí | Crear entidad |
| **PUT** |  Sí |  Sí | Actualizar completo |
| **PATCH** |  Sí |  Sí | Actualizar parcial |
| **GET** |  No |  Sí | Obtener entidad |
| **DELETE** |  No |  Opcional | Eliminar |

**Reglas Generales**:
-  Solo orquestación, sin lógica de negocio
-  `@Valid` en request bodies
-  Documentación Swagger completa
-  HTTP status codes correctos (201 para POST, 200 para GET/DELETE)
-  Usar `MediaType.APPLICATION_JSON_VALUE`
-  NO hace checks de null

---

## 4. Patrones de Implementación

### 4.1. Strategy Pattern (Collision)

#### Visión General

El **Strategy Pattern** permite manejar colisiones de datos con dos estrategias:
- **DEFAULT**: Rechaza si hay conflictos (lanza `ConflictException`)
- **FORCE**: Soft-delete de conflictos y procede

#### Arquitectura

```
strategy/
├── CollisionDetector.java          // Interface para detectar colisiones
├── PersistenceStrategy.java        // Interface para estrategias
├── impl/
│   ├── DefaultPersistenceStrategy.java
│   └── ForcePersistenceStrategy.java
└── detectors/
    └── CertificationCollisionDetector.java
```

#### Interfaces

**CollisionDetector**:

```java
public interface CollisionDetector<T> {
    /**
     * Detects if there is a collision between new entity and existing entities.
     * 
     * @param newEntity New entity to check
     * @param existing Existing entities
     * @return List of conflicting entities (empty if no collision)
     */
    Collection<T> detectCollisions(T newEntity, Collection<T> existing);
}
```

**PersistenceStrategy**:

```java
public interface PersistenceStrategy<T extends HasDeletedAt> {
    /**
     * Applies persistence strategy.
     * 
     * @param newEntity New entity to persist
     * @param existing Existing entities
     * @param saver Function to save entity
     * @param detector Collision detector
     * @return ID of persisted entity
     */
    Long apply(T newEntity, Collection<T> existing, 
               Consumer<T> saver, CollisionDetector<T> detector);
}
```

#### Implementaciones

**DefaultPersistenceStrategy**:

```java
@Component("defaultPersistenceStrategy")
@Slf4j
public class DefaultPersistenceStrategy<T extends HasDeletedAt> 
        implements PersistenceStrategy<T> {
    
    @Override
    public Long apply(T newEntity, Collection<T> existing, 
                     Consumer<T> saver, CollisionDetector<T> detector) {
        log.info("Applying DEFAULT strategy");
        
        Collection<T> conflicts = detector.detectCollisions(newEntity, existing);
        
        if (!conflicts.isEmpty()) {
            log.warn("Collision detected in DEFAULT mode. Conflicts: {}", 
                conflicts.stream().map(HasId::getId).collect(Collectors.toList()));
            
            throw new ConflictException(
                "Collision detected. Use FORCE mode to override.",
                conflicts.stream().map(HasId::getId).collect(Collectors.toList())
            );
        }
        
        saver.accept(newEntity);
        log.info("Entity persisted successfully with id: {}", newEntity.getId());
        return newEntity.getId();
    }
}
```

**ForcePersistenceStrategy**:

```java
@Component("forcePersistenceStrategy")
@Slf4j
public class ForcePersistenceStrategy<T extends HasDeletedAt> 
        implements PersistenceStrategy<T> {
    
    @Override
    public Long apply(T newEntity, Collection<T> existing, 
                     Consumer<T> saver, CollisionDetector<T> detector) {
        log.info("Applying FORCE strategy");
        
        Collection<T> conflicts = detector.detectCollisions(newEntity, existing);
        
        if (!conflicts.isEmpty()) {
            log.info("Soft-deleting {} conflicting entities", conflicts.size());
            ZonedDateTime deletionTimestamp = ZonedDateTime.now();
            conflicts.forEach(c -> c.setDeletedAt(deletionTimestamp));
            conflicts.forEach(saver);
        }
        
        saver.accept(newEntity);
        log.info("Entity persisted successfully with id: {}", newEntity.getId());
        return newEntity.getId();
    }
}
```

#### Detector

**CertificationCollisionDetector**:

```java
@Component
@Slf4j
public class CertificationCollisionDetector 
        implements CollisionDetector<PersonCertificationEntity> {
    
    @Override
    public Collection<PersonCertificationEntity> detectCollisions(
            PersonCertificationEntity newEntity,
            Collection<PersonCertificationEntity> existing) {
        
        log.debug("Detecting collisions for certification code: {}", 
            newEntity.getCertificationCode());
        
        return existing.stream()
            .filter(e -> e.getCertificationCode().equals(newEntity.getCertificationCode()))
            .filter(e -> !e.isDeleted())
            .collect(Collectors.toList());
    }
}
```

**Reglas para Detectors**:
-  NO acceso a DB (solo comparación en memoria)
-  Filtrar solo activos (`!isDeleted()`)
-  Logging con nivel `debug`

#### Uso en Service

```java
@Service
public class PersonCertificationServiceImpl {
    
    private final PersistenceStrategy<PersonCertificationEntity> defaultStrategy;
    private final PersistenceStrategy<PersonCertificationEntity> forceStrategy;
    private final CollisionDetector<PersonCertificationEntity> detector;
    
    // Constructor explícito con @Qualifier
    public PersonCertificationServiceImpl(
            @Qualifier("defaultPersistenceStrategy") 
            PersistenceStrategy<PersonCertificationEntity> defaultStrategy,
            @Qualifier("forcePersistenceStrategy") 
            PersistenceStrategy<PersonCertificationEntity> forceStrategy,
            CollisionDetector<PersonCertificationEntity> detector) {
        this.defaultStrategy = defaultStrategy;
        this.forceStrategy = forceStrategy;
        this.detector = detector;
    }
    
    @Transactional
    public Long create(Long personId, CreateRequest request, PersistenceMode mode) {
        // ... validaciones ...
        
        PersistenceStrategy<PersonCertificationEntity> strategy = 
            mode == PersistenceMode.FORCE ? forceStrategy : defaultStrategy;
        
        return strategy.apply(newEntity, existing, repository::save, detector);
    }
}
```

**Reglas**:
-  Usar `@Qualifier` para inyección
-  Constructor explícito (NO `@RequiredArgsConstructor`)
-  Seleccionar estrategia según `PersistenceMode`

---

### 4.2. Soft-Delete con @Where

#### Configuración en Entity

```java
@Entity
@Where(clause = "deleted_at IS NULL")  // ← CLAVE: Filtra eliminados automáticamente
public class PersonCertificationEntity implements HasDeletedAt {
    
    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;
    
    @Override
    public boolean isDeleted() {
        return deletedAt != null;
    }
}
```

#### Comportamiento de @Where

```java
// Con @Where(clause = "deleted_at IS NULL")

// Método JPA estándar
repository.findById(1L);
// SQL: SELECT * FROM person_certification WHERE id = 1 AND deleted_at IS NULL

// findAll()
repository.findAll();
// SQL: SELECT * FROM person_certification WHERE deleted_at IS NULL
```

**Resultado**: Todos los métodos JPA automáticamente excluyen registros eliminados.

#### Problema: Recuperar Registros Eliminados

**Escenario**: Necesitas acceder a registros eliminados para:
-  Operaciones idempotentes (DELETE múltiple)
-  Auditoría e historial
-  Recuperación de datos

**Solución: Query Nativa**

```java
@Repository
public interface PersonCertificationRepository extends JpaRepository<PersonCertificationEntity, Long> {
    
    //  Este método NO retorna eliminados (usa @Where)
    Optional<PersonCertificationEntity> findByIdAndPersonId_Id(Long id, Long personId);
    
    /**
     *  Este método SÍ retorna eliminados (bypasea @Where con query nativa)
     */
    @Query(value = "SELECT * FROM person_certification WHERE id = :id AND person_id = :personId", 
           nativeQuery = true)
    Optional<PersonCertificationEntity> findByIdAndPersonIdIncludingDeleted(
        @Param("id") Long id,
        @Param("personId") Long personId);
}
```

#### Comparación: JPA vs Native Query

| Aspecto | Método JPA | Query Nativa |
|---------|------------|--------------|
| **Sintaxis** | `findByIdAndPersonId_Id(...)` | `@Query(value = "SELECT * ...", nativeQuery = true)` |
| **Respeta @Where** |  Sí |  No (bypasea) |
| **Retorna eliminados** |  No |  Sí |
| **Cuándo usar** | Operaciones normales | Acceso a eliminados |

#### Uso en Service

```java
@Service
public class PersonCertificationServiceImpl {
    
    // Listar → Método JPA (solo activos)
    @Transactional(readOnly = true)
    public List<PersonCertification> getValidCertifications(Long personId) {
        return repository.findByPersonId_Id(personId)
            .stream()
            .map(mapper::toDto)
            .collect(Collectors.toList());
    }
    
    // Get by ID → Query nativa (incluye eliminados para auditoría)
    @Transactional(readOnly = true)
    public PersonCertification getById(Long personId, Long certificationId) {
        return repository.findByIdAndPersonIdIncludingDeleted(certificationId, personId)
            .map(mapper::toDto)
            .orElseThrow(() -> new ResourceNotFoundException(...));
    }
    
    // Delete → Query nativa (idempotente)
    @Transactional
    public Long delete(Long personId, Long certificationId) {
        PersonCertificationEntity entity = repository
            .findByIdAndPersonIdIncludingDeleted(certificationId, personId)
            .orElseThrow(() -> new ResourceNotFoundException(...));
        
        entity.setDeletedAt(ZonedDateTime.now());
        repository.save(entity);
        return certificationId;
    }
}
```

#### Tabla de Decisión

| Operación | Método a Usar | Razón |
|-----------|---------------|-------|
| **Listar todos** | Método JPA | Solo activos |
| **Buscar por criterio** | Método JPA | Solo activos |
| **Get by ID** | Query nativa | Incluir eliminados (auditoría) |
| **Delete** | Query nativa | Idempotente |
| **Auditoría** | Query nativa | Acceso a historial completo |

---

### 4.3. @Transactional

#### Reglas Básicas

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

#### Tabla de Uso

| Operación | Anotación | Razón |
|-----------|-----------|-------|
| **CREATE** | `@Transactional` | Escribe en DB |
| **UPDATE** | `@Transactional` | Escribe en DB |
| **DELETE** | `@Transactional` | Escribe en DB (soft-delete) |
| **GET** | `@Transactional(readOnly = true)` | Solo lectura, optimización |
| **LIST** | `@Transactional(readOnly = true)` | Solo lectura, optimización |

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

#### Excepciones del Proyecto

| Excepción | HTTP Status | Cuándo Usar | Dónde Lanzar |
|-----------|-------------|-------------|--------------|
| **ResourceNotFoundException** | 404 | Entity no encontrada | Service |
| **ConflictException** | 409 | Colisión en DEFAULT mode | Strategy |
| **IllegalArgumentException** | 400 | Validaciones de negocio | Validator, Service |
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

#### Reglas

-  Loggear ANTES de lanzar excepción
-  Mensajes descriptivos y específicos
-  Incluir IDs relevantes en el mensaje
-  Usar excepción apropiada según contexto
-  NO usar excepciones genéricas (`Exception`, `RuntimeException`)

---

## 5. Testing y Documentación

### 5.1. Tests de Service

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

**Reglas**:
-  NO usar `@InjectMocks` cuando hay `@Qualifier`
-  Instanciar service manualmente en `@BeforeEach`
-  Usar `@DisplayName` descriptivos
-  Patrón Arrange-Act-Assert

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

### 5.3. JavaDoc

**Estructura**:

```java
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
 * <p><b>Collision Strategies:</b></p>
 * <ul>
 * <li>DEFAULT: Rejects if collision exists (throws ConflictException)</li>
 * <li>FORCE: Soft-deletes conflicting records and proceeds</li>
 * </ul>
 * 
 * @param personId Person ID
 * @param request Certification data
 * @param mode Collision handling strategy
 * @return ID of the created certification
 * @throws ResourceNotFoundException if person not found
 * @throws IllegalArgumentException if validation fails
 * @throws ConflictException if collision detected (DEFAULT mode)
 */
public Long create(Long personId, CreateRequest request, PersistenceMode mode);
```

---

### 5.4. Swagger/OpenAPI

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

### 6.1. Conventional Commits

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

### 6.2. Branching Strategy

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

#### Library
- [ ]  CHANGELOG.md actualizado (al INICIO)
- [ ]  pom.xml con nueva versión
- [ ]  DTOs sin `deletedAt`
- [ ]  Requests con validaciones
- [ ]  Enums con `@JsonCreator`
- [ ]  PathV2 en orden alfabético

#### Microservice
- [ ]  Entity con `@Where` (si soft-delete)
- [ ]  Repository con query nativa (si aplica)
- [ ]  Mapper con `ignore = true` en campos autogenerados
- [ ]  Validator con reglas de negocio
- [ ]  Service con `@Transactional`
- [ ]  Controller con MediaType
- [ ]  Tests de Service y Controller
- [ ]  JavaDoc en métodos públicos
- [ ]  Swagger/OpenAPI completo

---

## 7. Checklist de Revisión

### Por Capa

#### Entity
- [ ] Implementa `HasId`, `HasDeletedAt` (si aplica)
- [ ] Usa `@Where(clause = "deleted_at IS NULL")` para soft-delete
- [ ] `@CreationTimestamp` para `createdAt`
- [ ] `@JsonBackReference` en relaciones bidireccionales

#### Repository
- [ ] Extiende `JpaRepository`
- [ ] Queries personalizadas documentadas con JavaDoc
- [ ] Usa `@Param` en queries nativas
- [ ] Query para incluir eliminados (si aplica)

#### Mapper
- [ ] `@Mapper(componentModel = "spring")`
- [ ] Campos autogenerados con `ignore = true`

#### Validator
- [ ] Validaciones de reglas de negocio complejas
- [ ] Mensajes de error claros
- [ ] Logging con nivel `warn`

#### Service
- [ ] Interface con JavaDoc detallado
- [ ] `@Transactional` en métodos que escriben
- [ ] `@Transactional(readOnly = true)` en consultas
- [ ] Validaciones antes de persistir
- [ ] Logging en inicio/fin de operaciones
- [ ] Excepciones con mensajes descriptivos
- [ ] NO retorna `null` en métodos públicos

#### Controller
- [ ] Solo orquestación, sin lógica de negocio
- [ ] `@Valid` en request bodies
- [ ] Documentación Swagger completa
- [ ] HTTP status codes correctos
- [ ] MediaType en `@PostMapping`, `@GetMapping`, etc.

#### Tests
- [ ] Tests de service con mocks
- [ ] Tests de controller con MockMvc
- [ ] Tests de validaciones
- [ ] Coverage > 80%

---

## Recursos Adicionales

### Documentos del Proyecto

- `SECCION_ESTRATEGIA_COLISIONES.md` - Guía completa del patrón Strategy
- `SECCION_LIBRARY_ESTRUCTURA.md` - Guía detallada de Library
- `ANALISIS_REORGANIZACION.md` - Análisis de esta reorganización

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
5. Aprobar y merge

---

**Versión**: 2.0  
**Última actualización**: 2024-12-15  
**Próxima revisión**: Trimestral o cuando sea necesario

---

**Este documento es un acuerdo vivo del equipo. Cualquier cambio debe ser discutido y aprobado por todos los miembros.**