# Patrón de Arquitectura: Certification & PersonCertification

## Guía de Referencia para Implementación de Nuevos Servicios

Este documento describe la arquitectura completa y los patrones utilizados en la implementación de `Certification` y `PersonCertification`. Úsalo como referencia para implementar nuevos servicios con la misma estructura y consistencia.

---

## 📁 Estructura de Archivos

### Ubicación de Archivos por Capa

```
people-center/
├── library/
│   └── model/
│       └── src/main/java/ar/com/bds/lib/peoplecenter/model/
│           ├── Certification.java                    # DTO de respuesta (catálogo)
│           ├── PersonCertification.java              # DTO de respuesta (relación)
│           └── requests/
│               └── CreatePersonCertificationRequest.java  # DTO de entrada
└── microservice/
    └── src/main/java/ar/com/bds/people/center/
        ├── entity/
        │   ├── CertificationEntity.java              # Entidad JPA (catálogo)
        │   └── PersonCertificationEntity.java        # Entidad JPA (relación)
        ├── repository/
        │   ├── CertificationRepository.java          # Repositorio (catálogo)
        │   └── PersonCertificationRepository.java    # Repositorio (relación)
        ├── service/
        │   ├── PersonCertificationService.java       # Interface del servicio
        │   └── impl/
        │       └── PersonCertificationServiceImpl.java  # Implementación
        ├── controller/
        │   └── CertificationsController.java         # REST Controller
        └── mapper/
            └── PersonCertificationMapper.java        # MapStruct Mapper
```

---

## 🏗️ Capa 1: Entidades (JPA Entities)

### Patrón: Entidad de Catálogo (`CertificationEntity`)

**Propósito**: Representa datos de referencia/catálogo que raramente cambian.

```java
@Getter
@Setter
@Entity
@Table(name = "certification")
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Where(clause = "deleted_at IS NULL")  // ← Soft delete automático
public class CertificationEntity {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "code", nullable = false, unique = true, length = 50)
    private String code;  // ← Identificador de negocio
    
    @Column(name = "name", nullable = false, length = 100)
    private String name;
    
    @Column(name = "created_at", updatable = false, nullable = false)
    @CreationTimestamp
    private ZonedDateTime createdAt;
    
    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;  // ← Soft delete
}
```

**Características clave:**
- ✅ `@Where(clause = "deleted_at IS NULL")` - Filtra registros eliminados automáticamente
- ✅ `code` único - Identificador de negocio
- ✅ `@CreationTimestamp` - Auditoría automática
- ✅ `deletedAt` - Soft delete con fecha

### Patrón: Entidad de Relación (`PersonCertificationEntity`)

**Propósito**: Relaciona una persona con un catálogo, agregando datos específicos de la relación.

```java
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
public class PersonCertificationEntity implements HasId, HasDeleted, HasType {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    // ← Relación con Person
    @JoinColumn(name = "person_id", nullable = false)
    @ManyToOne
    @JsonBackReference
    private PersonEntity personId;
    
    // ← Relación con Catálogo
    @JoinColumn(name = "certification_id", nullable = false)
    @ManyToOne
    private CertificationEntity certification;
    
    // ← Datos específicos de la relación
    @Column(name = "url", nullable = false, length = 250)
    private String url;
    
    @Column(name = "percentage", precision = 5, scale = 4)
    private BigDecimal percentage;
    
    @Column(name = "aliquot", precision = 5, scale = 4)
    private BigDecimal aliquot;
    
    @Column(name = "start_date")
    private ZonedDateTime startDate;
    
    @Column(name = "end_date")
    private ZonedDateTime endDate;
    
    // ← Auditoría
    @Column(name = "created_at", updatable = false, nullable = false)
    @CreationTimestamp
    private ZonedDateTime createdAt;
    
    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;
    
    // ← Implementación de interfaces
    @Override
    public boolean isDeleted() {
        return deletedAt != null;
    }
    
    @Override
    public void setDeleted(boolean deleted) {
        this.deletedAt = deleted ? ZonedDateTime.now() : null;
    }
    
    @Override
    public String getType() {
        return certification.getCode();
    }
}
```

**Características clave:**
- ✅ Implementa interfaces: `HasId`, `HasDeleted`, `HasType`
- ✅ Índices en claves foráneas para performance
- ✅ `@JsonBackReference` en `personId` para evitar recursión infinita
- ✅ Métodos `@Override` para lógica personalizada (conversión fecha ↔ boolean)

---

## 🗄️ Capa 2: Repositorios (Spring Data JPA)

### Patrón: Repositorio de Catálogo (`CertificationRepository`)

