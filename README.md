# SIGET: Sistema Inteligente de Gestión y Evaluación de Tesis

## Plan de Arquitectura y Guía de Operación (Senior Academic & Full-Stack perspective)

**SIGET** es una plataforma web y móvil empresarial diseñada para automatizar la gestión, revisión y evaluación de avances de tesis universitarias. Mediante un core inteligente desacoplado operado en colas asíncronas (**BullMQ**), el sistema extrae, vectoriza y analiza avances de tesis (.docx y .pdf) contrastándolos contra un **Documento Patrón Institucional** y rúbricas académicas configurables.

---

## 1. Arquitectura del Monorepo

El proyecto está diseñado bajo un esquema de **Monorepo** administrado por **Turborepo** para garantizar la modularidad, velocidad de compilación y compartición eficiente de tipos:

```text
siget-monorepo/
├── apps/
│   ├── api/                    # NestJS Backend (Clean Architecture / Modular)
│   │   ├── prisma/
│   │   │   └── schema.prisma   # Esquema 3NF con pgvector
│   │   ├── src/
│   │   │   ├── main.ts         # Bootstrap y configuración de Swagger/OpenAPI
│   │   │   ├── app.module.ts
│   │   │   └── modules/
│   │   │       └── ai/         # Pipeline de Análisis IA, Extracción y Embeddings
│   │   └── Dockerfile
│   │
│   ├── web/                    # Next.js 15 Frontend (App Router, Tailwind CSS, shadcn/ui)
│   │   ├── src/
│   │   │   ├── app/            # Vistas interactivas de Estudiante, Asesor y Coordinador
│   │   │   └── globals.css     # Estilos premium con Glassmorphism y Dark-Mode
│   │   └── Dockerfile
│   │
│   └── mobile/                 # Expo React Native App (SDK 52+)
│       ├── src/
│       │   └── screens/        # Dashboard y listado de hallazgos del estudiante
│       └── app.json
│
├── docker-compose.yml          # Postgres (pgvector) + Redis + MinIO + Ollama
├── package.json
└── turbo.json
```

---

## 2. Variables de Entorno y Configuración

Cree un archivo `.env` en la raíz del monorepo (y enlaces simbólicos o copias en `apps/api/.env` y `apps/web/.env` si ejecuta de manera aislada sin Docker):

### Backend API (`apps/api/.env`)
```bash
# Servidores e infraestructura
PORT=4000
NODE_ENV=development
DATABASE_URL="postgresql://siget_user:siget_secure_pwd_2026@localhost:5432/siget_db?schema=public"
REDIS_URL="redis://localhost:6379"

# Proveedor de Almacenamiento (S3 Compatible)
MINIO_ENDPOINT="localhost"
MINIO_PORT=9000
MINIO_ACCESS_KEY="siget_admin"
MINIO_SECRET_KEY="minio_secure_pwd_2026"

# Inteligencia Artificial
# Reemplace con su clave API de OpenAI para activar GPT-4o
OPENAI_API_KEY="sk-proj-..." 
# Endpoint opcional para LLM local (Ollama)
OLLAMA_URL="http://localhost:11434"

# Citas Académicas (Polite Pool de CrossRef)
CROSSREF_MAILTO="soporte.academico@universidad.edu"

# Seguridad y Autenticación
JWT_SECRET="nestjs_passport_secure_jwt_secret_2026"
```

### Frontend Web (`apps/web/.env.local`)
```bash
NEXT_PUBLIC_API_URL="http://localhost:4000/api"
NEXTAUTH_URL="http://localhost:3000"
NEXTAUTH_SECRET="nextauth_secure_jwt_secret_2026_super_long"
```

---

## 3. Instrucciones de Instalación y Despliegue

### Paso 1: Levantar Infraestructura con Docker
En la raíz del proyecto, ejecute el comando para iniciar la base de datos PostgreSQL con `pgvector`, Redis para colas y MinIO para archivos:
```bash
docker-compose up -d
```

### Paso 2: Instalar Dependencias del Monorepo
Instale las dependencias de todos los proyectos (`apps/api`, `apps/web`, `apps/mobile`) de forma centralizada utilizando Yarn:
```bash
yarn install
```

