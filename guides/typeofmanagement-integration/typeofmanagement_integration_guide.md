# Ejemplo de Integración: TypeOfManagement en PersonCertification

## Introducción

Este documento explica paso a paso cómo se integró la estrategia **TypeOfManagement** en el servicio de PersonCertification para manejar duplicados de manera estandarizada.

---

## Paso 1: Preparar la Entidad PersonCertificationEntity

### 1.0 Verificar que las entidades esten anotadas con ***@where***

La entidad debe estar anotada con @where para filtrar automáticamente los registros eliminados lógicamente

Agrega en cada una de tus entidades relacionadas, EJEMPLO:

```java
@Where(clause = "deleted_at IS NULL") //<---- ANOTA CADA ENTITY CLASS ASI
public class CertificationEntity { 
    // ....
}

@Where(clause = "deleted_at IS NULL") //<---- ANOTA CADA ENTITY CLASS ASI
public class PersonCertificationEntity {
    // ....
}
```

### 1.1 Implementar Interfaces Requeridas

La entidad debe implementar tres interfaces para que funcione con TypeOfManagement:

```java
public class PersonCertificationEntity implements HasId, HasDeleted, HasType {
    // ...
}
```

**Archivo**: [PersonCertificationEntity.java](file:///c:/repos/bds/people-center/microservice/src/main/java/ar/com/bds/people/center/entity/PersonCertificationEntity.java)

### 1.2 Agregar Imports Necesarios

```java
import ar.com.bds.lib.peoplecenter.model.interfaces.HasDeleted;
import ar.com.bds.lib.peoplecenter.model.interfaces.HasId;
import ar.com.bds.lib.peoplecenter.model.interfaces.HasType;
```

### 1.3 Implementar Métodos de las Interfaces

#### Interface `HasId`
Ya está implementado por defecto con el campo `id`:
```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;
```

#### Interface `HasDeleted`
Implementar métodos para soft delete:
```java
@Override
public boolean isDeleted() {
    return deletedAt != null;
}

@Override
public void setDeleted(boolean deleted) {
    this.deletedAt = deleted ? ZonedDateTime.now() : null;
}
```

#### Interface `HasType`
Implementar método que retorna el tipo de certificación:
```java
@Override
public String getType() {
    return certification.getCode();
}
```

> [!IMPORTANT]
> El método `getType()` retorna `certification.getCode()` porque este código identifica de manera única el tipo de certificación (ej: "CERT_IVA", "CERT_GANANCIAS").

---

## Paso 2: Actualizar el Repositorio

### 2.1 Extender JpaRepositoryWithTypeOfManagement

Cambiar la interfaz base del repositorio:

**Antes**:
```java
public interface PersonCertificationRepository extends JpaRepository<PersonCertificationEntity, Long> {
    // ...
}
```

**Después**:
```java
public interface PersonCertificationRepository extends 
        JpaRepositoryWithTypeOfManagement<PersonCertificationEntity, Long> {
    // ...
}
```

**Archivo**: [PersonCertificationRepository.java](file:///c:/repos/bds/people-center/microservice/src/main/java/ar/com/bds/people/center/repository/PersonCertificationRepository.java)

### 2.2 Agregar Método findByPersonId

Este método es necesario para obtener las certificaciones existentes de una persona:

```java
Set<PersonCertificationEntity> findByPersonId(PersonEntity personId);
```

## Paso 3: Actualizar la Interfaz del Servicio

### 3.1 Agregar Parámetro TypeOfManagement al Método create()

**Antes**:
```java
Long create(Long personId, CreatePersonCertificationRequest request);
```

**Después**:
```java
Long create(Long personId, CreatePersonCertificationRequest request, TypeOfManagement typeOfManagement);
```

**Archivo**: [PersonCertificationService.java](file:///c:/repos/bds/people-center/microservice/src/main/java/ar/com/bds/people/center/service/PersonCertificationService.java)

### 3.2 Agregar Import

```java
import ar.com.bds.lib.peoplecenter.model.enums.TypeOfManagement;
```

## Paso 4: Implementar la Lógica en el Servicio

### 4.1 Actualizar el Método create()

**Archivo**: [PersonCertificationServiceImpl.java](file:///c:/repos/bds/people-center/microservice/src/main/java/ar/com/bds/people/center/service/impl/PersonCertificationServiceImpl.java)

### 4.2 Obtener Certificaciones Existentes

Antes de guardar, obtener las certificaciones existentes de la persona:

```java
Set<PersonCertificationEntity> existingCertifications = 
    personCertificationRepository.findByPersonId(person);
```

### 4.3 Usar saveWithTypeOfManagement()

Reemplazar el `save()` tradicional con el método que maneja duplicados:

**Antes**:
```java
PersonCertificationEntity saved = personCertificationRepository.save(entity);
```

**Después**:
```java
PersonCertificationEntity saved = personCertificationRepository.saveWithTypeOfManagement(
    entity, 
    existingCertifications, 
    typeOfManagement
);
```

### 4.4 Agregar Imports

```java
import ar.com.bds.lib.peoplecenter.model.enums.TypeOfManagement;
import java.util.Set;
```

---

## Paso 5: Actualizar el Controller

### 5.1 Agregar Parámetro TypeOfManagement al Endpoint POST

**Archivo**: [CertificationsController.java](file:///c:/repos/bds/people-center/microservice/src/main/java/ar/com/bds/people/center/controller/CertificationsController.java)

```java
@PostMapping
@Operation(summary = "Create a new certification for a person")
public ResponseEntity<Long> createCertification(
        @PathVariable Long personId,
        @RequestParam(name = "create-type", defaultValue = "ONLY") TypeOfManagement type,
        @Valid @RequestBody CreatePersonCertificationRequest request) {
    
    log.info("POST /v2/people/{}/certifications - Request: {} - Type: {}", personId, request, type);
    Long certificationId = certificationService.create(personId, request, type);
    return ResponseEntity.status(HttpStatus.CREATED).body(certificationId);
}
```

> [!TIP]
> El parámetro `create-type` tiene valor por defecto `ONLY`, lo que significa que si no se especifica, se comportará como antes (solo crear sin verificar duplicados).

### 5.2 Agregar Import

```java
import ar.com.bds.lib.peoplecenter.model.enums.TypeOfManagement;
```

---

## Paso 6: Actualizar Tests

### 6.1 Actualizar PersonCertificationServiceTest

**Archivo**: [PersonCertificationServiceTest.java](file:///c:/repos/bds/people-center/microservice/src/test/java/ar/com/bds/people/center/service/PersonCertificationServiceTest.java)

Agregar el parámetro `TypeOfManagement.ONLY` en todas las llamadas al método `create()`:

```java

Long id = service.create(personId, request, TypeOfManagement.ONLY);
```

### 6.2 Actualizar CertificationsControllerTest

**Archivo**: [CertificationsControllerTest.java](file:///c:/repos/bds/people-center/microservice/src/test/java/ar/com/bds/people/center/controller/CertificationsControllerTest.java)

#### 6.2.1 Actualizar Mocks

```java

when(certificationService.create(eq(personId), any(CreatePersonCertificationRequest.class), any(TypeOfManagement.class)))
    .thenReturn(1L);
```

#### 6.2.2 Actualizar Assertions de Tipos

```java

verify(certificationService).create(eq(personId), any(CreatePersonCertificationRequest.class), any(TypeOfManagement.class));
```


## Resumen de Cambios por Archivo

| Archivo | Cambios Principales |
|---------|---------------------|
| **PersonCertificationEntity** | Implementar `HasId`, `HasDeleted`, `HasType` |
| **PersonCertificationRepository** | Extender `JpaRepositoryWithTypeOfManagement`; agregar `findByPersonId()` |
| **PersonCertificationService** | Agregar parámetro `TypeOfManagement` a `create()` |
| **PersonCertificationServiceImpl** | Usar `saveWithTypeOfManagement()`; obtener certificaciones existentes |
| **CertificationsController** | Agregar `@RequestParam` para TypeOfManagement; actualizar tipos |
| **Tests** | Agregar `TypeOfManagement.ONLY` en llamadas; actualizar mocks y assertions |

---

## Cómo Funciona TypeOfManagement

### Estrategias Disponibles

1. **ONLY**: Solo guarda, no toca nada
2. **REMOVING_SAME_TYPE**: Borra del mismo tipo
3. **REMOVING_REST**: Borra todo lo demas 

### Ejemplo de Uso en API

```bash
# Crear certificación sin verificar duplicados (comportamiento por defecto)
POST /v2/people/123/certifications

# Crear certificación solo si no existe
POST /v2/people/123/certifications?create-type=ONLY

# Reemplazar certificación existente del mismo tipo
POST /v2/people/123/certifications?create-type=REMOVING_SAME_TYPE

# Permitir múltiples certificaciones del mismo tipo
POST /v2/people/123/certifications?create-type=REMOVING_REST
```

### Flujo de Ejecución

```mermaid
graph TD
    A[Controller recibe request] --> B[Extrae TypeOfManagement del query param]
    B --> C[Service obtiene certificaciones existentes]
    C --> D[Repository.findByPersonId]
    D --> E[Service llama saveWithTypeOfManagement]
    E --> F{TypeOfManagement?}
    F -->|ONLY| G[Permite duplicados, Solo guarda, no toca nada]
    F -->|REMOVING_SAME_TYPE| H[Borra del mismo tipo]
    F -->|REMOVING_REST| I[Borra todo lo demás]
    G --> J[Guarda nueva certificación]
    H --> J
    I --> J
    J --> K[Retorna ID de certificación]
```

## Beneficios de la Integración

✅ **Estandarización**: Mismo patrón usado en todo el proyecto  
✅ **Flexibilidad**: El cliente decide la estrategia de duplicados  
✅ **Consistencia**: Tipos de datos alineados entre DB y código  
✅ **Mantenibilidad**: Lógica centralizada en `JpaRepositoryWithTypeOfManagement`  
✅ **Retrocompatibilidad**: Valor por defecto `ONLY` mantiene comportamiento esperado