```java
@Repository
public interface CertificationRepository extends JpaRepository<CertificationEntity, Long> {
    
    Optional<CertificationEntity> findByCode(String code);
}
```

**Características:**
- ✅ Método de búsqueda por identificador de negocio (`code`)
- ✅ Extiende `JpaRepository` para operaciones CRUD básicas

### Patrón: Repositorio de Relación (`PersonCertificationRepository`)

```java
@Repository
public interface PersonCertificationRepository extends 
                JpaRepositoryWithTypeOfManagement<PersonCertificationEntity, Long> {
    
    // ← Búsqueda por ID de certificación Y ID de persona (seguridad)
    Optional<PersonCertificationEntity> findByIdAndPersonId_Id(Long id, Long personId);
    
    // ← Búsqueda por persona (todas las certificaciones)
    Set<PersonCertificationEntity> findByPersonId(PersonEntity personId);
    
    // ← Query personalizada con lógica de negocio
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

**Características clave:**
- ✅ Extiende `JpaRepositoryWithTypeOfManagement` para soporte de `TypeOfManagement`
- ✅ `findByIdAndPersonId_Id` - Validación de pertenencia (seguridad)
- ✅ Notación `personId_Id` para navegar propiedades anidadas
- ✅ `@Query` personalizada para lógica de negocio compleja

---

## 📦 Capa 3: DTOs (Data Transfer Objects)

### Patrón: DTO de Respuesta de Catálogo (`Certification`)

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Certification {
    private String code;
    private String name;
    // ← Sin validaciones (solo para respuestas)
}
```

### Patrón: DTO de Respuesta de Relación (`PersonCertification`)

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class PersonCertification {
    private Integer id;
    private String url;
    private Certification certification;  // ← DTO anidado
    private BigDecimal aliquot;
    private BigDecimal percentage;
    private ZonedDateTime startDate;
    private ZonedDateTime endDate;
    private ZonedDateTime createdAt;
    private ZonedDateTime deletedAt;
    // ← Sin validaciones (solo para respuestas)
    // ← Sin @JsonFormat en ZonedDateTime (usa JavaTimeModule)
}
```

### Patrón: DTO de Request (`CreatePersonCertificationRequest`)

```java
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
@Schema(description = "Request to create a new person certification")
public class CreatePersonCertificationRequest {
    
    // ← Validaciones completas
    @NotBlank(message = "Certification code is required")
    @Size(max = 50, message = "Certification code must not exceed 50 characters")
    @Schema(description = "...", example = "CERT_IVA", required = true)
    private String certificationCode;
    
    @NotBlank(message = "URL is required")
    @Size(max = 250, message = "URL must not exceed 250 characters")
    @Schema(description = "...", example = "https://...", required = true)
    private String url;
    
    @DecimalMin(value = "0.0", inclusive = true, message = "Percentage must be between 0 and 1")
    @DecimalMax(value = "1.0", inclusive = true, message = "Percentage must be between 0 and 1")
    @Schema(description = "...", example = "0.2150", minimum = "0", maximum = "1")
    private BigDecimal percentage;
    
    @DecimalMin(value = "0.0", inclusive = true, message = "Aliquot must be between 0 and 1")
    @DecimalMax(value = "1.0", inclusive = true, message = "Aliquot must be between 0 and 1")
    @Schema(description = "...", example = "0.1050", minimum = "0", maximum = "1")
    private BigDecimal aliquot;
    
    @Schema(description = "...", example = "2024-01-15T10:30:00.000-03:00", type = "string", format = "date-time")
    private ZonedDateTime startDate;
    
    @Schema(description = "...", example = "2024-12-31T23:59:59.999-03:00", type = "string", format = "date-time")
    private ZonedDateTime endDate;
}
```

**Características clave:**
- ✅ Validaciones Bean Validation (`@NotBlank`, `@Size`, `@DecimalMin/Max`)
- ✅ `@Schema` para documentación OpenAPI/Swagger
- ✅ Mensajes de error descriptivos
- ✅ Límites de `@Size` coinciden con `@Column(length=...)` de la entidad
- ✅ Sin `@JsonFormat` en `ZonedDateTime` (usa `JavaTimeModule`)

---

## 🔄 Capa 4: Mappers (MapStruct)

### Patrón: Mapper (`PersonCertificationMapper`)

```java
@Mapper(componentModel = "spring")
public interface PersonCertificationMapper {
    
    // ← Entity → DTO
    @Mapping(source = "id", target = "id")
    @Mapping(source = "certification", target = "certification")
    PersonCertification toDto(PersonCertificationEntity entity);
    
