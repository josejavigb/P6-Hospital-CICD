# Guía P6 — CI con GitHub Actions

## ¿Qué hace este workflow?

El fichero `.github/workflows/ci.yml` define un pipeline de integración continua que se ejecuta automáticamente cada vez que se hace un push o una pull request a `main`. El pipeline:

1. Descarga el código del repositorio
2. Configura Java 17
3. Compila el proyecto con Maven
4. Ejecuta los tests de integración
5. Publica un informe de resultados en GitHub
6. Genera badges de cobertura de código con JaCoCo
7. Sube los resultados como artifacts descargables
8. Guarda los badges automáticamente en el repositorio

---

## Estructura del workflow

### Trigger (`on:`)

```yaml
on:
  push:
    branches: ["main", "master"]
  pull_request:
    branches: ["main", "master"]
```

El workflow se dispara en dos situaciones:
- **push**: cuando se sube código directamente a `main` o `master`
- **pull_request**: cuando se abre o actualiza una PR hacia esas ramas

Se filtra por rama para no gastar minutos de GitHub Actions en ramas de desarrollo.

---

### Permisos (`permissions:`)

```yaml
permissions:
  contents: write
  checks: write
  pull-requests: write
```

Se definen a nivel de workflow (fuera de `jobs:`) para que apliquen a todos los jobs.

| Permiso | Para qué |
|---|---|
| `contents: write` | Permite hacer `git push` al repo (necesario para guardar los badges) |
| `checks: write` | Permite crear el informe de tests en la pestaña Checks de GitHub |
| `pull-requests: write` | Permite anotar las PRs con los resultados de tests |

---

### Runner (`runs-on:`)

```yaml
runs-on: ubuntu-latest
```

El job corre en un servidor de GitHub (cloud). No requiere ninguna configuración local. GitHub proporciona una máquina virtual con Ubuntu, 2 cores, 7 GB de RAM y 14 GB de disco de forma gratuita para repositorios públicos.

---

## Steps detallados

### 1. Checkout

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

Clona el repositorio en el runner. **Siempre es el primer step** — sin él el runner no tiene acceso al código.

---

### 2. Configurar Java 17

```yaml
- name: Set up Java 17
  uses: actions/setup-java@v4
  with:
    java-version: '17'
    distribution: 'temurin'
    cache: maven
```

Instala el JDK 17 de la distribución Temurin (Eclipse). El parámetro `cache: maven` guarda la carpeta `~/.m2/repository` entre ejecuciones — en la primera run descarga todas las dependencias del `pom.xml`, en las siguientes las restaura desde caché. Ahorra varios minutos por ejecución.

---

### 3. Dar permisos al Maven Wrapper

```yaml
- name: Give permissions to mvnw
  run: chmod +x ./mvnw
```

En Linux (Ubuntu) los ficheros no tienen permisos de ejecución por defecto. Sin este step, `./mvnw` fallaría con `Permission denied`. Va **antes** de cualquier comando que use `./mvnw`.

---

### 4. Compilar

```yaml
- name: Compile
  run: ./mvnw compile --no-transfer-progress
```

Compila las 20 clases del proyecto. `--no-transfer-progress` suprime las barras de progreso de descarga de dependencias, dejando los logs limpios y legibles. Equivalente a `-B` (`--batch-mode`) pero más específico.

---

### 5. Ejecutar tests de integración

```yaml
- name: Run integration tests
  run: ./mvnw verify --no-transfer-progress
```

Ejecuta todos los tests del proyecto:
- **Tests unitarios** via plugin Surefire (clases `*Test.java`) con `mvn test`
- **Tests de integración** via plugin Failsafe (clases `*IT.java`) — se añaden con `verify`

Los tests de integración del proyecto cubren:
- `MedicoControllerMockMvcIT` — CRUD de médicos con MockMvc (sin servidor completo)
- `PacienteControllerMockMvcIT` — CRUD de pacientes con MockMvc
- `ImagenControllerWebTestClientIT` — subida de imágenes con WebTestClient (servidor completo)
- `InformeControllerWebTestClientIT` — ciclo de vida de informes con WebTestClient

> **¿Por qué MockMvc para médicos/pacientes y WebTestClient para imágenes?**
> MockMvc no levanta el servidor HTTP real — es más rápido y suficiente para GET/POST simples.
> WebTestClient levanta el servidor completo (`RANDOM_PORT`) — necesario para subida de ficheros multipart y redirecciones.

---

### 6. Publicar informe de tests

