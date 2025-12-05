# Propuesta: Entidad Espejo para Acceso a Registros Eliminados

## Ejemplo basado en el servicio PersonTaxActivity

## Problema

La entidad `PersonTaxActivityEntity` tiene la anotación `@Where(clause = "deleted_at IS NULL")` que filtra automáticamente todos los registros con soft delete. Esto impide acceder al historial completo de actividades económicas de una persona, incluyendo las eliminadas lógicamente.

## Solución Propuesta

Implementar un patrón de **entidad espejo** usando `@MappedSuperclass` para compartir la estructura común entre dos entidades:
- Una entidad **filtrada** (actual) para operaciones normales
- Una entidad **espejo sin filtro** para acceso al historial completo

---

## Arquitectura de la Solución

### 1. Clase Base Abstracta (`@MappedSuperclass`)

Crear una clase base que contenga todos los campos y relaciones comunes:

```java
@MappedSuperclass
@Getter
@Setter
public abstract class PersonTaxActivityBase implements HasId, HasDeleted, HasType, HasOrigin {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "person_id", nullable = false)
    private PersonEntity personId;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "type", nullable = false, length = 50)
    private TaxActivityType type;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "origin", nullable = false, length = 50)
    private TaxActivityOrigin origin;
    
    @Column(name = "adjusts_for_inflation", nullable = false)
    private Boolean adjustsForInflation;
    
    @Column(name = "is_independent", nullable = false)
    private Boolean isIndependent;
    
    @Column(name = "is_service", nullable = false)
    private Boolean isService;
    
    @Column(name = "is_principal", nullable = false)
    private Boolean isPrincipal;
    
    @ManyToOne
    @JoinColumn(name = "person_tax_activity_economic_id", nullable = false)
    private PersonTaxActivityEconomicEntity economicActivity;
    
    @ManyToOne
    @JoinColumn(name = "person_tax_activity_monotribute_id")
    private PersonTaxActivityMonotributeEntity monotribute;
    
    @CreationTimestamp
    @Column(name = "created_at", nullable = false, updatable = false)
    private ZonedDateTime createdAt;
    
    @Column(name = "deleted_at")
    private ZonedDateTime deletedAt;
    
    // Implementaciones de interfaces
    @Override
    public String getType() {
        return type.name();
    }
    
    @Override
    public String getOrigin() {
        return origin.name();
    }
    
    @Override
    public boolean isDeleted() {
        return deletedAt != null;
    }
    
    @Override
    public void setDeleted(boolean deleted) {
        this.deletedAt = deleted ? ZonedDateTime.now() : null;
    }
}
```

### 2. Entidad Filtrada (Operaciones Normales)

Mantener la entidad actual con el filtro `@Where`:

```java
@Entity
@Table(name = "person_tax_activity")
@Where(clause = "deleted_at IS NULL") // <--- Filtro presente
public class PersonTaxActivityEntity extends PersonTaxActivityBase {
    // Sin campos adicionales, hereda todo de la clase base
}
```

**Uso:**
- CRUD normal (create, update, delete)
- Consultas que solo necesitan registros activos
- Operaciones con TypeOfManagement

### 3. Entidad Espejo (Historial Completo)

Crear una nueva entidad **sin** el filtro `@Where`:

```java
@Entity
@Table(name = "person_tax_activity")
@Immutable  // Opcional: marca como read-only para evitar modificaciones accidentales
            // Filtro @where ausente
public class PersonTaxActivityHistoryEntity extends PersonTaxActivityBase {
    // Sin campos adicionales, hereda todo de la clase base
}
```

**Uso:**
- Consultas de historial completo
- Reportes que incluyen registros eliminados
- Auditoría

---

## Implementación de Repositorios

### Repositorio para Entidad Filtrada (LA que usamnos actualmene)

```java
@Repository
public interface PersonTaxActivityRepository 
    extends JpaRepositoryWithTypeAndOriginManagement<PersonTaxActivityEntity, Long> {
    
    Set<PersonTaxActivityEntity> findByPersonId(PersonEntity person);
    
    Optional<PersonTaxActivityEntity> findByPersonId_IdAndDeletedAtIsNull(Long personId);
    
    Optional<PersonTaxActivityEntity> findByIdAndPersonId_Id(Long id, Long personId);
    
    List<PersonTaxActivityEntity> findByPersonId_IdOrderByCreatedAtDesc(Long personId);
}
```

### Repositorio para Entidad Espejo (Historial o entidad sin filtros)