    // ← DTO → Entity (si es necesario)
    PersonCertificationEntity toEntity(PersonCertification dto);
}
```

**Características:**
- ✅ `@Mapper(componentModel = "spring")` - Inyección de dependencias
- ✅ Mapeo automático de campos con mismo nombre
- ✅ `@Mapping` explícito para relaciones anidadas

---

## 💼 Capa 5: Servicios (Business Logic)

### Patrón: Interface del Servicio (`PersonCertificationService`)

```java
public interface PersonCertificationService {
    Long create(Long personId, CreatePersonCertificationRequest request, TypeOfManagement typeOfManagement);
    List<PersonCertification> getValidCertifications(Long personId);
    PersonCertification getById(Long personId, Long certificationId);
    Long delete(Long personId, Long certificationId);
}
```

### Patrón: Implementación del Servicio (`PersonCertificationServiceImpl`)

```java
@Service
@Slf4j
@RequiredArgsConstructor
public class PersonCertificationServiceImpl implements PersonCertificationService {
    
    private final PersonCertificationRepository personCertificationRepository;
    private final CertificationRepository certificationRepository;
    private final PeopleCenterRepository peopleCenterRepository;
    private final PersonCertificationMapper mapper;
    
    // ← CREATE con validación y TypeOfManagement
    @Override
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Long create(Long personId, CreatePersonCertificationRequest request, TypeOfManagement typeOfManagement) {
        log.info("Creating certification for person: {} with strategy: {}", personId, typeOfManagement);
        
        validateRequest(request);  // ← Validación de negocio
        
        PersonEntity person = peopleCenterRepository.findById(personId)
                .orElseThrow(() -> new ResourceNotFoundException("Person not found with id: " + personId));
        
        CertificationEntity certification = certificationRepository.findByCode(request.getCertificationCode())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Certification not found with code: " + request.getCertificationCode()));
        
        PersonCertificationEntity entity = PersonCertificationEntity.builder()
                .personId(person)
                .url(request.getUrl())
                .certification(certification)
                .percentage(request.getPercentage())
                .aliquot(request.getAliquot())
                .startDate(request.getStartDate())
                .endDate(request.getEndDate())
                .build();
        
        Set<PersonCertificationEntity> existingCertifications = personCertificationRepository.findByPersonId(person);
        
        PersonCertificationEntity saved = personCertificationRepository.saveWithTypeOfManagement(entity,
                existingCertifications, typeOfManagement);
        
        log.info("Certification created with id: {}", saved.getId());
        return saved.getId();
    }
    
    // ← READ (lista) - Solo lectura
    @Override
    @Transactional(readOnly = true)
    public List<PersonCertification> getValidCertifications(Long personId) {
        log.info("Getting valid certifications for person: {}", personId);
        
        ZonedDateTime now = ZonedDateTime.now();
        List<PersonCertificationEntity> entities = personCertificationRepository
                .findValidCertifications(personId, now);
        
        return entities.stream()
                .map(mapper::toDto)
                .collect(Collectors.toList());
    }
    
    // ← READ (por ID) - Solo lectura con validación de pertenencia
    @Override
    @Transactional(readOnly = true)
    public PersonCertification getById(Long personId, Long certificationId) {
        log.info("Getting certification {} for person: {}", certificationId, personId);
        
        PersonCertificationEntity entity = personCertificationRepository
                .findByIdAndPersonId_Id(certificationId, personId)
                .orElseThrow(() -> new ResourceNotFoundException(
                        String.format("Certification not found with id %d for person %d", certificationId, personId)));
        
        return mapper.toDto(entity);
    }
    
    // ← DELETE (soft delete)
    @Override
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Long delete(Long personId, Long certificationId) {
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
    
    // ← Validación de negocio privada
    private void validateRequest(CreatePersonCertificationRequest request) {
        if (request.getPercentage() != null && request.getAliquot() != null) {
            throw new IllegalArgumentException("Only one of 'percentage' or 'aliquot' can be specified");
        }
        
        if (request.getStartDate() != null && request.getEndDate() != null) {
            if (request.getStartDate().isAfter(request.getEndDate())) {
                throw new IllegalArgumentException("Start date must be before end date");
            }
        }
    }
}
```

**Características clave:**
- ✅ `@Transactional(isolation = SERIALIZABLE)` en operaciones de escritura
- ✅ `@Transactional(readOnly = true)` en operaciones de lectura
- ✅ Logging en todos los métodos públicos
- ✅ Validación de pertenencia (`findByIdAndPersonId_Id`)
- ✅ Validaciones de negocio en método privado
- ✅ Uso de `TypeOfManagement` para gestión de duplicados
- ✅ Soft delete con `setDeletedAt()`

---

## 🎮 Capa 6: Controllers (REST API)

### Patrón: Controller (`CertificationsController`)

```java
@RestController
@RequestMapping("/v2/people/{personId}/certifications")
@Tag(name = "People Center - Certifications")
@RequiredArgsConstructor
@Slf4j
@Validated
public class CertificationsController {
    