### Paso 3: Ejecutar Migraciones de Base de Datos
Generar el cliente Prisma e inyectar el esquema relacional normalizado con soporte de vectores en PostgreSQL:
```bash
# Ejecutar migraciones
yarn db:migrate

# Generar cliente
yarn db:generate
```

### Paso 4: Iniciar Entorno de Desarrollo
Lance el servidor de desarrollo de Turborepo. Esto iniciará simultáneamente el backend en NextJS (`http://localhost:4000`) con Swagger en (`http://localhost:4000/docs`) y la web en Next.js (`http://localhost:3000`):
```bash
yarn dev
```

---

## 4. Prompts de Sistema para Evaluación Académica (GPT-4o)

El pipeline inteligente (`AIService`) ejecuta el siguiente prompt optimizado para estructurar de manera estricta el análisis científico y de cumplimiento de normas:

```text
Actúas como un Evaluador Académico e Inteligencia Artificial de Alto Nivel de Tesis de Posgrado y Maestría (SIGET).
Tu misión es evaluar el avance de tesis provisto contra el "Documento Patrón Institucional" titulado: "{patternTitle}".

Rúbrica de Criterios Generales de Calificación:
1. Estructura (30%): Presencia, orden y completitud de las secciones obligatorias.
2. Contenido (40%): Calidad conceptual, planteamiento del problema, hipótesis/preguntas de investigación claras, metodología rigurosa, y coherencia.
3. Forma (20%): Redacción académica, normas de estilo, citas bibliográficas (ej. APA/IEEE), y longitud.
4. Originalidad/Calidad (10%): Lenguaje formal y cohesión lógica interna del escrito.

Normas estructurales del patrón a validar:
{structureRules}

Debes responder ÚNICAMENTE con un objeto JSON válido que cumpla exactamente la estructura de TypeScript...
```

---

## 5. Propuesta de Mejoras Adicionales y Roadmap Técnico

### A. Calibración y RLHF (Fine-Tuning con Retroalimentación Humana)
* **Mapeo:** Las correcciones efectuadas por el Asesor en `AIFindingHumanComment` registran discrepancias entre la IA y el experto.
* **Proceso:** Al acumular **500+ registros**, un script automatizado consolida los pares `(texto_original, feedback_asesor)` en un dataset JSONL.
* **Ejecución:** Se dispara un job de Fine-Tuning a la API de OpenAI (`gpt-4o-mini`) para calibrar el modelo, logrando que el evaluador IA reduzca su brecha de criterio del 8% actual a menos del 2% en un semestre académico.

### B. Detección de Plagio Intra-Programa en `pgvector`
* **Mecanismo:** El avance se fragmenta en chunks semánticos de 1500 caracteres con solapamiento de 200.
* **Vectorización:** Cada chunk se transforma en un vector mediante `text-embedding-3-small` (1536 dimensiones).
* **Búsqueda:** Se realiza una búsqueda por similitud de coseno (`1 - (chunk.embedding <=> :input_embedding) > 0.85`) en base de datos PostgreSQL contra trabajos previos del mismo programa académico, alertando al revisor de correspondencias exactas no citadas.

### C. Validación Cruzada de Citas con CrossRef Polite Pool
* **Lookup:** El sistema extrae las referencias y realiza consultas a la API de CrossRef utilizando el encabezado `User-Agent` configurado con el email institucional (`Polite Pool`), lo que otorga acceso a recursos de alta tasa de peticiones.
* **Rate Limiting:** Se implementa una cola con algoritmo *token-bucket* que limita las llamadas a 1 petición por segundo para evitar baneos de IP, detectando referencias inexistentes o alteradas ("Alucinaciones Académicas").

### D. Vinculación ORCID OAuth 2.0 y Asignación Inteligente
* **Autorización:** Los asesores enlazan su ID digital ORCID mediante OAuth 2.0.
* **Expertise Semantic Check:** Se calcula la similitud vectorial entre los abstracts de las publicaciones indexadas del asesor y el título del avance del estudiante. Si la similitud coseno cae por debajo de **0.40**, el sistema alerta al Coordinador que el asesor asignado podría no ser el especialista idóneo para la temática investigada.
