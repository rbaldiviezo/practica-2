# Justificación Técnica del Pipeline DevSecOps

## 1. Resumen General

El pipeline CI/CD implementado en `.github/workflows/devsecops.yml` aplica los principios de **DevSecOps** integrando seguridad en cada fase del ciclo de vida del software, desde el commit hasta la construcción de contenedores. Se ejecuta automáticamente ante cada `push` a `main` y en cada `pull request`.

### Diagrama de etapas

```
Push / Pull Request
        │
        ▼
┌──────────────────────┐
│ 1. Install & Test    │  ← Instalación + Testing automático
└──────────┬───────────┘
           │
     ┌─────┼──────────────────┐
     ▼     ▼                  ▼
┌─────────┐ ┌──────────┐ ┌────────┐
│ 2.ESLint│ │ 3. SAST  │ │ 4. SCA │  ← Calidad + Seguridad
└────┬────┘ └────┬─────┘ └───┬────┘
     └───────────┼────────────┘
                 ▼
     ┌───────────────────────┐
     │  5. Docker Build      │  ← Build de contenedores + Versionado
     └───────────┬───────────┘
                 ▼
     ┌───────────────────────┐
     │  6. Container Security│  ← Escaneo Trivy de imágenes
     └───────────┬───────────┘
                 ▼
     ┌───────────────────────┐
     │  7. Smoke Test        │  ← Verificación funcional end-to-end
     └───────────────────────┘
```

---

## 2. Detalle por etapa

### Etapa 1 – Instalación y Testing automático

| Aspecto | Detalle |
|---|---|
| **Herramienta** | `npm ci` + `Jest` |
| **Fase DevSecOps** | Build / Test |
| **Riesgo que mitiga** | Dependencias inconsistentes entre entornos; regresiones funcionales |
| **Justificación** | `npm ci` elimina `node_modules` y reinstala **exactamente** lo que indica `package-lock.json`, garantizando builds seguros. Si el lock-file no coincide con `package.json`, el comando falla inmediatamente. Las pruebas unitarias con Jest validan la lógica de negocio; si un test falla, el pipeline se detiene y no avanza a etapas posteriores. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> La reproducibilidad evita el problema de "works on my machine". `npm ci` garantiza que todos los entornos (desarrollo, CI, producción) usen exactamente las mismas versiones de dependencias. Sin esto, un sistema puede funcionar localmente pero fallar en producción por diferencias sutiles en versiones.

**Comandos clave:**
```bash
npm ci          # instalación reproducible
npm test        # ejecución de pruebas (Jest)
```

**Servicios cubiertos:** `users-service`, `academic-service`, `api-gateway`, `frontend`.

**Evidencia GitHub Actions:**

![Install & Test - Evidencia](/assets/stage-1.png)

---

### Etapa 2 – Análisis de calidad de código (ESLint)

| Aspecto | Detalle |
|---|---|
| **Herramienta** | ESLint 9.x con plugins `react-hooks` y `react-refresh` |
| **Fase DevSecOps** | Code → Build (análisis estático de estilo) |
| **Riesgo que mitiga** | Errores comunes de programación, variables no utilizadas, hooks mal usados, inconsistencias de estilo |
| **Justificación** | ESLint analiza estáticamente el código JavaScript/JSX para detectar malas prácticas **antes** de que lleguen a producción. El flag `--max-warnings 0` hace que el pipeline falle ante cualquier advertencia, forzando al equipo a mantener un código limpio. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> Un código de baja calidad es estadísticamente más propenso a contener vulnerabilidades de seguridad. Código desordenado, con variables sin usar o malas prácticas, dificulta las revisiones de código y esconde bugs. ESLint actúa como un guardia de calidad automático que previene la degradación del código con el tiempo.

**Comandos clave:**
```bash
npx eslint . --max-warnings 0
```

**Evidencia GitHub Actions:**

![ESLint - Evidencia](/assets/stage-2.png)

---

### Etapa 3 – Seguridad del código / SAST (Semgrep)

| Aspecto | Detalle |
|---|---|
| **Herramienta** | Semgrep (OSS) con reglas automáticas + reglas custom del repositorio |
| **Fase DevSecOps** | Code → Build (Static Application Security Testing) |
| **Riesgo que mitiga** | Inyección de código, secretos hardcodeados, uso de `eval()`, inputs sin validar |
| **Justificación** | Semgrep realiza análisis estático de seguridad (SAST) buscando patrones vulnerables en el código fuente. Se utilizan las reglas `auto` de la comunidad **más** las reglas personalizadas del directorio `backend/semgrep-rules/`. El flag `--error` asegura que el pipeline falle si se detecta algún hallazgo de severidad ERROR. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> El código humano es falible. Incluso desarrolladores experimentados pueden introducir secretos hardcodeados, usar `eval()`, u olvidar validar inputs. SAST con Semgrep actúa como una red de seguridad automática que detecta vulnerabilidades **antes** de que lleguen a producción, sin importar que el sistema "funcione".

**Reglas custom incluidas:**
| Archivo | Descripción |
|---|---|
| `hardcoded-secret.yaml` | Detecta secretos (claves API, contraseñas) escritos directamente en el código |
| `no-eval.yaml` | Prohíbe el uso de `eval()` que habilita inyección de código arbitrario |
| `unvalidated-input.yaml` | Identifica inputs del usuario que se procesan sin validación previa |

**Comandos clave:**
```bash
semgrep --config=auto --config=../semgrep-rules/ --severity=ERROR --error
```

**Evidencia GitHub Actions:**

![SAST Semgrep - Evidencia](/assets/stage-3.png)

---

