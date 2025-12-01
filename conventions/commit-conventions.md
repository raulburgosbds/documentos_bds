Aquí tienes la versión limpia, sin iconos gráficos, ideal para documentación técnica o wikis.

-----

# CHULETA: CONVENTIONAL COMMITS Y SEMVER

## 1\. La Regla de Oro (SemVer)

El número de versión es **X.Y.Z** (ej. 1.5.2).

| Parte | Nombre | Significado | Disparador en Commit |
| :--- | :--- | :--- | :--- |
| **X** | **MAJOR** | Cambio incompatible (Rompe API) | `BREAKING CHANGE` o `!` |
| **Y** | **MINOR** | Nueva funcionalidad (Compatible) | `feat` |
| **Z** | **PATCH** | Corrección de error (Compatible) | `fix` |

-----

## 2\. Estructura del Commit

```text
<tipo>[ámbito opcional]: <descripción corta>

[cuerpo opcional: detalles del cambio]

[nota al pie: BREAKING CHANGE o referencias a tickets]
```

-----

## 3\. Tipos de Commits

| Tipo | Descripción | Impacto SemVer |
| :--- | :--- | :--- |
| **fix** | Arregla un bug. | PATCH |
| **feat** | Añade una nueva característica. | MINOR |
| **build** | Cambios en compilación (Maven, Gradle). | Patch / Ninguno |
| **chore** | Mantenimiento (dependencias, limpieza). | Patch / Ninguno |
| **ci** | Integración continua (Jenkins, Actions). | Patch / Ninguno |
| **docs** | Solo cambios en documentación. | Patch / Ninguno |
| **style** | Formato (espacios) sin cambiar lógica. | Patch / Ninguno |
| **refactor**| Cambio de código (ni fix, ni feat). | Patch / Ninguno |
| **test** | Añadir o corregir tests. | Patch / Ninguno |

-----

## 4\. Ejemplos Prácticos

### PATCH (Fix simple)

```text
fix(login): corregir error de null pointer en validación de usuario
```

### MINOR (Nueva feature)

```text
feat: agregar soporte para exportar reportes en PDF
```

### MAJOR (Breaking Change - Opción 1)

```text
feat(api)!: cambiar autenticación de Basic a Bearer Token
```

### MAJOR (Breaking Change - Opción 2)

```text
refactor: eliminar métodos obsoletos de la clase Cliente

BREAKING CHANGE: getEdad() ha sido eliminado. Usar calcularEdad().
```

-----

## 5\. Configuración Maven (Automatización)

Añadir al `pom.xml` para calcular versiones automáticamente con `mvn release:prepare`.

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-release-plugin</artifactId>
    <version>3.0.0</version>
    <configuration>
        <projectVersionPolicyId>ConventionalCommits</projectVersionPolicyId>
    </configuration>
    <dependencies>
        <dependency>
            <groupId>nl.basjes.maven.release</groupId>
            <artifactId>conventional-commits-version-policy</artifactId>
            <version>1.0.6</version>
        </dependency>
    </dependencies>
</plugin>
```