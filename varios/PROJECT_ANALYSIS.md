# People Center - Análisis Profundo del Proyecto

## 📋 Información General

### Descripción
Microservicio encargado de centralizar y gestionar la información de las personas (BUP - Base Única de Personas).

### Stack Tecnológico
- **Lenguaje:** Java 11
- **Framework:** Spring Boot 2.3.0.RELEASE
- **Parent Artifact:** bds-parent-springboot 1.10.16
- **Base de Datos:** MySQL
- **Cache:** Redis
- **Migraciones:** Flyway 6.5.7
- **Build Tool:** Maven
- **Testing:** JUnit 5 + Mockito

---

## 🏗️ Arquitectura del Proyecto

### Estructura de Módulos

```
people-center/
├── library/                    # Librería compartida
│   ├── model/                 # Modelos y DTOs
│   ├── client/                # Cliente REST
│   └── pom.xml
├── microservice/              # Aplicación principal
│   ├── src/main/java/
│   │   └── ar/com/bds/people/center/
│   │       ├── config/        # Configuraciones (Cache, DB, etc)
│   │       ├── connector/     # Conectores externos (Delorean/D94)
│   │       ├── controller/    # REST Controllers
│   │       ├── entity/        # Entidades JPA
│   │       ├── exception/     # Excepciones personalizadas
│   │       ├── mapper/        # MapStruct mappers
│   │       ├── repository/    # Repositorios JPA
│   │       └── service/       # Lógica de negocio
│   ├── src/main/resources/
│   │   └── db/migration/      # Scripts Flyway
│   └── src/test/
└── data/                      # Datos de prueba
```

---

## 🎯 Patrones y Convenciones

### 1. **Patrón de Capas**

```
Controller → Service → Repository → Database
     ↓          ↓
   Mapper    Entity
```

### 2. **Naming Conventions**

#### Controllers
- **Ubicación:** `ar.com.bds.people.center.controller`
- **Patrón:** `{Entity}Controller.java`
- **Ejemplo:** `CertificationsController.java`
- **Anotaciones:**
  ```java
  @RestController
  @RequestMapping("/v2/people/{personId}/{resource}")
  @Tag(name = "People Center - {Resource}")
  @RequiredArgsConstructor
  @Slf4j
  @Validated
  ```

#### Services
- **Interface:** `ar.com.bds.people.center.service.{Entity}Service.java`
- **Implementación:** `ar.com.bds.people.center.service.impl.{Entity}ServiceImpl.java`
- **Patrón de herencia:** Muchos servicios extienden `AbstractServiceImpl`
- **Anotaciones:**
  ```java
  @Service(SERVICE_NAME)
  @Slf4j
  ```

#### Repositories
- **Ubicación:** `ar.com.bds.people.center.repository`
- **Patrón:** `{Entity}Repository.java`
- **Herencia:** `extends JpaRepository<Entity, ID>`

#### Entities
- **Ubicación:** `ar.com.bds.people.center.entity`
- **Patrón:** `{Entity}Entity.java`
- **Tabla:** `@Table(name = "{table_name}")`
- **Convenciones:**
  - Soft delete: campo `deleted_at`
  - Timestamps: `created_at`, `updated_at`
  - IDs: Auto-increment para entidades secundarias, Long para personas

#### Mappers
- **Ubicación:** `ar.com.bds.people.center.mapper`
- **Patrón:** `{Entity}Mapper.java`
- **Framework:** MapStruct
- **Anotación:** `@Mapper(componentModel = "spring")`

### 3. **Convenciones de Código**

#### Mensajes de Excepción
- **Idioma:** INGLÉS (consistente en todo el proyecto)
- **Ejemplos:**
  ```java
  "Person not found with id: " + personId
  "Certification not found with code: " + code
  "{Entity} not found"
  ```

#### Logging
- **Nivel INFO:** Operaciones principales (CRUD)
  ```java
  log.info("Creating certification for person: {}", personId);
  log.info("Certification created with id: {}", saved.getId());
  ```
- **Nivel DEBUG:** Detalles de operación
- **Nivel ERROR:** Excepciones y errores