    private final PersonCertificationService certificationService;
    
    // ← POST - Crear
    @PostMapping
    @Operation(summary = "Add an item by person ID")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "201", description = "Certification created successfully"),
            @ApiResponse(responseCode = "400", description = "Invalid request data"),
            @ApiResponse(responseCode = "404", description = "Person or certification code not found"),
            @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<Long> createCertification(
            @PathVariable Long personId,
            @RequestParam(name = "create-type", defaultValue = "ONLY") TypeOfManagement type,
            @Valid @RequestBody CreatePersonCertificationRequest request) {
        
        log.info("POST /v2/people/{}/certifications - Request: {} - Type: {}", personId, request, type);
        Long certificationId = certificationService.create(personId, request, type);
        return ResponseEntity.status(HttpStatus.CREATED).body(certificationId);
    }
    
    // ← GET - Listar
    @GetMapping
    @Operation(summary = "Returns a list of items by person ID")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "List of valid certifications retrieved successfully"),
            @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<List<PersonCertification>> getValidCertifications(
            @PathVariable Long personId) {
        
        log.info("GET /v2/people/{}/certifications", personId);
        List<PersonCertification> certifications = certificationService.getValidCertifications(personId);
        return ResponseEntity.ok(certifications);
    }
    
    // ← GET - Por ID
    @GetMapping("/{idEntity}")
    @Operation(summary = "Returns an item by ID")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Certification retrieved successfully"),
            @ApiResponse(responseCode = "404", description = "Certification not found for the given person"),
            @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<PersonCertification> getCertificationById(
            @PathVariable Long personId,
            @PathVariable Long idEntity) {
        
        log.info("GET /v2/people/{}/certifications/{}", personId, idEntity);
        PersonCertification certification = certificationService.getById(personId, idEntity);
        return ResponseEntity.ok(certification);
    }
    
    // ← DELETE - Soft delete
    @DeleteMapping("/{idEntity}")
    @Operation(summary = "Logical deletion by ID")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Certification deleted successfully"),
            @ApiResponse(responseCode = "404", description = "Certification not found for the given person"),
            @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<Long> deleteCertification(
            @PathVariable Long personId,
            @PathVariable Long idEntity) {
        
        log.info("DELETE /v2/people/{}/certifications/{}", personId, idEntity);
        Long deletedId = certificationService.delete(personId, idEntity);
        return ResponseEntity.ok(deletedId);
    }
}
```

**Características clave:**
- ✅ `@RestController` + `@RequestMapping` con path base
- ✅ `@Tag` para agrupación en Swagger
- ✅ `@Validated` para habilitar validaciones
- ✅ `@Valid` en `@RequestBody` para validar DTOs
- ✅ `@ApiResponses` en todos los endpoints
- ✅ Logging de todas las operaciones
- ✅ Path variables: `{personId}` y `{idEntity}`
- ✅ Query param: `create-type` con valor por defecto
- ✅ Códigos HTTP apropiados: `201` para POST, `200` para GET/DELETE

---

## 🧪 Capa 7: Tests

### Patrón: Controller Tests (Integration)

```java
@WebMvcTest(controllers = CertificationsController.class)
@ActiveProfiles("test")
@DisplayName("CertificationsController - Integration Tests")
class CertificationsControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private ObjectMapper objectMapper;
    
    @MockBean
    private PersonCertificationService certificationService;
    
    @Test
    @DisplayName("POST /certifications - Should return 201 when request is valid")
    void createCertification_ShouldReturn201_WhenRequestIsValid() throws Exception {
        // Arrange
        CreatePersonCertificationRequest request = CreatePersonCertificationRequest.builder()
                .certificationCode("CERT_IVA")
                .url("http://example.com/cert.pdf")
                .aliquot(new BigDecimal("0.21"))
                .build();
        
        when(certificationService.create(anyLong(), any(), any()))
                .thenReturn(100L);
        
        // Act & Assert
        mockMvc.perform(post("/v2/people/{personId}/certifications", 1L)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isCreated())
                .andExpect(content().string("100"));
        
        verify(certificationService, times(1)).create(eq(1L), any(), eq(TypeOfManagement.ONLY));
    }
    
    @Test
    @DisplayName("POST /certifications - Should return 400 when validation fails")
    void createCertification_ShouldReturn400_WhenValidationFails() throws Exception {
        // Arrange - Request con aliquot fuera de rango
        CreatePersonCertificationRequest request = CreatePersonCertificationRequest.builder()
                .certificationCode("CERT_IVA")
                .url("http://example.com/cert.pdf")
                .aliquot(new BigDecimal("1.5"))  // ← Inválido
                .build();
        
        // Act & Assert
        mockMvc.perform(post("/v2/people/{personId}/certifications", 1L)
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
                .andExpect(status().isBadRequest());
        
        verify(certificationService, never()).create(anyLong(), any(), any());
    }
}
```

---

## 📝 Capa 8: Logging (Best Practices)

### Patrón: Logging en Controllers

**Propósito**: Registrar todas las operaciones HTTP para trazabilidad y debugging.

```java
@RestController
@Slf4j  // ← Lombok genera el logger automáticamente
public class CertificationsController {
    
