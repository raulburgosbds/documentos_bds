# Guía de Anotaciones @Transactional por Método HTTP

## Tabla de Referencia Rápida

| Método HTTP | Operación | Anotación Recomendada | Nivel de Aislamiento | Justificación |
|-------------|-----------|----------------------|---------------------|---------------|
| **GET** | Lectura simple | `@Transactional(readOnly = true)` | `READ_COMMITTED` (default) | Optimización de rendimiento, previene escrituras accidentales |
| **GET** | Lectura con joins complejos | `@Transactional(readOnly = true, isolation = REPEATABLE_READ)` | `REPEATABLE_READ` | Garantiza consistencia en múltiples consultas |
| **POST** | Creación | `@Transactional(isolation = SERIALIZABLE)` | `SERIALIZABLE` | Previene duplicados y garantiza unicidad |
| **POST** | Creación simple | `@Transactional` | `READ_COMMITTED` (default) | Suficiente para operaciones sin validaciones complejas |
| **PUT** | Actualización completa | `@Transactional(isolation = SERIALIZABLE)` | `SERIALIZABLE` | Previene actualizaciones concurrentes conflictivas |
| **PATCH** | Actualización parcial | `@Transactional(isolation = REPEATABLE_READ)` | `REPEATABLE_READ` | Balance entre consistencia y rendimiento |
| **DELETE** | Borrado lógico | `@Transactional(isolation = SERIALIZABLE)` | `SERIALIZABLE` | Previene borrados duplicados y garantiza consistencia |
| **DELETE** | Borrado físico | `@Transactional(isolation = SERIALIZABLE)` | `SERIALIZABLE` | Operación crítica que requiere máxima consistencia |

## Ejemplos Detallados para una Clase `Client`

### 1. GET - Lectura Simple

```java
@Service
public class ClientService {
    
    /**
     * Obtener un cliente por ID
     * - readOnly = true: Optimiza la consulta y previene escrituras
     * - Nivel de aislamiento: READ_COMMITTED (default)
     */
    @Transactional(readOnly = true)
    public Client getById(Long id) {
        return clientRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    }
    
    /**
     * Listar todos los clientes activos
     * - readOnly = true: Optimización para consultas de solo lectura
     */
    @Transactional(readOnly = true)
    public List<Client> getAllActive() {
        return clientRepository.findByDeletedAtIsNull();
    }
}
```

### 2. GET - Lectura con Joins Complejos

```java
/**
 * Obtener cliente con todas sus relaciones (pedidos, direcciones, etc.)
 * - readOnly = true: Optimización
 * - REPEATABLE_READ: Garantiza que si se hacen múltiples consultas,
 *   los datos no cambien entre ellas
 */
@Transactional(readOnly = true, isolation = Isolation.REPEATABLE_READ)
public ClientDetailDTO getClientWithDetails(Long id) {
    Client client = clientRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    
    // Múltiples consultas que deben ser consistentes
    List<Order> orders = orderRepository.findByClientId(id);
    List<Address> addresses = addressRepository.findByClientId(id);
    
    return ClientDetailDTO.builder()
        .client(client)
        .orders(orders)
        .addresses(addresses)
        .build();
}
```

### 3. POST - Creación Simple

```java
/**
 * Crear un nuevo cliente (sin validaciones complejas)
 * - Sin readOnly: Permite escritura
 * - Nivel default (READ_COMMITTED): Suficiente para operaciones simples
 */
@Transactional
public Long createSimple(CreateClientRequest request) {
    Client client = Client.builder()
        .name(request.getName())
        .email(request.getEmail())
        .build();
    
    return clientRepository.save(client).getId();
}
```

### 4. POST - Creación con Validaciones

```java
/**
 * Crear un nuevo cliente con validación de unicidad
 * - SERIALIZABLE: Previene que dos transacciones concurrentes creen
 *   clientes duplicados con el mismo email
 */
@Transactional(isolation = Isolation.SERIALIZABLE)
public Long create(CreateClientRequest request) {
    // Validar que no exista un cliente con el mismo email
    if (clientRepository.existsByEmail(request.getEmail())) {
        throw new ConflictException("Client with email already exists");
    }
    
    Client client = Client.builder()
        .name(request.getName())
        .email(request.getEmail())
        .phone(request.getPhone())
        .build();
    
    return clientRepository.save(client).getId();
}
```