#### Validaciones
- **Request:** `@Valid` en parámetros de controller
- **Service:** Validaciones de negocio en métodos privados
- **Ejemplo:**
  ```java
  private void validateRequest(CreatePersonCertificationRequest request) {
      if (request.getPercentage() != null && request.getAliquot() != null) {
          throw new IllegalArgumentException("Only one of 'percentage' or 'aliquot' can be specified");
      }
  }
  ```

### 4. **Patrón de Servicios CRUD**

Muchos servicios siguen el patrón `AbstractServiceImpl` que proporciona:

```java
public abstract class AbstractServiceImpl<DTO, ENT, ENTITY_ID> {
    // Métodos implementados:
    - createEntity(Long idPerson, TypeOfManagement typeOfManagement, DTO dto)
    - get(Long idPerson, ENTITY_ID idEntity)
    - getCollection(Long idPerson)
    - logicalDeletion(Long idPerson, ENTITY_ID idEntity)
    
    // Métodos abstractos a implementar:
    - findById(Long idPerson, ENTITY_ID idEntity)
    - getFromPerson(Person personDto)
    - getAll(PersonEntity person)
    - getServiceName()
    - saveEntity(ENT entity)
    - saveWithTypeOfManagement(ENT ent, Set<ENT> originalCollection, TypeOfManagement typeOfManagement)
    - toEntity(DTO dto)
    - toDto(ENT entity)
}
```

---

## 🗄️ Modelo de Datos

### Entidades Principales

#### 1. **PersonEntity**
- Tabla: `people`
- ID: `BIGINT` (generado por secuencia)
- Relaciones: OneToMany con todas las entidades secundarias

#### 2. **CertificationEntity**
- Tabla: `certification`
- Campos:
  - `id`: INT AUTO_INCREMENT
  - `code`: VARCHAR(50) UNIQUE
  - `name`: VARCHAR(100)
  - `created_at`: DATETIME
  - `deleted_at`: DATETIME (soft delete)

#### 3. **PersonCertificationEntity**
- Tabla: `person_certification`
- Campos:
  - `id`: INT AUTO_INCREMENT
  - `person_id`: BIGINT (FK → people)
  - `certification_id`: INT (FK → certification)
  - `url`: VARCHAR(250)
  - `percentage`: DECIMAL(5,4) (nullable)
  - `aliquot`: DECIMAL(5,4) (nullable)
  - `start_date`: DATETIME (nullable)
  - `end_date`: DATETIME (nullable)
  - `created_at`: DATETIME
  - `deleted_at`: DATETIME (soft delete)

### Convenciones de Base de Datos

1. **Soft Delete:** Todas las entidades tienen `deleted_at`
2. **Timestamps:** `created_at` obligatorio, `updated_at` opcional
3. **Foreign Keys:** Nombradas como `{tabla}_{columna}_fk`
4. **Índices:** Nombrados como `{columna}_idx`
5. **Unique Constraints:** Nombrados como `{columna}_unique`

---

## 📦 Dependencias Clave

### Core Dependencies
```xml
<parent>
    <groupId>ar.com.bds.assets</groupId>
    <artifactId>bds-parent-springboot</artifactId>
    <version>1.10.16</version>
</parent>

<!-- Spring Boot -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Database -->
<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <version>8.0.32</version>
</dependency>

<!-- Flyway -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
    <version>6.5.7</version>
</dependency>

<!-- Redis -->
<dependency>
    <groupId>org.springframework.data</groupId>
    <artifactId>spring-data-redis</artifactId>
    <version>2.3.0.RELEASE</version>
</dependency>

<!-- People Center Library -->
<dependency>
    <groupId>ar.com.bds.lib</groupId>
    <artifactId>people-center-model</artifactId>
    <version>0.10.5-SNAPSHOT</version>
</dependency>
```

### Inherited from Parent (bds-parent-springboot)
- Lombok
- MapStruct
- OpenAPI/Swagger
- Jaeger (tracing)
- Actuator
- Validation API

---

## 🔧 Configuración