    @PostMapping
    public ResponseEntity<Long> createCertification(
            @PathVariable Long personId,
            @RequestParam TypeOfManagement type,
            @Valid @RequestBody CreatePersonCertificationRequest request) {
        
        // ✅ CORRECTO: Log al inicio con método HTTP, ruta y parámetros relevantes
        log.info("POST /v2/people/{}/certifications - Request: {} - Type: {}", personId, request, type);
        
        Long certificationId = certificationService.create(personId, request, type);
        return ResponseEntity.status(HttpStatus.CREATED).body(certificationId);
    }
    
    @GetMapping
    public ResponseEntity<List<PersonCertification>> getValidCertifications(@PathVariable Long personId) {
        // ✅ CORRECTO: Log simple para operaciones de lectura
        log.info("GET /v2/people/{}/certifications", personId);
        
        List<PersonCertification> certifications = certificationService.getValidCertifications(personId);
        return ResponseEntity.ok(certifications);
    }
    
    @GetMapping("/{idEntity}")
    public ResponseEntity<PersonCertification> getCertificationById(
            @PathVariable Long personId,
            @PathVariable Long idEntity) {
        
        // ✅ CORRECTO: Incluye ambos IDs para trazabilidad
        log.info("GET /v2/people/{}/certifications/{}", personId, idEntity);
        
        PersonCertification certification = certificationService.getById(personId, idEntity);
        return ResponseEntity.ok(certification);
    }
    
    @DeleteMapping("/{idEntity}")
    public ResponseEntity<Long> deleteCertification(
            @PathVariable Long personId,
            @PathVariable Long idEntity) {
        
        // ✅ CORRECTO: Log de operaciones de eliminación
        log.info("DELETE /v2/people/{}/certifications/{}", personId, idEntity);
        
        Long deletedId = certificationService.delete(personId, idEntity);
        return ResponseEntity.ok(deletedId);
    }
}
```

**Características del logging en Controllers:**
- ✅ Incluye método HTTP (POST, GET, DELETE, etc.)
- ✅ Incluye ruta completa con path variables
- ✅ Usa placeholders `{}` en lugar de concatenación
- ✅ Incluye parámetros relevantes (IDs, request body, query params)
- ✅ Log al **inicio** de la operación

### Patrón: Logging en Services

**Propósito**: Registrar operaciones de negocio, resultados y errores.

```java
@Service
@Slf4j
public class PersonCertificationServiceImpl implements PersonCertificationService {
    
    @Override
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Long create(Long personId, CreatePersonCertificationRequest request, TypeOfManagement typeOfManagement) {
        // ✅ CORRECTO: Log al inicio con contexto de negocio
        log.info("Creating certification for person: {} with strategy: {}", personId, typeOfManagement);
        
        validateRequest(request);
        
        PersonEntity person = peopleCenterRepository.findById(personId)
                .orElseThrow(() -> new ResourceNotFoundException("Person not found with id: " + personId));
        
        CertificationEntity certification = certificationRepository.findByCode(request.getCertificationCode())
                .orElseThrow(() -> new ResourceNotFoundException(
                        "Certification not found with code: " + request.getCertificationCode()));
        
        PersonCertificationEntity entity = PersonCertificationEntity.builder()
                .personId(person)
                .url(request.getUrl())
                .certification(certification)
                .percentage(request.getPercentage())
                .aliquot(request.getAliquot())
                .startDate(request.getStartDate())
                .endDate(request.getEndDate())
                .build();
        
        Set<PersonCertificationEntity> existingCertifications = personCertificationRepository.findByPersonId(person);
        PersonCertificationEntity saved = personCertificationRepository.saveWithTypeOfManagement(entity,
                existingCertifications, typeOfManagement);
        
        // ✅ CORRECTO: Log al final con resultado exitoso
        log.info("Certification created with id: {}", saved.getId());
        return saved.getId();
    }
    
