# Lab-Química - Sistema de Gestión de Laboratorio

Sistema de gestión de turnos para laboratorio de química con simulación de experimentos en tiempo real. Permite a múltiples usuarios reservar turnos y ejecutar experimentos de forma ordenada mediante una cola FIFO.

## Descripción

**Lab-Química** es una aplicación web que gestiona el acceso a experimentos de laboratorio químico, asegurando que solo un experimento se ejecute a la vez. El sistema simula un experimento de temperatura con control de relé, transmitiendo datos en tiempo real mediante WebSockets.

### Características Principales

- **Sistema de Cola FIFO**: Gestión ordenada de turnos para experimentos
- **Autenticación Segura**: Sistema de registro y login con JWT y hashing de contraseñas
- **Tiempo Real**: Transmisión de datos de experimentos vía WebSockets (Socket.IO)
- **Simulación de Experimento**:
  - Sensor de temperatura con curva gaussiana
  - Control de relé simulado (GPIO 18)
  - Datos cada 500ms durante 15 segundos
- **Gestión de Estados**: Pipeline de estados de turno (pendiente → aceptado → finalizado)
- **Arquitectura Moderna**: Express.js + Sequelize + MySQL + Socket.IO

## Tecnologías Utilizadas

| Componente | Tecnología | Versión |
|-----------|-----------|---------|
| Runtime | Node.js | LTS Alpine |
| Framework Web | Express.js | 4.21.2 |
| ORM | Sequelize | 6.37.7 |
| Base de Datos | MySQL | 8.0 |
| WebSocket | Socket.IO | 4.8.1 |
| Autenticación | JWT | 9.0.2 |
| Hashing | bcryptjs | 3.0.2 |
| Dev Tool | Nodemon | 2.0.20 |
| Containerización | Docker Compose | - |

## Requisitos Previos

