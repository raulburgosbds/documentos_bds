# Guía de Workflow Git - People Center

## Convención de Commits (Conventional Commits)

### Estructura del Commit

```
<tipo>[ámbito opcional]: <descripción corta>

[cuerpo opcional: detalles del cambio]

[nota al pie: BREAKING CHANGE o referencias a tickets]
```

### Tipos de Commits

| Tipo | Descripción | Impacto SemVer |
|------|-------------|----------------|
| `fix` | Arregla un bug | PATCH |
| `feat` | Añade una nueva característica | MINOR |
| `build` | Cambios en compilación (Maven, Gradle) | Patch / Ninguno |
| `chore` | Mantenimiento (dependencias, limpieza) | Patch / Ninguno |
| `ci` | Integración continua (Jenkins, Actions) | Patch / Ninguno |
| `docs` | Solo cambios en documentación | Patch / Ninguno |
| `style` | Formato (espacios) sin cambiar lógica | Patch / Ninguno |
| `refactor` | Cambio de código (ni fix, ni feat) | Patch / Ninguno |
| `test` | Añadir o corregir tests | Patch / Ninguno |

### Ejemplos Prácticos

#### PATCH (Fix simple)
```bash
git commit -m "fix(login): corregir error de null pointer en validación de usuario"
```

#### MINOR (Nueva feature)
```bash
git commit -m "feat(certifications): agregar gestión de certificaciones de personas

- Agregar modelos Certification y PersonCertification en library
- Agregar CreatePersonCertificationRequest con validaciones
- Agregar entidades y repositorios en microservice
- Agregar migración V26 para tablas de certificaciones"
```

#### MAJOR (Breaking Change - Opción 1)
```bash
git commit -m "feat(api)!: cambiar autenticación de Basic a Bearer Token"
```

#### MAJOR (Breaking Change - Opción 2)
```bash
git commit -m "refactor: eliminar métodos obsoletos de la clase Cliente

BREAKING CHANGE: getEdad() ha sido eliminado. Usar calcularEdad()."
```

---

## Workflow Completo para Nueva Feature

### 1. Antes de empezar a codear

```bash
# a) Asegurarte de estar en la rama principal actualizada
git checkout develop  # o develop, según tu proyecto
git pull origin develop

# b) Crear la rama ANTES de hacer cambios
git checkout -b feature/nombre-de-tu-feature

# c) Verificar que estás en la rama correcta
git branch
```

### 2. Durante el desarrollo

```bash
# Hacer cambios en el código...

# Ver qué cambió
git status

# Agregar archivos específicos
git add <archivo1> <archivo2>
# O agregar todos los cambios
git add .

# Commit siguiendo la convención
git commit -m "tipo(ámbito): descripción corta

- Detalle 1
- Detalle 2
- Detalle 3"

# Push (primera vez)
git push -u origin feature/nombre-de-tu-feature

# Push (siguientes veces)
git push
```

### 3. Al terminar la feature

```bash
# Actualizar tu rama con los últimos cambios de main
git fetch origin
git rebase origin/main  # o git merge origin/main

# Resolver conflictos si los hay

# Push final
git push

# Crear Pull Request en GitHub/GitLab/Bitbucket
```

---

## Checklist para cada nueva feature

```
☐ 1. git checkout main
☐ 2. git pull origin main
☐ 3. git checkout -b feature/nombre-descriptivo
☐ 4. Codear y hacer commits
☐ 5. git push origin feature/nombre-descriptivo
☐ 6. Crear Pull Request
```

---

## ¿Olvidaste crear la rama antes de codear?

### Solución: Crear la rama ahora (con tus cambios)

```bash
# 1. Verificar en qué rama estás
git branch

# 2. Crear y cambiar a la nueva rama (CON tus cambios)
git checkout -b feature/nombre-de-tu-feature

# 3. Verificar que estás en la rama correcta
git branch
# Debería mostrar:
#   main
# * feature/nombre-de-tu-feature  <- El asterisco indica la rama actual

# 4. Ver tus cambios
git status

# 5. Hacer el commit normalmente
git add .
git commit -m "feat(ámbito): descripción"

# 6. Push de la rama
git push origin feature/nombre-de-tu-feature
```

✅ **¡Listo!** Todos tus cambios ahora están en la nueva rama.

---

## Comandos Útiles

### Si ya hiciste commit en la rama equivocada

```bash
# Estás en main con commits que no deberías tener
git checkout -b feature/mi-feature  # Crea rama con tus commits
git checkout main                    # Vuelve a main
git reset --hard origin/main         # Resetea main al estado remoto
```

### Cambiar el nombre de tu rama

```bash
git branch -m feature/nuevo-nombre
```

### Ver todas las ramas

```bash
git branch -a  # Locales y remotas
git branch     # Solo locales
```

### Eliminar una rama local

```bash
git branch -d feature/nombre-rama  # Solo si ya fue mergeada
git branch -D feature/nombre-rama  # Forzar eliminación
```

### Actualizar tu rama con cambios de main