    @Override
    @Transactional(readOnly = true)
    public List<PersonCertification> getValidCertifications(Long personId) {
        // ✅ CORRECTO: Log de operaciones de lectura
        log.info("Getting valid certifications for person: {}", personId);
        
        ZonedDateTime now = ZonedDateTime.now();
        List<PersonCertificationEntity> entities = personCertificationRepository
                .findValidCertifications(personId, now);
        
        return entities.stream()
                .map(mapper::toDto)
                .collect(Collectors.toList());
    }
    
    @Override
    @Transactional(readOnly = true)
    public PersonCertification getById(Long personId, Long certificationId) {
        // ✅ CORRECTO: Incluye ambos IDs para trazabilidad
        log.info("Getting certification {} for person: {}", certificationId, personId);
        
        PersonCertificationEntity entity = personCertificationRepository
                .findByIdAndPersonId_Id(certificationId, personId)
                .orElseThrow(() -> new ResourceNotFoundException(
                        String.format("Certification not found with id %d for person %d", certificationId, personId)));
        
        return mapper.toDto(entity);
    }
    
    @Override
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Long delete(Long personId, Long certificationId) {
        // ✅ CORRECTO: Log al inicio
        log.info("Deleting certification {} for person: {}", certificationId, personId);
        
        PersonCertificationEntity entity = personCertificationRepository
                .findByIdAndPersonId_Id(certificationId, personId)
                .orElseThrow(() -> new ResourceNotFoundException(
                        String.format("Certification not found with id %d for person %d", certificationId, personId)));
        
        entity.setDeletedAt(ZonedDateTime.now());
        personCertificationRepository.save(entity);
        
        // ✅ CORRECTO: Log al final confirmando éxito
        log.info("Certification {} deleted successfully", certificationId);
        return entity.getId();
    }
    
    private void validateRequest(CreatePersonCertificationRequest request) {
        if (request.getPercentage() != null && request.getAliquot() != null) {
            // ⚠️ OPCIONAL: Log de validaciones fallidas
            log.warn("Validation failed: both percentage and aliquot specified for person certification");
            throw new IllegalArgumentException("Only one of 'percentage' or 'aliquot' can be specified");
        }
        
        if (request.getStartDate() != null && request.getEndDate() != null) {
            if (request.getStartDate().isAfter(request.getEndDate())) {
                log.warn("Validation failed: start date {} is after end date {}", 
                        request.getStartDate(), request.getEndDate());
                throw new IllegalArgumentException("Start date must be before end date");
            }
        }
    }
}
```

**Características del logging en Services:**
- ✅ Log al **inicio** de operaciones importantes
- ✅ Log al **final** de operaciones exitosas
- ✅ Incluye IDs relevantes para trazabilidad
- ✅ Incluye contexto de negocio (estrategias, estados, etc.)
- ✅ Usa placeholders `{}` para parámetros
- ⚠️ Opcional: Log de validaciones fallidas con `log.warn()`

### Logging de Errores (Opcional pero Recomendado)

```java
@Override
@Transactional(isolation = Isolation.SERIALIZABLE)
public Long create(Long personId, CreatePersonCertificationRequest request, TypeOfManagement typeOfManagement) {
    log.info("Creating certification for person: {} with strategy: {}", personId, typeOfManagement);
    
    try {
        validateRequest(request);
        
        PersonEntity person = peopleCenterRepository.findById(personId)
                .orElseThrow(() -> new ResourceNotFoundException("Person not found with id: " + personId));
        
        // ... resto de la lógica ...
        
        log.info("Certification created with id: {}", saved.getId());
        return saved.getId();
        
    } catch (ResourceNotFoundException e) {
        // ✅ CORRECTO: Log de errores de negocio
        log.error("Failed to create certification for person {}: {}", personId, e.getMessage());
        throw e;
    } catch (Exception e) {
        // ✅ CORRECTO: Log de errores inesperados con stack trace
        log.error("Unexpected error creating certification for person {}", personId, e);
        throw e;
    }
}
```

### ✅ Buenas Prácticas

#### 1. **Usar Placeholders, NO Concatenación**

```java
// ✅ CORRECTO - Usa placeholders {}
log.info("Creating certification for person: {}", personId);