- [Node.js](https://nodejs.org/) (LTS)
- [Docker](https://www.docker.com/) y Docker Compose (recomendado)
- MySQL 8.0 (si no se usa Docker)

## Instalación

### Opción 1: Con Docker Compose (Recomendado)

```bash
# Clonar el repositorio
git clone <url-del-repositorio>
cd Lab-quimica

# Iniciar servicios con Docker Compose
docker-compose up
```

El servidor estará disponible en `http://localhost:3000`

### Opción 2: Instalación Local

```bash
# Clonar el repositorio
git clone <url-del-repositorio>
cd Lab-quimica

# Instalar dependencias
npm install

# Configurar variables de entorno (ver sección Configuración)
cp .env.example .env
# Editar .env con tus credenciales

# Iniciar servidor de desarrollo
npm run dev

# O iniciar en producción
npm start
```

## Configuración

### Variables de Entorno

Crear un archivo `.env` en la raíz del proyecto:

```env
# Base de datos
MYSQL_HOST=localhost
MYSQL_USER=root
MYSQL_PASSWORD=secret
MYSQL_DB=lab

# JWT
JWT_SECRET=tu-secreto-super-seguro-aqui
JWT_EXPIRES=24h

# Servidor
PORT=3000
```

### Inicialización de Base de Datos

El sistema crea automáticamente las tablas al iniciar. Para inicializar los estados de turno:

```bash
node init-estados.js
```

Esto crea los siguientes estados en la tabla `TurnoEstado`:
- pendiente
- aceptado
- rechazado
- finalizado
- cancelado

## Estructura del Proyecto

```
Lab-quimica/
├── src/
│   ├── app.js                          # Configuración de Express
│   ├── server.js                       # Servidor HTTP + Socket.IO
│   ├── socket.handler.js               # Gestión de WebSockets
│   ├── initDB.js                       # Inicialización de BD
│   │
│   ├── config/
│   │   ├── database.js                 # Configuración Sequelize
│   │   ├── init-database.js            # Sincronización de modelos
│   │   └── init-estados.js             # Inicialización de estados
│   │
│   ├── models/
│   │   ├── index.js                    # Relaciones entre modelos
│   │   ├── user.model.js               # Modelo Usuario
│   │   ├── shift.model.js              # Modelo Turno
│   │   └── turnoEstado.model.js        # Modelo Estado de Turno
│   │
│   ├── controllers/
│   │   ├── auth.controller.js          # Login/Registro/Logout
│   │   ├── shift.controller.js         # Gestión de turnos
│   │   ├── startExperiment.controller.js # Control de experimentos
│   │   └── turnoEstado.controller.js   # CRUD de estados
│   │
│   ├── routes/
│   │   ├── auth.routes.js              # Rutas de autenticación
│   │   ├── shift.routes.js             # Rutas de turnos
│   │   ├── experiment.routes.js        # Rutas de experimentos
│   │   └── turnoEstado.routes.js       # Rutas de estados
│   │
│   ├── services/
│   │   ├── auth.service.js             # Lógica de autenticación
│   │   └── startExperiment.service.js  # Lógica del experimento (CORE)
│   │
│   └── middlewares/
│       ├── auth.middleware.js          # Verificación JWT
│       └── shift.middleware.js         # Validación de turno actual
│
├── docs/
│   └── SEQUELIZE_MIGRATION.md          # Documentación de migración
│
├── index.html                          # Frontend (interfaz web)
├── package.json                        # Dependencias y scripts
├── compose.yaml                        # Configuración Docker
├── init-estados.js                     # Script de inicialización
└── README.md                           # Este archivo
```

## API Endpoints

### Autenticación (`/auth`)

| Método | Ruta | Autenticación | Descripción |
|--------|------|---------------|-------------|
| POST | `/auth/register` | No | Registrar nuevo usuario |
| POST | `/auth/login` | No | Iniciar sesión (retorna JWT) |
| POST | `/auth/logout` | No | Cerrar sesión |

**Ejemplo - Registro:**
```bash
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email": "usuario@ejemplo.com", "password": "miPassword123"}'
```

**Ejemplo - Login:**
```bash
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "usuario@ejemplo.com", "password": "miPassword123"}'
```

### Turnos (`/shift`)

| Método | Ruta | Autenticación | Descripción |
|--------|------|---------------|-------------|
| POST | `/shift/create` | Requerida | Crear nuevo turno |
| GET | `/shift/shifts` | No | Obtener todos los turnos |
| POST | `/shift/shift` | No | Obtener turnos del usuario |

**Ejemplo - Crear Turno:**
```bash
curl -X POST http://localhost:3000/shift/create \
  -H "Content-Type: application/json" \
  -H "Cookie: token=TU_JWT_TOKEN" \
  -d '{"socketId": "socket-id", "userId": 1}'
```

### Experimentos (`/exp`)

| Método | Ruta | Autenticación | Descripción |
|--------|------|---------------|-------------|
| POST | `/exp/start` | Requerida | Iniciar experimento (requiere turno actual) |

**Ejemplo - Iniciar Experimento:**
```bash
curl -X POST http://localhost:3000/exp/start \
  -H "Content-Type: application/json" \
  -H "Cookie: token=TU_JWT_TOKEN" \
  -d '{"shiftId": 1}'
```

### Estados de Turno (`/turno-estado`)

| Método | Ruta | Autenticación | Descripción |
|--------|------|---------------|-------------|
| GET | `/turno-estado/estados` | No | Obtener todos los estados |
| GET | `/turno-estado/estados/:id` | No | Obtener estado por ID |
| GET | `/turno-estado/estados/nombre/:nombre` | No | Obtener estado por nombre |
| POST | `/turno-estado/estados` | Requerida | Crear nuevo estado |
| PUT | `/turno-estado/estados/:id` | Requerida | Actualizar estado |
| DELETE | `/turno-estado/estados/:id` | Requerida | Eliminar estado |

## WebSockets

### Eventos Emitidos por el Servidor

| Evento | Destinatario | Datos | Descripción |
|--------|--------------|-------|-------------|
| `hello` | Cliente específico | `{ socketId }` | Confirmación de conexión |
| `turno:ofrecido` | Cliente específico | `{ turnoId, tiempoRespuesta: 30 }` | Oferta de turno (30s para aceptar) |
| `exp:data` | Todos los clientes | `{ timestamp, temperatura }` | Datos del experimento en tiempo real |

### Conexión con Socket.IO (Cliente)

```javascript
// Conectar al servidor WebSocket
const socket = io('http://localhost:3000', {
  withCredentials: true
});

// Escuchar oferta de turno
socket.on('turno:ofrecido', (data) => {
  console.log(`Turno ${data.turnoId} ofrecido. Tiempo: ${data.tiempoRespuesta}s`);
  // Usuario debe aceptar dentro de 30 segundos
});

// Escuchar datos del experimento
socket.on('exp:data', (data) => {
  console.log(`Temperatura: ${data.temperatura}°C - Timestamp: ${data.timestamp}`);
});

// Escuchar confirmación de conexión
socket.on('hello', (data) => {
  console.log('Conectado con socket ID:', data.socketId);
});
```

## Flujo de Funcionamiento

### 1. Registro e Inicio de Sesión

```
Usuario → POST /auth/register → Backend hashea password → Guarda en BD
Usuario → POST /auth/login → Backend verifica credenciales → Retorna JWT en cookie httpOnly
```

### 2. Creación de Turno

```
Usuario → Conecta WebSocket → Recibe socketId
Usuario → POST /shift/create → Backend:
  1. Verifica que no exista turno pendiente activo
  2. Crea turno con estado "pendiente"
  3. Guarda socketId en socketList
  4. Emite evento "hello" vía WebSocket
  5. Inicia procesamiento de cola FIFO
```

### 3. Procesamiento de Cola FIFO

```
Sistema busca primer turno pendiente (order by created_at ASC)
  ├─ Si usuario CONECTADO:
  │    ├─ Emite "turno:ofrecido" vía WebSocket
  │    ├─ Espera 30 segundos para aceptación
  │    ├─ Si ACEPTA → Usuario puede iniciar experimento
  │    └─ Si NO ACEPTA → Cambia estado a "rechazado", continúa siguiente
  │
  └─ Si usuario DESCONECTADO:
       └─ Cambia estado a "rechazado", continúa siguiente
```

### 4. Ejecución del Experimento

```
Usuario → POST /exp/start con shiftId
  ├─ authMiddleware valida JWT
  ├─ shiftMiddleware valida que sea turno actual
  └─ Backend:
      1. Cambia estado del turno a "aceptado"
      2. Inicia ciclo de simulación (30 pasos × 500ms = 15 segundos)
      3. Cada 500ms:
         ├─ Calcula temperatura con función gaussiana (máx 60°C)
         ├─ Controla relé:
         │   ├─ Enciende si temp ≤ 30°C
         │   └─ Apaga si temp ≥ 46°C
         └─ Emite "exp:data" a todos los clientes conectados
      4. Condición de finalización:
         └─ temp ≤ 30°C Y step >= 30
      5. Cambia estado del turno a "finalizado"
      6. Procesa siguiente turno en cola
```

### 5. Diagrama de Estados de Turno

```
        ┌─────────────┐
        │  PENDIENTE  │ ← Turno creado
        └──────┬──────┘
               │
        ┌──────▼──────────────────┐
        │  Procesador FIFO        │
        │  (Espera 30s)           │
        └──┬────────────┬──────┬──┘
           │            │      │
    ┌──────▼──────┐    │      │
    │  ACEPTADO   │◄───┘      │
    │  (Inicia    │           │
    │  experimento)│          │
    └──────┬──────┘           │
           │                  │
    ┌──────▼──────┐    ┌──────▼──────┐
    │ FINALIZADO  │    │  RECHAZADO  │
    └─────────────┘    └─────────────┘
```

## Base de Datos

### Modelos y Relaciones

```
┌─────────────────┐         ┌─────────────────┐         ┌──────────────────┐
│      User       │         │      Shift      │         │   TurnoEstado    │
├─────────────────┤         ├─────────────────┤         ├──────────────────┤
│ id (PK)         │────┐    │ id (PK)         │    ┌────│ id (PK)          │
│ email (unique)  │    │    │ id_usuario (FK) │────┤    │ nombreEstado     │
│ password (hash) │    └───>│ id_estado (FK)  │    │    │   - pendiente    │
│ created_at      │         │ created_at      │    │    │   - aceptado     │
│ updated_at      │         │ updated_at      │    │    │   - rechazado    │
└─────────────────┘         └─────────────────┘    │    │   - finalizado   │
                                                    │    │   - cancelado    │
                                                    └────│ created_at       │
                                                         │ updated_at       │
                                                         └──────────────────┘

Relaciones:
• User (1) ──< Shift (N)          → Un usuario puede tener múltiples turnos
• Shift (N) >── TurnoEstado (1)   → Cada turno tiene un estado
```

### Script de Inicialización

```bash
# Crear estados predeterminados
node init-estados.js
```

Este script crea los 5 estados en la tabla `TurnoEstado`:
1. pendiente
2. aceptado
3. rechazado
4. finalizado
5. cancelado

## Simulación del Experimento

### Características de la Simulación

**Sensor de Temperatura:**
- Curva gaussiana con pico de 60°C en el paso 15
- Función: `gaussTemp(step, A=60, mu=15, sigma=5)`
- Transmite datos cada 500ms (30 pasos totales = 15 segundos)

**Control de Relé (GPIO 18 simulado):**
- **Enciende** cuando temperatura ≤ 30°C
- **Apaga** cuando temperatura ≥ 46°C
- Estado del relé se incluye en transmisión WebSocket

**Condiciones de Finalización:**
- Temperatura ≤ 30°C
- Step >= 30 (15 segundos transcurridos)

### Código Relevante

Archivo principal: `src/services/startExperiment.service.js`

```javascript
// Función gaussiana para simular temperatura
function gaussTemp(t, A = 60, mu = 15, sigma = 5) {
  const exponente = -Math.pow(t - mu, 2) / (2 * Math.pow(sigma, 2));
  return A * Math.exp(exponente);
}

// Control del relé
if (temperatura <= 30 && !releEncendido) {
  releEncendido = true;  // Encender
} else if (temperatura >= 46 && releEncendido) {
  releEncendido = false; // Apagar
}
```

## Uso del Sistema

### Desde la Interfaz Web

1. Abrir navegador en `http://localhost:3000`
2. Registrar usuario o iniciar sesión
3. Crear turno presionando el botón correspondiente
4. Esperar notificación de turno ofrecido (30s para aceptar)
5. Aceptar turno para iniciar experimento
6. Visualizar datos de temperatura en tiempo real
7. Esperar finalización del experimento

### Desde API REST

Consulta la sección [API Endpoints](#api-endpoints) para ejemplos con `curl`.

## Scripts Disponibles

```bash
# Iniciar servidor en producción
npm start

# Iniciar servidor en modo desarrollo (con nodemon)
npm run dev

# Inicializar estados en base de datos
node init-estados.js
```

## Seguridad

### Medidas Implementadas

- **Hashing de contraseñas**: bcryptjs con salt factor 10
- **JWT con cookies httpOnly**: Previene XSS
- **Middleware de autenticación**: Protege rutas sensibles
- **Middleware de turno**: Valida que sea el turno actual antes de iniciar experimento
- **CORS configurado**: Solo permite orígenes autorizados
- **Variables de entorno**: Credenciales fuera del código fuente

### Variables de Entorno Sensibles

```env
JWT_SECRET=cambiar-por-secreto-seguro-aleatorio
MYSQL_PASSWORD=cambiar-por-password-segura
```

**Recomendación**: Generar valores aleatorios fuertes y no compartirlos en control de versiones.

## Casos de Uso

### Laboratorio Educativo

Múltiples estudiantes desean realizar el mismo experimento:
- Cada estudiante registra una cuenta
- Solicitan turno a través de la interfaz
- El sistema procesa turnos de forma ordenada (FIFO)
- Solo un experimento se ejecuta a la vez
- Todos pueden ver datos en tiempo real vía WebSocket

### Demostración Remota

Instructor demuestra experimento a clase remota:
- Instructor inicia experimento
- Datos se transmiten en tiempo real a todos los clientes conectados
- Estudiantes visualizan curva de temperatura y estado del relé

### Investigación Colaborativa

Investigadores comparten acceso al laboratorio:
- Sistema de turnos evita conflictos de acceso
- Registro de experimentos con timestamps
- Datos guardados para análisis posterior

## Desarrollo

### Contribuir al Proyecto

1. Fork del repositorio
2. Crear rama de feature: `git checkout -b feature/nueva-funcionalidad`
3. Commit de cambios: `git commit -m 'Añade nueva funcionalidad'`
4. Push a la rama: `git push origin feature/nueva-funcionalidad`
5. Crear Pull Request

### Convenciones de Código

- **Estilo**: ESM (ECMAScript Modules)
- **Nomenclatura**: camelCase para variables y funciones
- **Controladores**: Separación clara entre controller y service
- **Rutas**: RESTful con verbos HTTP apropiados

## Solución de Problemas

### El servidor no inicia

```bash
# Verificar que MySQL esté corriendo
docker-compose ps

# Revisar logs
docker-compose logs app
docker-compose logs mysql
```

### Error de conexión a base de datos

Verificar variables de entorno en `.env` o `compose.yaml`:
```env
MYSQL_HOST=mysql  # Para Docker: "mysql", para local: "localhost"
MYSQL_USER=root
MYSQL_PASSWORD=secret
MYSQL_DB=lab
```

### WebSockets no conectan

Verificar configuración de CORS en `src/server.js`:
```javascript
const io = new Server(server, {
  cors: {
    origin: "http://localhost:3000",
    credentials: true
  }
});
```

### Turno no se ofrece

Posibles causas:
- Usuario desconectado del WebSocket
- No es el primer turno en la cola (verificar otros turnos pendientes)
- Error en procesador FIFO (revisar logs del servidor)

## Roadmap

- [ ] Dashboard de administración
- [ ] Historial de experimentos por usuario
- [ ] Exportación de datos a CSV/JSON
- [ ] Soporte para múltiples tipos de experimentos
- [ ] Notificaciones push
- [ ] Integración con hardware real (Raspberry Pi + sensores)
- [ ] Gráficos en tiempo real con Chart.js
- [ ] Sistema de permisos y roles

## Licencia

Este proyecto está bajo la licencia MIT. Consulta el archivo `LICENSE` para más detalles.

## Contacto y Soporte

Para reportar bugs o solicitar features, abre un issue en el repositorio.

---

**Desarrollado con Node.js, Express, Sequelize, Socket.IO y MySQL**
