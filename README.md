# Local SEO AI · Digital CRAO

Plataforma full-stack de **SEO local y optimización del Perfil de Negocio de Google** para agencias, consultores
SEO y dueños de negocios de **Perú y Latinoamérica**. Interfaz 100 % en español (es-PE), moneda **PEN (S/)** y
zona horaria **America/Lima**.

> **Transparencia de datos.** La plataforma **no está conectada a la API de Google Business Profile** y no inventa
> métricas de Google. Todo lo que se audita (información del perfil, fotos, reseñas, posiciones de palabras clave)
> es **información que el consultor registra manualmente**. Los datos de demostración son ficticios, se guardan con
> `is_demo = true` y se muestran siempre con la etiqueta **DEMO**.

---

## Funcionalidades

| Módulo | Descripción |
| --- | --- |
| **Autenticación** | Registro e inicio de sesión (scrypt + sesiones con cookie httpOnly), cuenta demo opcional, límite de intentos. |
| **Negocios** | CRUD completo: datos generales, categorías, servicios, contacto, ubicación y zonas de atención, horarios, datos del perfil de Google, honorarios en PEN, estado (activo/pausado/archivado). |
| **Auditoría GBP** | Motor de reglas de 100 puntos en 10 áreas (información, categorías, descripción, servicios, contacto, horarios, fotos, reseñas, publicaciones, palabras clave). Historial, comparación contra la auditoría anterior, notas y eliminación. |
| **SEO local** | Palabras clave (CRUD) con zona, intención y posición registrada; índice de visibilidad calculado solo con posiciones registradas; cobertura geográfica; oportunidades; sugerencias de palabras clave. |
| **Recomendaciones** | Plan de acción tipo tablero (pendiente / en progreso / completada) generado desde auditorías, SEO local y reseñas (+ IA opcional). CRUD de tareas. |
| **Reseñas** | Registro, análisis de sentimiento y temas (léxico en español), estadísticas, respuestas sugeridas (plantillas o IA), estados y filtros. |
| **Contenido** | Publicaciones de Google Business (novedad, oferta, evento, producto), descripciones (≤750), servicios (≤300), redes sociales (Facebook, Instagram, TikTok, WhatsApp). Biblioteca editable. |
| **Informes** | Informe profesional con puntaje, hallazgos, fortalezas, recomendaciones priorizadas, SEO local, reputación y plan 30-60-90. Imprimible / PDF desde el navegador. |
| **Panel** | KPIs de la cartera, puntaje promedio, evolución, estado por negocio, tareas prioritarias (con actualización optimista), reseñas pendientes. |
| **IA opcional** | OpenAI, Anthropic, Gemini o servicios compatibles con OpenAI. API key cifrada con AES-256-GCM. Botón de prueba de conexión. |

UX: estados vacíos, esqueletos de carga (`loading.tsx`), notificaciones, confirmaciones de borrado, **actualizaciones
optimistas** (`useOptimistic`), diseño responsive con barra lateral y tema verde / verde oscuro inspirado en Digital CRAO.

---

## Requisitos

- Node.js 20.9 o superior (probado con 22)
- PostgreSQL 14 o superior

## Puesta en marcha local

```bash
# 1. Instalar dependencias
npm install

# 2. Variables de entorno
cp .env.example .env        # edita DATABASE_URL y APP_SECRET

# 3. Crear las tablas (Drizzle ORM)
npx drizzle-kit push

# 4. Ejecutar en desarrollo
npm run dev                 # http://localhost:3000
```

La **cuenta de demostración** se crea automáticamente la primera vez que pulsas *«Entrar con la cuenta demo»* en la
pantalla de acceso (`demo@digitalcrao.com` / `Demo2026!`) con 4 negocios ficticios, auditorías históricas, palabras clave,
reseñas, tareas, contenido e informes. También puedes registrarte y marcar *«Cargar datos de demostración»*, o cargarlos /
eliminarlos luego desde **Configuración**.

### Base de datos

- Esquema en `src/db/schema.ts` (Drizzle ORM, `drizzle-orm/pg-core`).
- Conexión en `src/db/index.ts` (lee `DATABASE_URL`).
- Aplicar cambios de esquema: `npx drizzle-kit push`.
- Nota: `drizzle.config.json` contiene la URL local por defecto. En otros entornos puedes pasar la URL por CLI:
  `npx drizzle-kit push --dialect postgresql --schema ./src/db/schema.ts --url "$DATABASE_URL"` (renombrando temporalmente el
  archivo de configuración) o actualizar la URL del archivo.
- Con SQL directo: `psql "$DATABASE_URL" -c "select count(*) from businesses"`.

### Variables de entorno

Consulta `.env.example`. Resumen:

| Variable | Obligatoria | Uso |
| --- | --- | --- |
| `DATABASE_URL` | Sí | Cadena de conexión de PostgreSQL. |
| `APP_SECRET` | Recomendada | Deriva la clave AES-256-GCM para cifrar API keys. Si falta, se usa un valor derivado de `DATABASE_URL`. |
| `APP_URL` | Recomendada | Si empieza con `https://`, las cookies se emiten como `Secure`. |
| `COOKIE_SECURE` | No | `true`/`false` para forzar el atributo `Secure`. |
| `DEMO_ACCOUNT_ENABLED` | No | `false` deshabilita la cuenta demo (recomendado en producción). |
| `AI_PROVIDER`, `AI_API_KEY`, `AI_MODEL`, `AI_BASE_URL` | No | Configuración de IA por entorno (alternativa a la pantalla de Configuración). |
| `AI_ALLOW_PRIVATE_URLS` | No | Permite URLs base privadas para IA compatible (desactivado por defecto: protección SSRF). |
| `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` | No | Reservadas. **La integración con Google Business Profile API no está implementada.** |

