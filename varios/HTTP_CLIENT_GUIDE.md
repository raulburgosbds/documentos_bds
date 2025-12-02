# Cliente HTTP de People Center - Guía Completa

## 📚 ¿Qué es el Cliente HTTP?

El **Cliente HTTP** (`people-center-client`) es una librería Java que permite a **otros microservicios** consumir los endpoints de People Center de forma programática, sin necesidad de hacer llamadas HTTP manuales.

---

## 🎯 ¿Cuándo se crea un Cliente?

### ✅ **SE CREA cuando:**

1. **El recurso es parte del modelo core de una persona**
   - Ejemplos: Address, Contact, Document, CoreData, etc.
   - Son entidades que **otros microservicios necesitan consultar/modificar frecuentemente**

2. **Hay integración entre microservicios**
   - Cuando otros servicios del banco necesitan acceder a datos de personas
   - Para evitar duplicación de lógica HTTP en cada consumidor

3. **Se quiere estandarizar el acceso**
   - Proporciona una interfaz Java tipada
   - Maneja errores de forma consistente
   - Encapsula la lógica de comunicación HTTP

### ❌ **NO SE CREA cuando:**

1. **El recurso es muy específico o poco usado**
   - Ejemplo: `PersonCertification` (nuevo, específico para certificaciones fiscales)
   - No todos los microservicios necesitan acceder a certificaciones

2. **Es un endpoint interno del microservicio**
   - Solo se usa dentro de People Center
   - No hay necesidad de consumo externo

3. **El recurso está en desarrollo/experimental**
   - Mejor esperar a que se estabilice antes de crear el cliente

---

## 🏗️ Arquitectura del Cliente

### Estructura de Módulos

```
library/
├── model/                          # DTOs compartidos
│   └── PersonCertification.java   # ✅ Ya existe
├── client/                         # Cliente HTTP
│   ├── PeopleCenterClient<T>      # Interface genérica
│   ├── AbstractPeopleCenterClient # Implementación base
│   └── [Específicos por modelo]   # ❌ NO existe para Certification
└── pom.xml
```

### Clientes Existentes

Según el README, existen clientes para:

- ✅ AcceptedTerms
- ✅ Address
- ✅ Channel
- ✅ Contact
- ✅ CoreData
- ✅ Document
- ✅ EconomicActivity
- ✅ IncomeInformation
- ✅ LegalInformation
- ✅ PersonalInfo
- ✅ Relationship
- ✅ RiskEvaluation

**❌ NO existe para:** PersonCertification (todavía)

---

## 🔄 ¿Cómo Funciona?

### 1. **Sin Cliente (Acceso Directo por HTTP)**

```java
// En otro microservicio
RestTemplate restTemplate = new RestTemplate();
String url = "https://bs-int.bdsdigital.com.ar/v2/people/123/certifications";

// Hacer llamada HTTP manual
ResponseEntity<List<PersonCertification>> response = 
    restTemplate.exchange(url, HttpMethod.GET, ...);
```

**Problemas:**
- ❌ Código repetitivo
- ❌ Manejo de errores manual
- ❌ URLs hardcodeadas
- ❌ Sin tipado fuerte

### 2. **Con Cliente (Acceso Programático)**

```java
// En otro microservicio
@Autowired
private PeopleCenterClient<Address> addressClient;

// Uso simple y tipado
List<Address> addresses = addressClient.get(personId);
Long newAddressId = addressClient.create(personId, TypeOfManagement.ONLY, newAddress);
```

**Ventajas:**
- ✅ Código limpio y simple
- ✅ Manejo de errores centralizado
- ✅ Configuración por variables de entorno
- ✅ Tipado fuerte (compile-time safety)
- ✅ Reutilizable en múltiples servicios

---

## 📦 Uso del Cliente

### Dependencia en pom.xml

```xml
<!-- Solo modelo (DTOs) -->
<dependency>
    <groupId>ar.com.bds.lib</groupId>
    <artifactId>people-center-model</artifactId>
    <version>0.10.5-SNAPSHOT</version>
</dependency>

<!-- Modelo + Cliente HTTP -->
<dependency>
    <groupId>ar.com.bds.lib</groupId>
    <artifactId>people-center-client</artifactId>
    <version>0.10.5-SNAPSHOT</version>
</dependency>
```

### Configuración

```properties
# Obligatorio
people-center.server.host=${PEOPLE_CENTER_SERVER_HOST}

# Opcional
people-center.server.timeout.seconds=${PEOPLE_CENTER_SERVER_TIMEOUT_SECONDS}
```

### Ejemplo de Uso