// ❌ INCORRECTO - Concatenación de strings
log.info("Creating certification for person: " + personId);
```

**Por qué:** Los placeholders son más eficientes porque solo se evalúan si el nivel de log está habilitado.

#### 2. **Niveles de Log Apropiados**

| Nivel | Cuándo Usar | Ejemplo |
|-------|-------------|---------|
| `log.info()` | Operaciones normales, flujo de negocio | `log.info("Certification created with id: {}", id)` |
| `log.warn()` | Situaciones anormales pero recuperables | `log.warn("Validation failed: both percentage and aliquot specified")` |
| `log.error()` | Errores que requieren atención | `log.error("Failed to create certification: {}", e.getMessage())` |
| `log.debug()` | Información detallada para debugging | `log.debug("Validating request: {}", request)` |

#### 3. **Información Contextual Relevante**

```java
// ✅ CORRECTO - Incluye contexto completo
log.info("Creating certification for person: {} with strategy: {}", personId, typeOfManagement);

// ⚠️ INCOMPLETO - Falta contexto
log.info("Creating certification");
```

#### 4. **Log al Inicio y al Final de Operaciones Importantes**

```java
@Override
public Long create(Long personId, CreatePersonCertificationRequest request, TypeOfManagement typeOfManagement) {
    // ✅ Log al inicio
    log.info("Creating certification for person: {} with strategy: {}", personId, typeOfManagement);
    
    // ... lógica ...
    
    // ✅ Log al final
    log.info("Certification created with id: {}", saved.getId());
    return saved.getId();
}
```

#### 5. **No Loggear Información Sensible**

```java
// ❌ EVITAR - Puede contener información sensible
log.info("Creating certification with request: {}", request);

// ✅ MEJOR - Solo IDs y metadatos
log.info("Creating certification for person: {} with code: {}", personId, request.getCertificationCode());
```

### ❌ Malas Prácticas

```java
// ❌ NO HACER: Concatenación de strings
log.info("Person " + personId + " created certification " + certificationId);

// ❌ NO HACER: Logging excesivo en loops
for (PersonCertification cert : certifications) {
    log.info("Processing certification: {}", cert.getId());  // ← Evitar en loops
}

// ❌ NO HACER: Logging de objetos completos sin control
log.info("Request: {}", request);  // ← Puede ser muy grande o contener datos sensibles

