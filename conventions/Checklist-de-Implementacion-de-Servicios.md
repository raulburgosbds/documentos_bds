# Checklist de Implementación de Servicios

Este documento contiene las mejores prácticas identificadas durante la implementación del servicio `PersonTaxActivity`. Úsalo como guía para implementar nuevos servicios o refactorizar existentes.

---

## 📋 Índice

1. [Estructura de Capas](#estructura-de-capas)
2. [Manejo de Excepciones](#manejo-de-excepciones)
3. [Validaciones](#validaciones)
4. [Repositorio y Queries](#repositorio-y-queries)
5. [Mappers](#mappers)
6. [Transacciones](#transacciones)
7. [Logging](#logging)
8. [Documentación API](#documentación-api)
9. [Patrones de Diseño](#patrones-de-diseño)

---

## 1. Estructura de Capas

### ✅ Checklist

- [ ] **Controller**: Solo orquestación, sin lógica de negocio
- [ ] **Service**: Toda la lógica de negocio y validaciones
- [ ] **Repository**: Solo queries a la base de datos
- [ ] **Validator**: Validaciones de reglas de negocio complejas
- [ ] **Mapper**: Conversión entre DTOs y Entities

### 📝 Ejemplo Correcto

```java
// ✅ Controller - Solo orquesta
@PostMapping
public ResponseEntity<Long> create(@PathVariable Long personId, 
                                   @Valid @RequestBody CreateRequest request) {
    Long id = service.create(personId, request);
    return ResponseEntity.status(HttpStatus.CREATED).body(id);
}

// ✅ Service - Lógica de negocio
public Long create(Long personId, CreateRequest request) {
    validator.validate(request);  // Validaciones
    PersonEntity person = findPerson(personId);  // Buscar entidades
    Entity entity = mapper.toEntity(request);  // Mapeo
    return repository.save(entity).getId();  // Persistencia
}
```

---

## 2. Manejo de Excepciones

### ✅ Checklist

- [ ] **Service lanza excepciones**, NO retorna `null`
- [ ] **Controller NO maneja null checks**
- [ ] **Usar excepciones específicas del proyecto**:
  - `NotFoundException` → 404
  - `IllegalArgumentException` → 400 (auto)
  - `ConflictWithExistingPersonException` → 409
- [ ] **Mensajes descriptivos** con contexto (IDs, valores)
- [ ] **NO usar `try-catch` genéricos** en el service

### 📝 Ejemplo Correcto

```java
// ✅ Service lanza excepción
public PersonTaxActivity getCurrentTaxActivity(Long personId) {
    return repository.findByPersonId_IdAndDeletedAtIsNull(personId)
            .map(mapper::toDto)
            .orElseThrow(() -> new NotFoundException(
                String.format("Current tax activity not found for person ID: %s", personId)));
}

// ✅ Controller confía en el service
public ResponseEntity<PersonTaxActivity> getCurrent(@PathVariable Long personId) {
    PersonTaxActivity result = service.getCurrentTaxActivity(personId);
    return ResponseEntity.ok(result);  // No hay if (result == null)
}
```

### ❌ Ejemplo Incorrecto

```java
// ❌ Service retorna null
public PersonTaxActivity getCurrentTaxActivity(Long personId) {
    return repository.findByPersonId_IdAndDeletedAtIsNull(personId)
            .map(mapper::toDto)
            .orElse(null);  // ❌ NO HACER ESTO
}

// ❌ Controller maneja null
public ResponseEntity<PersonTaxActivity> getCurrent(@PathVariable Long personId) {
    PersonTaxActivity result = service.getCurrentTaxActivity(personId);
    if (result == null) {  // ❌ NO HACER ESTO
        return ResponseEntity.notFound().build();
    }
    return ResponseEntity.ok(result);
}
```

---

## 3. Validaciones

### ✅ Checklist

- [ ] **Crear clase Validator** para reglas de negocio complejas
- [ ] **Usar `@Valid`** en controllers para validaciones básicas (DTOs)
- [ ] **Validar en el Service** antes de persistir
- [ ] **Mensajes de error claros** y específicos
- [ ] **Usar `IllegalArgumentException`** para errores de validación

### 📝 Ejemplo Correcto

```java
// ✅ Validator dedicado
@Component
@Slf4j
public class TaxActivityValidator {
    
    public void validateMonotributeCoherence(CreatePersonTaxActivityRequest request) {
        if (request.getMonotribute() != null) {
            // Categoría obligatoria
            if (request.getMonotribute().getCategory() == null || 
                request.getMonotribute().getCategory().isBlank()) {
                log.warn("Validation failed: monotribute without category");
                throw new IllegalArgumentException(
                    "If monotribute is specified, category is mandatory");
            }
            
            // Null-safe comparison
            if (Boolean.TRUE.equals(request.getAdjustsForInflation())) {
                log.warn("Validation failed: monotributista with inflation adjustment");
                throw new IllegalArgumentException(
                    "A Monotributista taxpayer cannot adjust for inflation");
            }
        }
    }
}

// ✅ Service usa el validator
public Long create(Long personId, CreateRequest request) {
    validator.validateMonotributeCoherence(request);  // Primero validar
    // ... resto de la lógica
}
```

### 💡 Tip: Comparaciones Null-Safe

```java
// ✅ Null-safe
if (Boolean.TRUE.equals(request.getAdjustsForInflation())) { ... }

// ❌ Puede lanzar NullPointerException
if (request.getAdjustsForInflation() == true) { ... }
if (request.getAdjustsForInflation().equals(true)) { ... }
```

---

## 4. Repositorio y Queries

### ✅ Checklist

- [ ] **Extender `JpaRepository`** (no custom interfaces si no es necesario)
- [ ] **Queries nativas** cuando necesites bypass `@Where` clause
- [ ] **Nombres descriptivos** para métodos de query
- [ ] **Documentar queries complejas** con JavaDoc
- [ ] **Usar `@Param`** en queries nativas para claridad

### 📝 Ejemplo Correcto

```java
@Repository
public interface PersonTaxActivityRepository extends JpaRepository<PersonTaxActivityEntity, Long> {

    /**
     * Find a tax activity by ID and person ID (only active ones due to @Where clause)
     */
    Optional<PersonTaxActivityEntity> findByIdAndPersonId_Id(Long id, Long personId);

    /**
     * Find a tax activity by ID and person ID including deleted ones.
     * Uses native query to bypass @Where clause.
     */
    @Query(value = "SELECT * FROM person_tax_activity WHERE id = :id AND person_id = :personId", 
           nativeQuery = true)
    Optional<PersonTaxActivityEntity> findByIdAndPersonIdIncludingDeleted(
            @Param("id") Long id, 
            @Param("personId") Long personId);

    /**
     * Find the current (active) tax activity for a person
     */
    Optional<PersonTaxActivityEntity> findByPersonId_IdAndDeletedAtIsNull(Long personId);

    /**
     * Find all active tax activities ordered by creation date
     */
    List<PersonTaxActivityEntity> findByPersonId_IdOrderByCreatedAtDesc(Long personId);
}
```

### 🔒 Seguridad en Queries

```java
// ✅ SIEMPRE filtrar por person_id para seguridad
@Query(value = "SELECT * FROM person_tax_activity WHERE id = :id AND person_id = :personId", 
       nativeQuery = true)
Optional<PersonTaxActivityEntity> findByIdAndPersonIdIncludingDeleted(
        @Param("id") Long id, 
        @Param("personId") Long personId);  // ← Previene acceso a datos de otras personas
```

---

## 5. Mappers

### ✅ Checklist

- [ ] **Usar MapStruct** para conversiones DTO ↔ Entity
- [ ] **`@AfterMapping`** para lógica post-mapeo (ej: bidirectional links)
- [ ] **Ignorar campos autogenerados** (`id`, `createdAt`, etc.)
- [ ] **Métodos helper** para mapeos complejos

### 📝 Ejemplo Correcto

```java
@Mapper(componentModel = "spring")
public interface PersonTaxActivityMapper {

    @Mapping(target = "id", ignore = true)
    @Mapping(target = "personId", ignore = true)
    @Mapping(target = "createdAt", ignore = true)
    @Mapping(target = "deletedAt", ignore = true)
    PersonTaxActivityEntity toEntity(CreatePersonTaxActivityRequest request);

    /**
     * Establece enlaces bidireccionales para 1-to-1 Shared PK
     */
    @AfterMapping
    default void linkBidirectional(@MappingTarget PersonTaxActivityEntity entity) {
        if (entity.getEconomicActivity() != null) {
            entity.getEconomicActivity().setActivity(entity);
        }
        if (entity.getMonotribute() != null) {
            entity.getMonotribute().setActivity(entity);
        }
    }

    PersonTaxActivity toDto(PersonTaxActivityEntity entity);
}
```

---

## 6. Transacciones

### ✅ Checklist

- [ ] **`@Transactional`** en métodos que modifican datos
- [ ] **`@Transactional(readOnly = true)`** en métodos de consulta
- [ ] **NO usar `@Transactional`** en controllers
- [ ] **Dejar que excepciones propaguen** para rollback automático

### 📝 Ejemplo Correcto

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class PersonTaxActivityServiceImpl implements PersonTaxActivityService {

    @Override
    @Transactional  // ← Escribe en DB
    public Long create(Long personId, CreateRequest request) {
        // ... lógica que modifica datos
        return repository.save(entity).getId();
    }

    @Override
    @Transactional(readOnly = true)  // ← Solo lectura
    public PersonTaxActivity getCurrentTaxActivity(Long personId) {
        return repository.findByPersonId_IdAndDeletedAtIsNull(personId)
                .map(mapper::toDto)
                .orElseThrow(() -> new NotFoundException(...));
    }
}
```

---

## 7. Logging

### ✅ Checklist

- [ ] **Log al inicio** de operaciones importantes
- [ ] **Log al finalizar** operaciones exitosas
- [ ] **Log de errores** con contexto (IDs, valores)
- [ ] **Niveles apropiados**:
  - `info`: Operaciones normales
  - `warn`: Validaciones fallidas
  - `error`: Errores inesperados
- [ ] **NO loguear datos sensibles**

### 📝 Ejemplo Correcto

```java
@Override
@Transactional
public Long create(Long personId, CreateRequest request) {
    log.info("Creating tax activity for person: {}", personId);
    
    // ... lógica
    
    if (deletedCount > 0) {
        log.info("Soft-deleted {} previous tax activit{} for person: {}",
                deletedCount, deletedCount == 1 ? "y" : "ies", personId);
    }
    
    log.info("Tax activity created with ID: {}", savedEntity.getId());
    return savedEntity.getId();
}

@Override
@Transactional(readOnly = true)
public PersonTaxActivity getTaxActivityById(Long personId, Long taxActivityId) {
    log.info("Getting tax activity {} for person: {} (including deleted)", 
             taxActivityId, personId);
    
    return repository.findByIdAndPersonIdIncludingDeleted(taxActivityId, personId)
            .map(mapper::toDto)
            .orElseThrow(() -> new NotFoundException(
                String.format("Tax activity ID: %s not found for person ID: %s", 
                              taxActivityId, personId)));
}
```

---

## 8. Documentación API

### ✅ Checklist

- [ ] **`@Operation`** con summary descriptivo
- [ ] **`@ApiResponses`** con todos los códigos posibles
- [ ] **Descripciones claras** de parámetros
- [ ] **Tags** para agrupar endpoints

### 📝 Ejemplo Correcto

```java
@RestController
@RequestMapping("/v2/people/{personId}/tax-activities")
@Tag(name = "People Center - Tax Activities")
@RequiredArgsConstructor
@EnableControllerLogging
public class TaxActivitiesController {

    @PostMapping
    @Operation(summary = "Create a new tax activity for a person")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "201", description = "Tax activity created successfully"),
            @ApiResponse(responseCode = "400", description = "Invalid request data or business rule violation"),
            @ApiResponse(responseCode = "404", description = "Person not found"),
            @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<Long> createTaxActivity(
            @PathVariable Long personId,
            @Valid @RequestBody CreatePersonTaxActivityRequest request) {
        
        Long taxActivityId = taxActivityService.create(personId, request);
        return ResponseEntity.status(HttpStatus.CREATED).body(taxActivityId);
    }

    @GetMapping("/{taxActivityId}")
    @Operation(summary = "Get a specific tax activity by ID (including deleted)")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Tax activity retrieved successfully"),
            @ApiResponse(responseCode = "404", description = "Person or tax activity not found"),
            @ApiResponse(responseCode = "500", description = "Internal server error")
    })
    public ResponseEntity<PersonTaxActivity> getTaxActivityById(
            @PathVariable Long personId,
            @PathVariable Long taxActivityId) {
        
        PersonTaxActivity taxActivity = taxActivityService.getTaxActivityById(personId, taxActivityId);
        return ResponseEntity.ok(taxActivity);
    }
}
```

---

## 9. Patrones de Diseño

### ✅ Soft Delete Pattern

```java
// ✅ Implementación correcta
@Override
@Transactional
public void delete(Long personId, Long taxActivityId) {
    log.info("Deleting tax activity {} for person: {}", taxActivityId, personId);
    
    PersonTaxActivityEntity entity = repository.findByIdAndPersonId_Id(taxActivityId, personId)
            .orElseThrow(() -> new NotFoundException(
                String.format("Tax activity not found with ID: %d for person: %d",
                              taxActivityId, personId)));
    
    entity.setDeletedAt(ZonedDateTime.now());
    repository.save(entity);
    
    log.info("Tax activity {} soft deleted", taxActivityId);
}
```

### ✅ Timestamp Consistency Pattern

```java
// ✅ Usar el mismo timestamp para múltiples registros
ZonedDateTime deletionTimestamp = ZonedDateTime.now();

for (PersonTaxActivityEntity existing : existingActivities) {
    if (!existing.isDeleted()) {
        existing.setDeletedAt(deletionTimestamp);  // Mismo timestamp para todos
    }
}
```

### ✅ 1-to-1 Shared Primary Key Pattern

```java
// Parent Entity
@OneToOne(mappedBy = "activity", cascade = CascadeType.ALL)
@PrimaryKeyJoinColumn
private PersonTaxActivityEconomicEntity economicActivity;

// Child Entity
@Id
private Long id;

@MapsId
@OneToOne
@JoinColumn(name = "id")
private PersonTaxActivityEntity activity;

// Service - Establecer enlaces bidireccionales
if (entity.getEconomicActivity() != null) {
    entity.getEconomicActivity().setActivity(entity);
}
```

---

## 📊 Resumen de Mejores Prácticas

| Aspecto | ✅ Hacer | ❌ No Hacer |
|---------|---------|-------------|
| **Excepciones** | Lanzar desde service | Retornar null |
| **Validaciones** | Usar Validator dedicado | Validar en controller |
| **Queries** | Documentar y usar @Param | Queries sin documentación |
| **Transacciones** | @Transactional en service | @Transactional en controller |
| **Logging** | Info al inicio/fin | Log sin contexto |
| **Controller** | Solo orquestación | Lógica de negocio |
| **Null checks** | En service con excepciones | En controller |
| **Mappers** | MapStruct con @AfterMapping | Mapeo manual |

---

## 🔍 Checklist Final Pre-Merge

Antes de hacer merge, verifica:

- [ ] ✅ Todos los métodos públicos tienen JavaDoc
- [ ] ✅ Excepciones lanzan mensajes descriptivos
- [ ] ✅ Validaciones están en Validator, no en Service
- [ ] ✅ Controller no tiene lógica de negocio
- [ ] ✅ Queries nativas tienen @Param
- [ ] ✅ Métodos transaccionales marcados correctamente
- [ ] ✅ Logs informativos en operaciones clave
- [ ] ✅ API documentada con Swagger
- [ ] ✅ Tests compilando (mvn clean compile -DskipTests)
- [ ] ✅ Código sigue convenciones del proyecto

---

## 📚 Referencias

- `PersonTaxActivityServiceImpl` - Implementación de referencia
- `TaxActivityValidator` - Ejemplo de validador
- `PersonTaxActivityRepository` - Queries nativas y documentación
- `ControllerExceptionHandler` - Manejo centralizado de excepciones

---

**Última actualización:** 2024-12-12  
**Basado en:** Refactorización de PersonTaxActivity (1-to-1 Shared PK)