```java
@Repository
public interface PersonTaxActivityHistoryRepository 
    extends JpaRepository<PersonTaxActivityHistoryEntity, Long> {
    
    /**
     * Obtiene TODAS las actividades económicas de una persona,
     * incluyendo las eliminadas lógicamente.
     */
    List<PersonTaxActivityHistoryEntity> findByPersonId_IdOrderByCreatedAtDesc(Long personId);
    
    /**
     * Obtiene todas las actividades eliminadas de una persona.
     */
    List<PersonTaxActivityHistoryEntity> findByPersonId_IdAndDeletedAtIsNotNullOrderByCreatedAtDesc(Long personId);
    
    /**
     * Cuenta total de actividades (activas + eliminadas).
     */
    long countByPersonId_Id(Long personId);
}
```

---

## Modificaciones en el Servicio

### Agregar Método para Historial Completo

```java
public interface PersonTaxActivityService {
    
    // Métodos existentes...
    Long create(Long personId, CreatePersonTaxActivityRequest request, TypeOfManagement typeOfManagement);
    PersonTaxActivity getCurrentTaxActivity(Long personId);
    PersonTaxActivity getTaxActivityById(Long personId, Long taxActivityId);
    List<PersonTaxActivity> getTaxActivityHistory(Long personId);
    void delete(Long personId, Long taxActivityId);
    
    // NUEVO: Historial completo incluyendo eliminados
    /**
     * Obtiene el historial completo de actividades económicas,
     * incluyendo las eliminadas lógicamente.
     */
    List<PersonTaxActivity> getFullTaxActivityHistory(Long personId);
}
```

### Implementación en el Service

```java
@Service
@RequiredArgsConstructor
public class PersonTaxActivityServiceImpl implements PersonTaxActivityService {
    
    private final PersonTaxActivityRepository taxActivityRepository;
    private final PersonTaxActivityHistoryRepository taxActivityHistoryRepository; // NUEVO
    private final PersonTaxActivityMapper mapper;
    
    // Métodos existentes sin cambios...
    
    @Override
    @Transactional(readOnly = true)
    public List<PersonTaxActivity> getFullTaxActivityHistory(Long personId) {
        log.info("Getting FULL tax activity history (including deleted) for person: {}", personId);
        
        return taxActivityHistoryRepository
            .findByPersonId_IdOrderByCreatedAtDesc(personId)
            .stream()
            .map(mapper::toDto)  // Reutiliza el mapper existente
            .collect(Collectors.toList());
    }
}
```

---

## Modificaciones en el Controller

### Agregar Endpoint para Historial Completo

```java
@RestController
@RequestMapping("/api/v1/people/{personId}/tax-activities")
@RequiredArgsConstructor
public class TaxActivitiesController {
    
    private final PersonTaxActivityService service;
    
    // Endpoints existentes...
    
    /**
     * Obtiene el historial COMPLETO de actividades económicas,
     * incluyendo las eliminadas lógicamente.
     */
    @GetMapping("/history/full")
    @Operation(summary = "Get full tax activity history including deleted records")
    @ApiResponse(responseCode = "200", description = "Full history retrieved successfully")
    public ResponseEntity<List<PersonTaxActivity>> getFullHistory(
            @PathVariable Long personId) {
        
        List<PersonTaxActivity> history = service.getFullTaxActivityHistory(personId);
        return ResponseEntity.ok(history);
    }
}
```

---

## Ventajas de esta Solución

1. **Separación de Responsabilidades**
   - Entidad filtrada para operaciones normales
   - Entidad espejo para consultas de auditoría/historial

2. **Sin Duplicación de Código**
   - `@MappedSuperclass` centraliza toda la lógica común
   - Ambas entidades comparten la misma estructura

3. **Seguridad**
   - `@Immutable` en la entidad espejo previene modificaciones accidentales
   - Repositorio espejo solo tiene métodos de lectura

4. **Compatibilidad**
   - No rompe código existente
   - Entidad original mantiene su comportamiento

5. **Flexibilidad**
   - Fácil agregar más consultas específicas en el repositorio espejo
   - Permite filtros personalizados (ej: solo eliminados, rango de fechas, etc.)

---

## Consideraciones

### Performance
- Ambas entidades mapean a la misma tabla física
- No hay overhead de almacenamiento
- Hibernate cachea ambas entidades independientemente

### Mapper
- El `PersonTaxActivityMapper` existente funciona con ambas entidades
- Ambas heredan de la misma clase base
- No requiere cambios en el mapper

### Testing
- Tests existentes no se ven afectados
- Nuevos tests para verificar que la entidad espejo devuelve registros eliminados

## Resumen

Esta propuesta permite acceder al historial completo de actividades económicas (incluyendo eliminadas) mediante:
- Una clase base `@MappedSuperclass` que centraliza la estructura
- Una entidad filtrada para operaciones normales (actual)
- Una entidad espejo sin filtro para historial completo
- Repositorio y servicio dedicados para consultas de historial
- Nuevo endpoint REST para exponer la funcionalidad

La implementación es **no invasiva**, **type-safe** y **mantiene la compatibilidad** con el código existente.