```java
@Service
public class MyService {
    
    @Autowired
    private PeopleCenterClient<Address> addressClient;
    
    @Autowired
    private PeopleCenterClient<Contact> contactClient;
    
    public void updatePersonData(Long personId) {
        // Obtener direcciones
        List<Address> addresses = addressClient.get(personId);
        
        // Crear nueva dirección
        Address newAddress = new Address();
        newAddress.setStreet("Calle Falsa 123");
        Long addressId = addressClient.create(
            personId, 
            TypeOfManagement.ONLY, 
            newAddress
        );
        
        // Eliminar dirección antigua
        addressClient.logicalDeletion(personId, oldAddressId);
    }
}
```

---

## 🔍 Caso: PersonCertification

### Estado Actual

- ✅ **Modelo existe:** `PersonCertification.java` en `library/model`
- ✅ **Endpoint existe:** `CertificationsController` en microservice
- ❌ **Cliente NO existe:** No hay `PeopleCenterClient<PersonCertification>`

### ¿Por qué NO tiene cliente?

1. **Es una funcionalidad nueva** (migración V26)
2. **Uso específico:** Solo para certificaciones fiscales (IVA, Ganancias)
3. **No es core:** No todos los microservicios necesitan acceder a certificaciones
4. **Puede agregarse después:** Si otros servicios lo necesitan

### ¿Cuándo crear el cliente?

**Crear cuando:**
- Otro microservicio necesite consultar certificaciones
- Se requiera integración con sistemas externos
- Se identifique uso frecuente desde múltiples servicios

**Por ahora:**
- Los consumidores pueden usar el endpoint REST directamente
- O crear su propio cliente HTTP si lo necesitan

---

## 🆚 Comparación: Con vs Sin Cliente

| Aspecto | Sin Cliente | Con Cliente |
|---------|-------------|-------------|
| **Complejidad** | Alta (HTTP manual) | Baja (método Java) |
| **Reutilización** | Baja | Alta |
| **Mantenimiento** | Difícil | Fácil |
| **Tipado** | Débil (JSON) | Fuerte (Java) |
| **Errores** | Manual | Centralizado |
| **Testing** | Complejo | Simple (mock) |
| **Dependencias** | RestTemplate | Library |

---

## 🛠️ Cómo Crear un Cliente (Si se necesita)

### 1. Crear el Cliente Específico

```java
// En library/client/src/main/java/ar/com/bds/lib/client/

@Component
public class PersonCertificationClient 
    extends AbstractPeopleCenterClient<PersonCertification> {
    
    @Override
    protected Class<PersonCertification> getClassOfParam() {
        return PersonCertification.class;
    }
}
```

### 2. Registrar el Path

```java
// En config/HttpClientProperties.java
@Bean
public Map<Class<?>, String> mapOfPath() {
    Map<Class<?>, String> map = new HashMap<>();
    // ... otros paths
    map.put(PersonCertification.class, "/v2/people/{idPerson}/certifications");
    return map;
}
```

### 3. Publicar Nueva Versión

```bash
# Actualizar version en pom.xml
mvn clean install
mvn deploy
```

### 4. Usar en Otros Servicios

```java
@Autowired
private PeopleCenterClient<PersonCertification> certificationClient;

List<PersonCertification> certs = certificationClient.get(personId);
```

---

## ✅ Recomendaciones

### Para PersonCertification

**Por ahora:** ❌ NO crear cliente
- Es funcionalidad nueva
- Uso limitado
- Puede agregarse después si se necesita

**Crear cliente cuando:**
- ✅ Otro microservicio lo solicite
- ✅ Se identifique uso frecuente
- ✅ Se requiera integración externa

### Acceso Actual

Los consumidores pueden:

1. **Usar el endpoint REST directamente**
   ```bash
   GET https://bs-int.bdsdigital.com.ar/v2/people/{id}/certifications
   ```

2. **Usar solo el modelo**
   ```xml
   <dependency>
       <groupId>ar.com.bds.lib</groupId>
       <artifactId>people-center-model</artifactId>
   </dependency>
   ```

3. **Crear su propio cliente HTTP** (si lo necesitan)

---

## 📝 Resumen

| Pregunta | Respuesta |
|----------|-----------|
| **¿Qué es el cliente?** | Librería Java para consumir People Center desde otros microservicios |
| **¿Cuándo se crea?** | Para recursos core que otros servicios consumen frecuentemente |
| **¿PersonCertification tiene cliente?** | ❌ NO (por ahora) |
| **¿Puedo acceder sin cliente?** | ✅ SÍ, usando el endpoint REST directamente |
| **¿Debo crear el cliente ahora?** | ❌ NO, esperar a que se necesite |

---

**Última actualización:** 2025-12-01  
**Autor:** Antigravity AI Assistant