```bash
# Opción 1: Rebase (historial más limpio)
git fetch origin
git rebase origin/main

# Opción 2: Merge (más seguro)
git fetch origin
git merge origin/main
```

---

## Convención de Nombres de Ramas

### Formato recomendado

```
<tipo>/<descripción-corta-con-guiones>
```

### Tipos comunes

- `feature/` - Nueva funcionalidad
- `fix/` - Corrección de bug
- `hotfix/` - Corrección urgente en producción
- `refactor/` - Refactorización
- `docs/` - Documentación
- `test/` - Tests

### Ejemplos

```bash
# ✅ BIEN
feature/person-certifications
feature/add-email-validation
fix/null-pointer-in-login
hotfix/critical-security-issue
refactor/simplify-mapper-logic
docs/update-api-documentation

# ❌ MAL
feature/test
feature/cambios
feature/raul-branch
mi-rama
```

---

## Guía Rápida: Cuándo usar cada tipo de commit

| Situación | Tipo | Ejemplo |
|-----------|------|---------|
| Agregar endpoint nuevo | `feat` | `feat(api): agregar endpoint GET /certifications` |
| Agregar validación nueva | `feat` | `feat(validation): agregar validación de fechas` |
| Corregir bug | `fix` | `fix(certifications): corregir error al eliminar` |
| Actualizar dependencia | `chore` | `chore: actualizar Spring Boot a 2.7.0` |
| Agregar test | `test` | `test(certifications): agregar tests para service` |
| Refactorizar código | `refactor` | `refactor(mapper): simplificar lógica` |
| Actualizar README | `docs` | `docs: actualizar documentación de API` |
| Cambiar formato código | `style` | `style: aplicar formato con prettier` |
| Cambiar configuración Maven | `build` | `build: actualizar plugin de Maven compiler` |
| Cambiar pipeline CI/CD | `ci` | `ci: agregar step de análisis de código` |

---

## Estrategia para Library + Microservice

### ✅ Recomendado: Un solo commit por feature completa

Dado que `library` y `microservice` están en el **mismo repositorio** y son **interdependientes**:

```bash
# Agregar TODOS los archivos relacionados con la feature
git add library/model/src/main/java/ar/com/bds/lib/peoplecenter/model/
git add microservice/src/main/java/ar/com/bds/people/center/entity/
git add microservice/src/main/java/ar/com/bds/people/center/repository/
git add microservice/src/main/resources/db/migration/

# Un solo commit que incluye library + microservice
git commit -m "feat(certifications): agregar gestión de certificaciones

- Agregar modelos en library
- Agregar entidades y repositorios en microservice
- Agregar migración de base de datos"
```

**Ventajas:**
- Mantiene la atomicidad de la feature
- Más fácil hacer rollback si algo falla
- El historial es más limpio
- No hay commits que rompan la compilación

### ❌ NO recomendado: Commits separados

```bash
# MAL - Rompe la compilación entre commits
git commit -m "feat(library): agregar modelos"
# ⚠️ Microservice no compila porque falta la migración

git commit -m "feat(microservice): agregar entidades"
# ⚠️ Ahora sí compila, pero el historial tiene un commit roto
```

---

## Resolución de Conflictos

### Si hay conflictos durante rebase/merge

```bash
# 1. Git te mostrará los archivos en conflicto
git status

# 2. Abrir cada archivo y resolver conflictos manualmente
# Buscar las marcas: <<<<<<<, =======, >>>>>>>

# 3. Marcar como resuelto
git add <archivo-resuelto>

# 4. Continuar el rebase/merge
git rebase --continue  # Si estabas haciendo rebase
git merge --continue   # Si estabas haciendo merge

# 5. Si quieres cancelar todo
git rebase --abort
git merge --abort
```

---

## Tips y Mejores Prácticas

### ✅ Hacer

- Crear rama ANTES de empezar a codear
- Commits pequeños y frecuentes
- Mensajes de commit descriptivos
- Pull/rebase antes de push
- Revisar cambios con `git status` y `git diff` antes de commit

### ❌ Evitar

- Commits gigantes con muchos cambios no relacionados
- Mensajes vagos como "fix", "cambios", "update"
- Trabajar directamente en `main` o `develop`
- Hacer push sin revisar los cambios
- Commits que rompen la compilación

---

## Comandos de Emergencia

### Deshacer el último commit (manteniendo cambios)

```bash
git reset --soft HEAD~1
```

### Deshacer el último commit (descartando cambios)

```bash
git reset --hard HEAD~1
```

### Descartar todos los cambios locales

```bash
git reset --hard HEAD
git clean -fd  # Eliminar archivos no rastreados
```

### Ver historial de commits

```bash
git log --oneline --graph --all
```

### Ver diferencias antes de commit

```bash
git diff              # Cambios no staged
git diff --staged     # Cambios staged
```

---

## Recursos Adicionales

- [Conventional Commits](https://www.conventionalcommits.org/)
- [Semantic Versioning](https://semver.org/)
- [Git Flow](https://nvie.com/posts/a-successful-git-branching-model/)