### 5. PUT - Actualización Completa

```java
/**
 * Actualizar todos los campos de un cliente
 * - SERIALIZABLE: Previene actualizaciones concurrentes que puedan
 *   sobrescribir cambios de otras transacciones
 */
@Transactional(isolation = Isolation.SERIALIZABLE)
public Client update(Long id, UpdateClientRequest request) {
    Client client = clientRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    
    // Actualización completa de todos los campos
    client.setName(request.getName());
    client.setEmail(request.getEmail());
    client.setPhone(request.getPhone());
    client.setAddress(request.getAddress());
    client.setStatus(request.getStatus());
    
    return clientRepository.save(client);
}
```

### 6. PATCH - Actualización Parcial

```java
/**
 * Actualizar solo algunos campos del cliente
 * - REPEATABLE_READ: Balance entre consistencia y rendimiento
 * - Suficiente para actualizaciones parciales sin conflictos críticos
 */
@Transactional(isolation = Isolation.REPEATABLE_READ)
public Client partialUpdate(Long id, PatchClientRequest request) {
    Client client = clientRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    
    // Actualización solo de campos presentes
    if (request.getName() != null) {
        client.setName(request.getName());
    }
    if (request.getPhone() != null) {
        client.setPhone(request.getPhone());
    }
    
    return clientRepository.save(client);
}
```

### 7. DELETE - Borrado Lógico (Soft Delete)

```java
/**
 * Borrado lógico: Marca el registro como eliminado sin borrarlo físicamente
 * - SERIALIZABLE: Previene que se borre el mismo registro múltiples veces
 *   o que se borre mientras otra transacción lo está actualizando
 */
@Transactional(isolation = Isolation.SERIALIZABLE)
public Long softDelete(Long id) {
    Client client = clientRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    
    // Verificar que no esté ya eliminado
    if (client.getDeletedAt() != null) {
        throw new IllegalStateException("Client already deleted");
    }
    
    client.setDeletedAt(ZonedDateTime.now());
    clientRepository.save(client);
    
    return client.getId();
}
```

### 8. DELETE - Borrado Físico (Hard Delete)

```java
/**
 * Borrado físico: Elimina el registro permanentemente de la base de datos
 * - SERIALIZABLE: Máxima consistencia para operación crítica e irreversible
 * - Previene condiciones de carrera donde otra transacción intente
 *   acceder al registro mientras se está borrando
 */
@Transactional(isolation = Isolation.SERIALIZABLE)
public void hardDelete(Long id) {
    Client client = clientRepository.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    
    // Validaciones antes del borrado físico
    if (hasActiveOrders(client.getId())) {
        throw new IllegalStateException("Cannot delete client with active orders");
    }
    
    // Borrado en cascada de relaciones si es necesario
    addressRepository.deleteByClientId(id);
    contactRepository.deleteByClientId(id);
    
    // Borrado físico del cliente
    clientRepository.delete(client);
}

private boolean hasActiveOrders(Long clientId) {
    return orderRepository.existsByClientIdAndStatusNot(clientId, OrderStatus.COMPLETED);
}
```

## Niveles de Aislamiento Explicados

### READ_UNCOMMITTED (No recomendado)
- ⚠️ **Menos restrictivo**: Permite lecturas sucias
- ❌ **No usar**: Puede leer datos no confirmados de otras transacciones

### READ_COMMITTED (Default en Spring)
- ✅ **Recomendado para**: Operaciones simples de lectura/escritura
- ✅ **Previene**: Lecturas sucias
- ❌ **No previene**: Lecturas no repetibles, lecturas fantasma

### REPEATABLE_READ
- ✅ **Recomendado para**: Operaciones con múltiples consultas que deben ser consistentes
- ✅ **Previene**: Lecturas sucias, lecturas no repetibles
- ❌ **No previene**: Lecturas fantasma (en algunos motores de BD)

