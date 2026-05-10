# notara-frontend

Repositorio del frontend de **Notara**, plataforma de aprendizaje de inglés mediante música. Contiene la interfaz web construida con Next.js y un BFF (Backend For Frontend) en Spring Boot que agrega datos de los microservicios del backend.

---

## Arquitectura

```
notara-frontend/
├── frontend-pagina/      # Aplicación Next.js 14 (Puerto 3001)
│   ├── src/app/          # Páginas (login, register, search, lesson, dashboard)
│   ├── src/components/   # Componentes reutilizables (Navbar, SongCard, etc.)
│   ├── src/context/      # AuthContext con manejo de sesión JWT
│   ├── src/lib/api.js    # Cliente HTTP centralizado para el API Gateway
│   └── src/patterns/     # Patrón Strategy para visualización de letra
├── bff/                  # Backend For Frontend en Spring Boot (Puerto 8080)
│   └── src/              # Agrega datos de ms-usuarios y ms-notas-metas
└── docker-compose.yml    # Stack frontend completo
```

### Patrón Strategy en la visualización de letra

La letra de cada canción puede visualizarse en cuatro modos intercambiables, implementados como estrategias (`src/patterns/LyricsDisplayStrategy.js`):

| ID | Modo |
|---|---|
| `en-only` | Solo letra en inglés |
| `es-only` | Solo traducción al español |
| `bilingual` | Inglés y español en columnas paralelas |
| `synced` | Sincronizada con el tiempo de reproducción |

Agregar un nuevo modo solo requiere añadir un nuevo objeto estrategia sin modificar el componente principal (`LessonPage`).

---

## Tecnologías

| Capa | Tecnología |
|---|---|
| Frontend | Next.js 14, React 18, Tailwind CSS, Recharts |
| BFF | Spring Boot 4.0, Spring Cloud OpenFeign |
| Contenedores | Docker, Docker Compose |

---

## Variables de entorno

### Frontend (frontend-pagina)

Crea el archivo `frontend-pagina/.env.local` para desarrollo local:

```env
NEXT_PUBLIC_API_URL=http://localhost:3000
```

En producción (EC2), esta variable se inyecta como `build-arg` durante el build de Docker:

```bash
docker build --build-arg NEXT_PUBLIC_API_URL=http://<IP_PUBLICA_EC2>:8080 .
```

### BFF

Las URLs de los microservicios se configuran como variables de entorno del contenedor:

```env
MS_NOTAS_METAS_URL=http://<IP_PRIVADA_BACKEND>:8083
MS_USUARIOS_URL=http://<IP_PRIVADA_BACKEND>:8081
EUREKA_URL=http://<IP_PRIVADA_BACKEND>:8761/eureka
```

---

## Levantar el stack localmente

### Requisitos previos

- Docker Desktop instalado y corriendo
- Backend corriendo (ver `notara-backend`)

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/elpanxo/notara-frontend.git
cd notara-frontend

# 2. Crear archivo .env con la IP del backend
echo "BACKEND_EC2_IP=localhost" > .env
echo "FRONTEND_PUBLIC_IP=localhost" >> .env

# 3. Construir y levantar
docker compose up --build

# 4. Acceder a la aplicación
# Frontend: http://localhost:3001
# BFF:      http://localhost:8080
```

### Desarrollo sin Docker

```bash
cd frontend-pagina
npm install
npm run dev     # disponible en http://localhost:3001
```

---

## Páginas de la aplicación

| Ruta | Descripción |
|---|---|
| `/` | Redirección automática según sesión |
| `/login` | Inicio de sesión con JWT |
| `/register` | Registro de nuevo usuario |
| `/search` | Búsqueda de canciones via Spotify |
| `/lesson/[songId]` | Lección interactiva con letra sincronizada |
| `/dashboard` | Progreso del estudiante con gráficos |

---

## Puertos expuestos

| Servicio | Puerto local |
|---|---|
| frontend-pagina (Next.js) | `3001` |
| bff (Spring Boot) | `8080` |

---

## Pipeline CI/CD

El repositorio cuenta con un pipeline en **GitHub Actions** (`.github/workflows/deploy-frontend.yml`) que se activa al hacer push sobre la rama `deploy`.

### Flujo del pipeline

```
Push a rama deploy
       │
       ▼
  Checkout código
       │
       ▼
  Configurar credenciales AWS
       │
       ▼
  Login a Amazon ECR
       │
       ▼
  Build imagen frontend (Next.js)
  con NEXT_PUBLIC_API_URL como build-arg
  Push a ECR
       │
       ▼
  Build imagen BFF (Spring Boot)
  Push a ECR
       │
       ▼
  Deploy en EC2 via AWS SSM
  (git pull + docker compose pull + up)
```

### Secrets requeridos en GitHub

Configurar en `Settings > Secrets and variables > Actions`:

```
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
AWS_SESSION_TOKEN
AWS_REGION
ECR_REGISTRY              # URL del registry ECR (ej: 123456789.dkr.ecr.us-east-1.amazonaws.com)
EC2_FRONTEND_INSTANCE_ID  # ID de la instancia EC2 del frontend
FRONTEND_PUBLIC_IP        # IP pública de la instancia EC2 frontend
BACKEND_EC2_PRIVATE_IP    # IP privada de la instancia EC2 backend
```

---

## Dockerfiles

### frontend-pagina/Dockerfile

Construye la aplicación Next.js en modo producción. La variable `NEXT_PUBLIC_API_URL` se pasa como `build-arg` para que quede embebida en el bundle del cliente.

> **Nota de mejora**: Este Dockerfile no usa multi-stage build. Se puede optimizar separando la etapa de build (`node:20-alpine AS build` con `npm run build`) de la etapa de runtime para reducir el tamaño de imagen.

### bff/dockerfile

Compila el proyecto Maven y expone el JAR resultante. 

> **Nota de mejora**: Este Dockerfile tampoco usa multi-stage. La versión optimizada usaría `maven:3.9 AS build` para compilar y `eclipse-temurin:21-jre-alpine` para la imagen final, reduciendo el tamaño de ~800MB a ~200MB.

---

## Despliegue en AWS EC2

El frontend corre en una instancia EC2 con **IP pública**, siendo el único componente accesible desde Internet. El BFF se comunica con el backend (IP privada) a través de Security Groups internos.

```
Internet
   │
   ▼
[EC2 Frontend - IP pública :3001]
   │  BFF en mismo host puerto 8080
   ▼
[EC2 Backend - IP privada]
   (ms-usuarios :8081, ms-notas-metas :8083)
```

Solo el frontend es accesible desde Internet. El backend opera en subred privada.

---

## Usuario demo

La aplicación incluye un usuario de demostración para pruebas sin necesidad de tener el backend activo:

```
Email:    demo@notara.com
Password: demo1234
```