// ❌ NO HACER: Logging sin contexto
log.info("Operation completed");  // ← ¿Qué operación? ¿Para quién?
```

### 📊 Tabla Resumen: Qué Loggear en Cada Capa

| Capa | Qué Loggear | Nivel | Ejemplo |
|------|-------------|-------|---------|
| **Controller** | Método HTTP + Ruta + Parámetros | `INFO` | `log.info("POST /v2/people/{}/certifications", personId)` |
| **Service (inicio)** | Operación + Contexto de negocio | `INFO` | `log.info("Creating certification for person: {}", personId)` |
| **Service (fin)** | Resultado exitoso | `INFO` | `log.info("Certification created with id: {}", id)` |
| **Service (error)** | Error + Contexto | `ERROR` | `log.error("Failed to create certification: {}", e.getMessage())` |
| **Validaciones** | Validación fallida | `WARN` | `log.warn("Validation failed: invalid date range")` |
| **Repository** | ❌ Generalmente NO | - | Spring Data JPA ya loggea queries |

### 🎯 Checklist de Logging

- [ ] Agregar `@Slf4j` en controllers y services
- [ ] Log al inicio de cada endpoint en controller
- [ ] Log al inicio de operaciones importantes en service
- [ ] Log al final de operaciones exitosas en service
- [ ] Usar placeholders `{}` en lugar de concatenación
- [ ] Incluir IDs relevantes para trazabilidad
- [ ] Incluir contexto de negocio (estrategias, estados, etc.)
- [ ] Usar niveles apropiados (INFO, WARN, ERROR)
- [ ] NO loggear información sensible
- [ ] NO loggear en loops intensivos

---

## 📋 Checklist para Nuevos Servicios


### ✅ Paso 1: Base de Datos
- [ ] Crear tabla de catálogo con columnas: `id`, `code`, `name`, `created_at`, `deleted_at`
- [ ] Crear tabla de relación con columnas: `id`, `person_id`, `[catalog]_id`, campos específicos, `created_at`, `deleted_at`
- [ ] Agregar índices en `person_id` y `[catalog]_id`
- [ ] Agregar constraint `UNIQUE` en `code` del catálogo

### ✅ Paso 2: Entidades
- [ ] Crear `[Catalog]Entity` con `@Where(clause = "deleted_at IS NULL")`
- [ ] Crear `Person[Catalog]Entity` implementando `HasId`, `HasDeleted`, `HasType`
- [ ] Agregar `@ManyToOne` a `PersonEntity` y `[Catalog]Entity`
- [ ] Implementar métodos `isDeleted()`, `setDeleted()`, `getType()`

### ✅ Paso 3: Repositorios
- [ ] Crear `[Catalog]Repository` con método `findByCode(String code)`
- [ ] Crear `Person[Catalog]Repository` extendiendo `JpaRepositoryWithTypeOfManagement`
- [ ] Agregar método `findByIdAndPersonId_Id(Long id, Long personId)`
- [ ] Agregar método `findByPersonId(PersonEntity personId)`
- [ ] Agregar `@Query` personalizada si hay lógica de negocio compleja

### ✅ Paso 4: DTOs
- [ ] Crear `[Catalog]` DTO de respuesta (sin validaciones)
- [ ] Crear `Person[Catalog]` DTO de respuesta (sin validaciones)
- [ ] Crear `CreatePerson[Catalog]Request` con validaciones completas:
  - [ ] `@NotBlank` en campos requeridos
  - [ ] `@Size` en strings (coincidiendo con BD)
  - [ ] `@DecimalMin/Max` en números
  - [ ] `@Schema` para documentación
- [ ] NO usar `@JsonFormat` en `ZonedDateTime`

### ✅ Paso 5: Mapper
- [ ] Crear `Person[Catalog]Mapper` con `@Mapper(componentModel = "spring")`
- [ ] Agregar método `toDto(Entity)` y `toEntity(DTO)`

### ✅ Paso 6: Service
- [ ] Crear interface `Person[Catalog]Service`
- [ ] Crear implementación `Person[Catalog]ServiceImpl`
- [ ] Agregar `@Transactional(isolation = SERIALIZABLE)` en CREATE y DELETE
- [ ] Agregar `@Transactional(readOnly = true)` en GET
- [ ] Implementar validaciones de negocio
- [ ] Usar `findByIdAndPersonId_Id` para validar pertenencia

### ✅ Paso 7: Controller
- [ ] Crear `[Catalog]sController` con path `/v2/people/{personId}/[catalogs]`
- [ ] Agregar `@Tag`, `@Validated`, `@Slf4j`
- [ ] Implementar endpoints: POST, GET (lista), GET (por ID), DELETE
- [ ] Agregar `@ApiResponses` en todos los endpoints
- [ ] Usar `@Valid` en `@RequestBody`
- [ ] Logging en todos los métodos

### ✅ Paso 8: Tests
- [ ] Crear tests de controller con `@WebMvcTest`
- [ ] Probar validaciones (casos válidos e inválidos)
- [ ] Probar todos los endpoints
- [ ] Verificar códigos HTTP correctos

---

## 🎯 Principios Clave

1. **Separación de Responsabilidades**: Cada capa tiene una responsabilidad clara
2. **Validación en Capas**: Request DTO (Bean Validation) + Service (lógica de negocio)
3. **Seguridad**: Siempre validar pertenencia con `findByIdAndPersonId_Id`
4. **Transacciones**: SERIALIZABLE para escritura, readOnly para lectura
5. **Soft Delete**: Usar `deletedAt` con `@Where` clause
6. **Auditoría**: `createdAt` con `@CreationTimestamp`
7. **Documentación**: `@ApiResponses` + `@Schema` en todos los endpoints y DTOs
8. **Logging**: Log en todos los métodos públicos de service y controller
9. **TypeOfManagement**: Usar para gestión inteligente de duplicados
10. **Consistencia**: Seguir siempre el mismo patrón

---

## 📚 Referencias Rápidas

### Anotaciones Importantes

| Anotación | Uso | Capa |
|-----------|-----|------|
| `@Where(clause = "deleted_at IS NULL")` | Filtro automático de soft delete | Entity |
| `@Transactional(isolation = SERIALIZABLE)` | Escritura con máxima consistencia | Service |
| `@Transactional(readOnly = true)` | Lectura optimizada | Service |
| `@Valid` | Activar validaciones Bean Validation | Controller |
| `@ApiResponses` | Documentar códigos HTTP | Controller |
| `@Schema` | Documentar campos en Swagger | DTO |

### Convenciones de Nombres

| Elemento | Patrón | Ejemplo |
|----------|--------|---------|
| Entidad de catálogo | `[Name]Entity` | `CertificationEntity` |
| Entidad de relación | `Person[Name]Entity` | `PersonCertificationEntity` |
| DTO de respuesta | `[Name]` | `Certification` |
| DTO de request | `CreatePerson[Name]Request` | `CreatePersonCertificationRequest` |
| Repository | `Person[Name]Repository` | `PersonCertificationRepository` |
| Service | `Person[Name]Service` | `PersonCertificationService` |
| Controller | `[Name]sController` | `CertificationsController` |
| Mapper | `Person[Name]Mapper` | `PersonCertificationMapper` |

---

**Última actualización**: 2025-12-02  
**Versión**: 1.0  
**Autor**: Equipo People Center