### Variables de Entorno Requeridas

```bash
# Database
DATABASE_DATASOURCE_PEOPLE_CENTER=jdbc:mysql://localhost:3306/people-center
DATABASE_USERNAME_PEOPLE_CENTER=root
DATABASE_PASSWORD_PEOPLE_CENTER=root
DATABASE_DATASOURCE_PEOPLE_HUB=jdbc:mysql://localhost:3306/people-hub
DATABASE_USERNAME_PEOPLE_HUB=root
DATABASE_PASSWORD_PEOPLE_HUB=root

# Logging
LOG_PROFILE=local
LOG_LEVEL=debug

# Jackson
JSON_PROPERTY_NAMING=LOWER_CAMEL_CASE
TIMEZONE=America/Buenos_Aires

# Actuator
BASE_ACTUATOR_ENDPOINT=/actuator
EXPOSED_ACTUATOR_ENDPOINTS=info,health,prometheus

# Tracing
JAEGER_AGENT_HOST=localhost
JAEGER_AGENT_PORT=6831
TRACE_LOG_SPANS=false

# OpenAPI
SPECS_URL=/people/center/docs

# External Services
CUSTOMER_DELOREAN_CONNECTOR_BASE_URL=https://bs-int.bdsdigital.com.ar
CUSTOMER_DELOREAN_CONNECTOR_URN=/v1/customers

# People Hub
PEOPLE_HUB_BASE_URL=https://people-int.bdsdigital.com.ar/
PEOPLE_HUB_RESOURCE_UPDATE_PROFILE=/persons/{personId}/profile
PEOPLE_HUB_REST_CLIENT_TIMEOUT_IN_MS=1000
IMPORT_PERSON_FROM_PPH=false

# Cache
PC_CACHE_ENABLED=false

# Redis
REDIS_HOST_SSL=localhost
REDIS_AUTH_PASS=
REDIS_PORT=6379
REDIS_TTL=3600
```

---

## 🧪 Testing

### Estructura de Tests

```
src/test/java/
├── ar/com/bds/people/center/
│   ├── controller/         # Tests de integración (IT)
│   └── service/            # Tests unitarios
└── resources/
    ├── application-test.yml
    └── test-data/          # SQL scripts para tests
```

### Convenciones de Testing

#### Tests Unitarios
- **Naming:** `{Class}Test.java`
- **Framework:** JUnit 5 + Mockito
- **Anotaciones:**
  ```java
  @ExtendWith(MockitoExtension.class)
  @DisplayName("{Entity}Service - Unit Tests")
  ```
- **Estructura:**
  ```java
  @Mock
  private Repository repository;
  
  @InjectMocks
  private ServiceImpl service;
  
  @BeforeEach
  void setUp() { ... }
  
  @Test
  @DisplayName("methodName() - Should... When...")
  void methodName_ShouldDoSomething_WhenCondition() { ... }
  ```

#### Tests de Integración
- **Naming:** `{Class}IT.java` o `{Class}IntegrationTest.java`
- **Anotaciones:**
  ```java
  @SpringBootTest
  @AutoConfigureMockMvc
  @Transactional
  ```
- **Nota:** Requieren configuración completa del contexto de Spring

---

## 🔄 Flujo de Trabajo Git

### Branching Strategy
- **develop:** Rama principal de desarrollo
- **feature/CON-XX-descripcion:** Features nuevos
- **fix/CON-XX-descripcion:** Bug fixes
- **hotfix/:** Fixes urgentes para producción

### Commit Messages (Conventional Commits)
```
<type>: <description>

<body>

<footer>
```

**Types:**
- `feat`: Nueva funcionalidad
- `fix`: Bug fix
- `docs`: Documentación
- `style`: Formato, sin cambios de código
- `refactor`: Refactorización
- `test`: Tests
- `chore`: Tareas de mantenimiento

**Ejemplo:**
```
fix: update exception messages in PersonCertificationService

Change ResourceNotFoundException messages to English to maintain 
consistency with the rest of the project.

- Change 'Persona no encontrada' to 'Person not found'
- Change 'Certificación no encontrada' to 'Certification not found'
- Update tests to expect English exception messages
- All tests now pass successfully (394 tests, 0 failures)
```