```yaml
- name: Publish test report
  if: always()
  uses: ctrf-io/github-test-reporter@v1
  with:
    report-path: './target/*-reports/*.xml'
    github-report: true
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Lee los XMLs que generan Surefire y Failsafe y los muestra directamente en GitHub — en el resumen del workflow y en la pestaña Checks de cada PR. Muestra qué tests pasaron, cuáles fallaron y cuáles se saltaron sin necesidad de descargar ningún artifact.

`if: always()` es crítico aquí: si los tests fallan, este step se ejecuta igualmente para poder ver qué falló.

`GITHUB_TOKEN` es un secret automático que GitHub inyecta en cada workflow — no hay que configurarlo manualmente.

---

### 7. Generar badge de cobertura JaCoCo

```yaml
- name: Generate Jacoco badge
  id: jacoco
  if: always()
  uses: cicirello/jacoco-badge-generator@v2
  with:
    generate-branches-badge: true
    generate-summary: true
    jacoco-csv-file: target/site/jacoco/jacoco.csv
    badges-directory: .github/badges
```

JaCoCo mide qué porcentaje del código se ejecuta durante los tests. Durante `mvn verify`, JaCoCo instrumenta el código y genera `target/site/jacoco/jacoco.csv` con los datos de cobertura.

Esta Action lee ese CSV y genera dos ficheros SVG:
- `.github/badges/jacoco.svg` — badge de cobertura de líneas
- `.github/badges/branches.svg` — badge de cobertura de ramas

`id: jacoco` es necesario para referenciar los outputs de este step en el step siguiente.

`generate-summary: true` muestra el resumen de cobertura directamente en la página del workflow.

---

### 8. Log de cobertura

```yaml
- name: Log coverage percentage
  run: |
    echo "coverage = ${{ steps.jacoco.outputs.coverage }}"
    echo "branch coverage = ${{ steps.jacoco.outputs.branches }}"
```

Imprime los porcentajes en los logs del workflow. Usa `steps.jacoco.outputs` para acceder a los valores calculados por el step anterior — posible gracias al `id: jacoco`.

---

### 9. Upload de resultados de tests

```yaml
- name: Upload test results
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: test-results
    path: |
      target/surefire-reports/
      target/failsafe-reports/
```

Guarda los reportes XML de Surefire y Failsafe como artifact descargable desde GitHub (se retienen 90 días por defecto). Útil para inspeccionar los resultados en detalle fuera de GitHub.

---

### 10. Upload del informe JaCoCo

```yaml
- name: Upload Jacoco report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: jacoco-report
    path: target/site/jacoco/
```

Guarda el informe HTML completo de JaCoCo como artifact. Contiene el desglose de cobertura por clase y método, con el código fuente anotado línea a línea.

---

### 11. Guardar badges en el repositorio

```yaml
- name: Commit badge to repo
  if: github.event_name == 'push'
  uses: EndBug/add-and-commit@v9
  with:
    add: '.github/badges'
    message: 'Update coverage badges'
```

Hace commit de los SVGs generados al repositorio. Así los badges persisten entre ejecuciones y pueden referenciarse desde el README.

`if: github.event_name == 'push'` — solo en pushes reales, no en PRs. Evita que el workflow intente hacer push en el contexto de una PR (donde no tendría permisos).

---

## Badges en el README

Los badges se añaden al README apuntando a los ficheros SVG del repo y al estado del workflow:

```markdown
![CI](https://github.com/TU_USUARIO/TU_REPO/actions/workflows/ci.yml/badge.svg?branch=main)
![Coverage](.github/badges/jacoco.svg)
![Branches](.github/badges/branches.svg)
```

| Badge | Qué muestra |
|---|---|
| CI | Estado del último workflow (passing / failing) |
| Coverage | % de líneas de código cubiertas por los tests |
| Branches | % de ramas de código cubiertas por los tests |

---

## Diferencia entre `mvn test` y `mvn verify`

| Comando | Plugin | Clases que ejecuta | Cuándo usarlo |
|---|---|---|---|
| `./mvnw test` | Surefire | `*Test.java` | Solo tests unitarios |
| `./mvnw verify` | Surefire + Failsafe | `*Test.java` + `*IT.java` | Tests unitarios + integración |

En este proyecto se usa `verify` porque los tests de integración (los `*IT.java`) son los más importantes para validar la API completa.

---

## Resumen del flujo completo

```
git push → GitHub detecta el push
         → Arranca runner ubuntu-latest
         → Checkout del código
         → Instala Java 17 (desde caché si ya existe)
         → chmod +x ./mvnw
         → mvn compile → compila 20 clases
         → mvn verify  → ejecuta tests unitarios + integración
         → ctrf-io     → publica informe en GitHub Checks
         → jacoco      → genera badges SVG + summary
         → upload      → guarda artifacts (reportes + jacoco)
         → add-commit  → guarda badges en el repo
         → ✅ Done
```