---

## Arquitectura

```
src/
├── actions/          Server Actions por dominio (auth, negocios, auditorías, keywords, reseñas, tareas, contenido, informes, ajustes)
├── app/
│   ├── (auth)/       /login, /registro
│   ├── (app)/        /panel, /negocios, /auditorias, /seo-local, /recomendaciones, /resenas, /contenido, /informes, /configuracion
│   └── api/health    Verificación de salud (BD)
├── components/       UI reutilizable (ui/), shell con sidebar y componentes por módulo
├── db/               Esquema Drizzle y cliente pg
├── lib/              Motores puros y sin I/O: audit-engine, seo-engine, review-engine, content-engine, report-engine
│                     + auth, crypto, ai, formato es-PE/PEN, constantes
├── server/           Consultas (queries.ts), servicios compartidos y datos demo
└── proxy.ts          Redirección optimista a /login (Next.js 16)
```

Principios:

- **Motores puros** (`src/lib/*-engine.ts`): reciben datos y devuelven resultados; son fáciles de probar y de extender
  (por ejemplo, para añadir nuevas reglas de auditoría basta con agregar un `make(...)` en `runAudit`).
- **Acceso a datos aislado por usuario**: todas las tablas llevan `user_id` y cada Server Action valida `userId` + UUID.
- **Validación con Zod** en el servidor; los errores se devuelven en español.
- **Sin dependencias de IA obligatorias**: si la IA falla o no está configurada, se usan plantillas/reglas.

### Seguridad

- Contraseñas con `scrypt` + sal; comparación en tiempo constante.
- Sesiones: token aleatorio de 256 bits; en BD solo se guarda su hash SHA-256; cookie `httpOnly`, `sameSite=lax`, `secure` configurable.
- Límite básico de intentos de inicio de sesión (en memoria; para varios servidores usa Redis o un proxy con rate-limit).
- API keys de IA cifradas (AES-256-GCM) y nunca devueltas al navegador completas.
- Protección SSRF para la URL base de IA (solo `https` y sin redes privadas).
- Cabeceras de seguridad en `next.config.ts`.

---

## Despliegue

### Opción A · Docker Compose (servidor propio / VPS)

```bash
cp .env.example .env     # define APP_SECRET y APP_URL
docker compose up -d --build
docker compose exec app npx drizzle-kit push
```

La aplicación queda en `http://localhost:3000`. Coloca un proxy inverso (Caddy, Nginx o Traefik) con HTTPS y define
`APP_URL=https://tu-dominio.pe`.

### Opción B · Vercel + PostgreSQL gestionado (Neon, Supabase, Railway)

1. Crea la base de datos y copia su `DATABASE_URL` (con `?sslmode=require` si el proveedor lo exige).
2. Importa el repositorio en Vercel y define las variables de entorno (`DATABASE_URL`, `APP_SECRET`, `APP_URL`, `DEMO_ACCOUNT_ENABLED=false`).
3. Desde tu máquina aplica el esquema: `DATABASE_URL="…" npx drizzle-kit push`.
4. Despliega. Verifica `https://tu-dominio/api/health`.

### Opción C · Node.js directo

```bash
npm ci
npm run build
npx drizzle-kit push
npm run start            # PORT=3000 por defecto
```

Administra el proceso con systemd o PM2 y sirve detrás de HTTPS.

### Lista de verificación para producción

- [ ] `APP_SECRET` largo y aleatorio (guárdalo en un gestor de secretos).
- [ ] `APP_URL` con `https://` para cookies `Secure`.
- [ ] `DEMO_ACCOUNT_ENABLED=false`.
- [ ] Copias de seguridad automáticas de PostgreSQL.
- [ ] HTTPS y, si hay varias instancias, rate-limit centralizado.

---

## Integración con IA (opcional)

En **Configuración → Inteligencia artificial** elige proveedor, modelo y pega tu API key (o usa variables de entorno).
Al activarla aparecen las opciones *«Con IA»* en respuestas a reseñas, generador de contenido y recomendaciones.
El texto generado por IA debe **revisarse siempre** antes de publicarse, y los prompts piden no inventar datos.

## Integración con Google Business Profile (futuro)

Una integración oficial requiere: proyecto de Google Cloud, *Business Profile API* aprobada, credenciales OAuth 2.0 y
consentimiento del propietario de cada perfil. Hasta entonces la plataforma funciona con datos registrados manualmente.

## Scripts

| Comando | Descripción |
| --- | --- |
| `npm run dev` | Servidor de desarrollo. |
| `npm run build` / `npm run start` | Compilación y servidor de producción. |
| `npx drizzle-kit push` | Aplica el esquema a PostgreSQL. |
| `npx tsc --noEmit` | Verificación de tipos. |

---

© Digital CRAO Marketing · https://www.digitalcrao.com/ · WhatsApp 950 652 537