---

## 📝 Endpoints API

### Patrón de URLs
```
/v2/people/{personId}/{resource}
/v2/people/{personId}/{resource}/{idEntity}
```

### Ejemplo: Certifications
```
POST   /v2/people/{personId}/certifications
GET    /v2/people/{personId}/certifications
GET    /v2/people/{personId}/certifications/{idEntity}
DELETE /v2/people/{personId}/certifications/{idEntity}
```

### Respuestas HTTP
- **201 Created:** Recurso creado exitosamente (retorna ID)
- **200 OK:** Operación exitosa
- **204 No Content:** Eliminación exitosa
- **400 Bad Request:** Validación fallida
- **404 Not Found:** Recurso no encontrado
- **500 Internal Server Error:** Error del servidor

---

## 🚀 Comandos Útiles

### Desarrollo Local
```bash
# Ejecutar aplicación
mvn spring-boot:run -Dspring-boot.run.arguments=--spring.profiles.active=local

# Ejecutar tests
mvn test

# Ejecutar test específico
mvn test -Dtest=PersonCertificationServiceTest

# Compilar sin tests
mvn clean install -DskipTests

# Compilar con tests
mvn clean install
```

### Docker
```bash
# Build
mvn clean install
docker build -t people-center .

# Run
docker run -e "LOG_PROFILE=local" -e "JSON_PROPERTY_NAMING=LOWER_CAMEL_CASE" people-center
```

---

## 📚 Recursos

### Documentación
- **Swagger UI:** `http://localhost:8080/swagger-api.html`
- **OpenAPI Docs:** `/people-center/docs`
- **Confluence:** [Base de datos única de personas (BUP)](https://gss-bdsol.atlassian.net/wiki/spaces/DA/pages/2318991361)

### Repositorios
- **Código:** GitHub (gss-bds/people-center)
- **DevOps:** GitHub (gss-bds/devops/people-center)
- **Charts:** GitHub (gss-bds/charts)

### CI/CD
- **Azure DevOps:** Build pipeline
- **Artifacts:** Azure Artifacts (bdsdigital feed)

---

## ⚠️ Notas Importantes

1. **Soft Delete:** NUNCA eliminar físicamente registros, usar `deleted_at`
2. **Mensajes en Inglés:** Todos los mensajes de excepción y logs deben estar en inglés
3. **Validaciones:** Siempre validar en el service layer, no solo en el controller
4. **Transacciones:** Usar `@Transactional` en operaciones que modifican datos
5. **Cache:** Considerar invalidación de cache en operaciones de escritura
6. **Migraciones:** NUNCA modificar migraciones existentes, crear nuevas
7. **Tests:** Mantener cobertura de tests unitarios para servicios
8. **Lombok:** Usar `@RequiredArgsConstructor` para inyección de dependencias
9. **Logging:** Usar SLF4J con niveles apropiados
10. **Circular Dependencies:** Usar `@Autowired` + `@Setter` + `@Lazy` cuando sea necesario

---

## 🎯 Checklist para Nuevas Features

- [ ] Crear migración Flyway si requiere cambios en BD
- [ ] Crear/actualizar Entity
- [ ] Crear/actualizar Repository
- [ ] Crear/actualizar Mapper
- [ ] Crear/actualizar Service (interface + impl)
- [ ] Crear/actualizar Controller
- [ ] Crear/actualizar DTOs en library/model
- [ ] Escribir tests unitarios
- [ ] Actualizar documentación Swagger
- [ ] Verificar que todos los tests pasen
- [ ] Seguir convenciones de naming
- [ ] Mensajes de excepción en inglés
- [ ] Commit siguiendo Conventional Commits
- [ ] Actualizar CHANGELOG si es necesario

---

**Última actualización:** 2025-12-01
**Versión del proyecto:** 1.8.10-rc.1-SNAPSHOT
**Autor del análisis:** Antigravity AI Assistant