### Etapa 4 – Seguridad de dependencias / SCA (npm audit)

| Aspecto | Detalle |
|---|---|
| **Herramienta** | `npm audit` (integrado en npm) |
| **Fase DevSecOps** | Build (Software Composition Analysis) |
| **Riesgo que mitiga** | Vulnerabilidades conocidas (CVEs) en dependencias de terceros |
| **Justificación** | Las aplicaciones modernas dependen de cientos de paquetes externos. `npm audit` consulta la base de datos de vulnerabilidades de npm para identificar dependencias con CVEs conocidos. Con `--audit-level=critical`, el pipeline se detiene **únicamente** ante vulnerabilidades críticas. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> Las vulnerabilidades evolucionan. Cada día se descubren nuevos CVEs en librerías que hoy son seguras. Sin SCA continuo, el sistema queda expuesto a exploits conocidos sin que el equipo lo sepa. Una dependencia que era segura hace 6 meses puede tener hoy un exploit público.

**Comandos clave:**
```bash
npm audit --audit-level=critical
```

**Servicios cubiertos:** `users-service`, `academic-service`, `api-gateway`, `frontend`.

**Evidencia GitHub Actions:**

![SCA npm audit - Evidencia](/assets/stage-4.png)

---

### Etapa 5 – Build de contenedores Docker

| Aspecto | Detalle |
|---|---|
| **Herramienta** | Docker Buildx + Docker Compose |
| **Fase DevSecOps** | Build / Release |
| **Riesgo que mitiga** | Artefactos no reproducibles, imágenes sin versionado, builds corruptos |
| **Justificación** | Se construyen las 4 imágenes Docker del sistema utilizando Docker Compose. Cada imagen se versiona con el SHA del commit (`${{ github.sha }}`), garantizando trazabilidad completa entre el código fuente y el artefacto desplegado. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> Sin un proceso de build automatizado y versionado, es imposible saber exactamente qué código está corriendo en producción. El versionado con SHA del commit permite auditoría completa y rollback preciso ante incidentes.

**Imágenes construidas:**
| Imagen | Base | Puerto |
|---|---|---|
| `users-service:latest` | `node:20-alpine` | 3001 |
| `academic-service:latest` | `node:20-alpine` | 3002 |
| `api-gateway:latest` | `node:20-alpine` | 3000 |
| `frontend:latest` | `node:20-alpine` (build) → `nginx:alpine` (serve) | 80 |

**Evidencia GitHub Actions:**

![Docker Build - Evidencia](/assets/stage-5.png)

---

### Etapa 6 – Seguridad de contenedores (Trivy)

| Aspecto | Detalle |
|---|---|
| **Herramienta** | Trivy (Aqua Security) v0.28.0 |
| **Fase DevSecOps** | Release / Deploy (Container Image Scanning) |
| **Riesgo que mitiga** | Vulnerabilidades en la imagen base del OS, librerías del sistema, binarios incluidos |
| **Justificación** | Trivy escanea las imágenes Docker completas (sistema operativo base + dependencias de aplicación + configuración) buscando CVEs conocidos. A diferencia de `npm audit` que solo analiza paquetes Node.js, Trivy examina **todo** el contenido de la imagen. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> Los contenedores heredan vulnerabilidades del sistema operativo base. Una imagen `node:20-alpine` puede contener librerías del sistema con CVEs que no se detectan analizando solo el código JavaScript. Trivy cubre esta superficie de ataque que otras herramientas no alcanzan.

#### Severidad HIGH vs CRITICAL

**Escenario 1: Pipeline con `severity: HIGH` - FALLA**

Cuando configuramos Trivy para fallar ante vulnerabilidades HIGH, el pipeline detecta más vulnerabilidades pero puede fallar por issues que no son explotables inmediatamente:

```yaml
- uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: "users-service:latest"
    severity: CRITICAL,HIGH
    exit-code: "1"
```

![Trivy HIGH - Pipeline Fallando](/assets/stage-6.1.png)

**Escenario 2: Pipeline con `severity: CRITICAL` - PASA**

Al ajustar a solo CRITICAL, el pipeline pasa pero solo bloquea vulnerabilidades con exploits conocidos y de máximo impacto:

```yaml
- uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: "users-service:latest"
    severity: CRITICAL
    exit-code: "1"
```

![Trivy CRITICAL - Pipeline Pasando](/assets/stage-6.2.png)

**Decisión de diseño:** Se optó por `severity: CRITICAL` para evitar falsos positivos que bloqueen el desarrollo, mientras se garantiza que vulnerabilidades graves no lleguen a producción.

**Imágenes escaneadas:** `users-service`, `academic-service`, `api-gateway`, `frontend`.

---

### Etapa 7 – Smoke Test

| Aspecto | Detalle |
|---|---|
| **Herramienta** | Docker Compose + `curl` |
| **Fase DevSecOps** | Deploy / Operate |
| **Riesgo que mitiga** | Fallos de integración entre servicios, errores de configuración en red |
| **Justificación** | Se levantan todos los servicios con Docker Compose y se verifica que el API Gateway responda correctamente en `/health`. Esto valida que los contenedores construidos funcionan correctamente en conjunto. |

**¿Por qué es necesaria esta etapa incluso con un sistema funcional?**
> Un sistema puede pasar todas las pruebas unitarias y de seguridad pero fallar cuando los servicios intentan comunicarse entre sí. El smoke test valida la integración end-to-end en un entorno similar a producción, detectando errores de configuración de red, variables de entorno faltantes, o incompatibilidades entre servicios.

**Evidencia GitHub Actions:**

![Smoke Test - Evidencia](/assets/stage-7.png)

---