### SERIALIZABLE
- ✅ **Recomendado para**: Operaciones críticas que requieren máxima consistencia
- ✅ **Previene**: Lecturas sucias, lecturas no repetibles, lecturas fantasma
- ⚠️ **Costo**: Mayor impacto en rendimiento, puede causar más deadlocks

## Configuración Adicional de @Transactional

### Propagación (Propagation)

```java
// REQUIRED (default): Usa transacción existente o crea una nueva
@Transactional(propagation = Propagation.REQUIRED)

// REQUIRES_NEW: Siempre crea una nueva transacción, suspende la existente
@Transactional(propagation = Propagation.REQUIRES_NEW)

// NESTED: Crea una transacción anidada (savepoint)
@Transactional(propagation = Propagation.NESTED)

// MANDATORY: Requiere que exista una transacción, sino lanza excepción
@Transactional(propagation = Propagation.MANDATORY)
```

### Timeout

```java
// Timeout de 30 segundos
@Transactional(timeout = 30)
```

### Rollback

```java
// Rollback solo para excepciones específicas
@Transactional(rollbackFor = {CustomException.class, AnotherException.class})

// No hacer rollback para ciertas excepciones
@Transactional(noRollbackFor = {IgnoredException.class})
```

## Mejores Prácticas

### ✅ DO (Hacer)

1. **Usar `readOnly = true` para consultas**: Mejora rendimiento
2. **Usar SERIALIZABLE para operaciones críticas**: Creación, borrado, validaciones de unicidad
3. **Mantener transacciones cortas**: Evitar operaciones largas dentro de `@Transactional`
4. **Aplicar en la capa de servicio**: No en controladores ni repositorios
5. **Validar antes de modificar**: Reducir probabilidad de rollback

### ❌ DON'T (No hacer)

1. **No usar `@Transactional` en controladores**: Pertenece a la capa de servicio
2. **No hacer llamadas HTTP dentro de transacciones**: Pueden ser lentas y bloquear recursos
3. **No usar SERIALIZABLE para todo**: Impacta rendimiento innecesariamente
4. **No olvidar `readOnly = true` en lecturas**: Desperdicias optimizaciones
5. **No mezclar lógica de negocio con acceso a datos sin transacción**: Puede causar inconsistencias

## Ejemplo Completo de Servicio

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ClientService {
    
    private final ClientRepository clientRepository;
    private final OrderRepository orderRepository;
    
    // GET - Lectura simple
    @Transactional(readOnly = true)
    public Client getById(Long id) {
        return clientRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Client not found"));
    }
    
    // GET - Lectura con joins
    @Transactional(readOnly = true, isolation = Isolation.REPEATABLE_READ)
    public List<Client> getAllWithOrders() {
        return clientRepository.findAllWithOrders();
    }
    
    // POST - Creación con validación
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Long create(CreateClientRequest request) {
        if (clientRepository.existsByEmail(request.getEmail())) {
            throw new ConflictException("Email already exists");
        }
        
        Client client = mapper.toEntity(request);
        return clientRepository.save(client).getId();
    }
    
    // PUT - Actualización completa
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Client update(Long id, UpdateClientRequest request) {
        Client client = getById(id);
        mapper.updateEntity(client, request);
        return clientRepository.save(client);
    }
    
    // PATCH - Actualización parcial
    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public Client partialUpdate(Long id, PatchClientRequest request) {
        Client client = getById(id);
        mapper.partialUpdate(client, request);
        return clientRepository.save(client);
    }
    
    // DELETE - Borrado lógico
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public Long softDelete(Long id) {
        Client client = getById(id);
        client.setDeletedAt(ZonedDateTime.now());
        clientRepository.save(client);
        return client.getId();
    }
    
    // DELETE - Borrado físico
    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void hardDelete(Long id) {
        Client client = getById(id);
        
        if (orderRepository.existsByClientId(id)) {
            throw new IllegalStateException("Cannot delete client with orders");
        }
        
        clientRepository.delete(client);
    }
}
```
