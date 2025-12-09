# Guía de Implementación: Migración a Patrón Strategy con soporte Legacy
>
> **Objetivo**: Migrar de la herencia de interfaces (`JpaRepositoryWithTypeOfManagement`) al patrón Strategy, manteniendo la compatibilidad con el endpoint `?create-type=...`

Esta guía detalla el proceso paso a paso para refactorizar la gestión de duplicados en **Adviters People Center**.

---

## 📋 Prerrequisitos

Antes de comenzar, asegúrate de que las entidades de dominio (como `PersonCertificationEntity`, `ContactEntity`) implementen interfaces comunes para que las estrategias puedan manipularlas genéricamente.

Si no existen, créalas en `library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/interfaces`:

- `HasId`: `getId()`
- `HasType`: `getType()`
- `HasDeleted`: `setDeleted(Boolean)`, `isDeleted()`

---

## FASE 1: Infraestructura del Patrón (Core)

*Objetivo: Crear las herramientas sin tocar el negocio actual.*

### 1.1. La Interfaz de Estrategia

Crea el paquete `ar.com.bds.lib.peoplecenter.model.strategy` y define el contrato.

```java
public interface ManagementStrategy<T extends HasId & HasType & HasDeleted> {
    /**
     * Aplica la lógica de gestión de duplicados.
     * @param newEntity La nueva entidad a guardar.
     * @param existingEntities Colección de entidades existentes (ej. person.getContacts()).
     * @param saver Función para persistir (ej. repository::save).
     */
    void apply(T newEntity, Set<T> existingEntities, Consumer<T> saver);
}
```

### 1.2. Las Implementaciones (Beans)

Crea las estrategias como componentes de Spring (`@Component`).

**A. NewStrategy (Equivalente a ONLY)**

```java
@Component
public class NewStrategy<T extends HasId & HasType & HasDeleted> implements ManagementStrategy<T> {
    @Override
    public void apply(T newEntity, Set<T> existingEntities, Consumer<T> saver) {
        saver.accept(newEntity); // Solo guarda, no toca las existentes
    }
}
```

**B. NewAndReplaceStrategy (Equivalente a REMOVING_SAME_TYPE)**

```java
@Component
public class NewAndReplaceStrategy<T extends HasId & HasType & HasDeleted> implements ManagementStrategy<T> {
    @Override
    public void apply(T newEntity, Set<T> existingEntities, Consumer<T> saver) {
        existingEntities.stream()
            .filter(e -> Objects.equals(e.getType(), newEntity.getType()) // Mismo tipo
                      && !Objects.equals(e.getId(), newEntity.getId()))   // Distinto ID
            .forEach(e -> {
                e.setDeleted(true); // Borrado lógico
                saver.accept(e);    // Actualiza el anterior
            });
        saver.accept(newEntity);    // Guarda el nuevo
    }
}
```

---

## FASE 2: El Adaptador (Factory) para Retrocompatibilidad

*Objetivo: Resolver el problema de `create-type`. El cliente manda un Enum, pero el Servicio necesita una Strategy.*

Crea `ManagementStrategyFactory`. Esta clase es el puente.

```java
@Component
public class ManagementStrategyFactory {

    private final Map<TypeOfManagement, ManagementStrategy> strategyMap;

    public ManagementStrategyFactory(
            NewStrategy newStrategy,
            NewAndReplaceStrategy newAndReplaceStrategy) {
        
        // Mapeamos el Enum histórico a las clases nuevas
        this.strategyMap = Map.of(
            TypeOfManagement.ONLY, newStrategy,
            TypeOfManagement.REMOVING_SAME_TYPE, newAndReplaceStrategy,
            // Fallbacks para otros casos:
            TypeOfManagement.REMOVING_REST, newStrategy 
        );
    }

    public <T extends HasId & HasType & HasDeleted> ManagementStrategy<T> resolve(TypeOfManagement type) {
        if (type == null) return strategyMap.get(TypeOfManagement.ONLY);
        return strategyMap.getOrDefault(type, strategyMap.get(TypeOfManagement.ONLY));
    }
}
```

---

## FASE 3: Migración Vertical (Por Entidad)

*Proceso repetitivo. Aplica estos pasos a cada entidad (Certifications, Contacts, etc.) una por una.*

### 3.1. Actualizar el Controller

El Controller **traduce** el Enum a una Estrategia.

**Modificación en Controller:**

```java
@Autowired
private ManagementStrategyFactory strategyFactory; // 1. Inyectar Factory

@PostMapping(...)
public ResponseEntity<Long> create(
    @RequestParam(name = "create-type", defaultValue = "ONLY") TypeOfManagement type, 
    @RequestBody RequestDto request) {
    
    // 2. Resolver la estrategia
    ManagementStrategy<Entity> strategy = strategyFactory.resolve(type);

    // 3. Pasar la estrategia al servicio
    Long id = service.create(personId, request, strategy); 
    
    return ResponseEntity.ok(id);
}
```

### 3.2. Actualizar el Servicio

El servicio deja de depender del Enum.

**Modificación en Service:**

```java
// Cambiar firma: de TypeOfManagement -> ManagementStrategy
public Long create(Long personId, Dto request, ManagementStrategy<Entity> strategy) {
    var person = personRepository.findById(personId)...;
    var entity = mapper.toEntity(request);
    
    // 4. USAR LA ESTRATEGIA
    // Pasamos: (nueva, existentes, referencia al save del repo)
    strategy.apply(entity, person.getCertifications(), repository::save);
    
    return entity.getId();
}
```

### 3.3. Refactorizar el Repositorio (Objetivo Final)

El repositorio se limpia de herencia innecesaria.

**Modificación en Repository:**

```java
// ANTES
// public interface XRepository extends JpaRepositoryWithTypeOfManagement<XEntity, Long> ...

// DESPUÉS (¡Limpieza!)
public interface XRepository extends JpaRepository<XEntity, Long>, JpaRepositoryPeople<XEntity, Long> ...
```

> Si el código compila después de este paso, significa que la dependencia ha sido eliminada exitosamente.

---

## FASE 4: Flujo de Ejecución Resultante

1. **Cliente HTTP** → `POST ...?create-type=REMOVING_SAME_TYPE`
2. **Controller** → Recibe Enum. Llama a `Factory.resolve(Enum)`.
3. **Controller** → Obtiene instancia de `NewAndReplaceStrategy`.
4. **Service** → Recibe `ManagementStrategy` (abstracto).
5. **Service** → Ejecuta `strategy.apply(..., repository::save)`.
6. **Strategy** → Ejecuta lógica de filtrado y borrado lógico.
7. **Repository** → Solo ejecuta `save()` (JPA nativo).
