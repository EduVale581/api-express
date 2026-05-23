# Taller: CRUD con Node.js + JWT + Base de Datos

> Aprende paso a paso a construir una API REST completa con autenticación por tokens.

---

## Paso 0 — Introducción y arquitectura

### ¿Qué vamos a construir?

Una API REST de usuarios con autenticación JWT. El cliente se comunica con el servidor usando rutas y métodos HTTP:

```
1. POST /auth/register  → Crear cuenta
2. POST /auth/login     → Recibir TOKEN
3. GET  /users          → Ver usuarios   (requiere token)
4. POST /users          → Crear usuario  (requiere token)
5. PUT  /users/:id      → Editar usuario (requiere token)
6. DELETE /users/:id    → Borrar usuario (requiere token)
```

### Estructura de carpetas

```
api/
├── src/
│   ├── config/
│   │   └── db.js          # Conexión a BD
│   ├── middleware/
│   │   └── auth.js        # Validar token
│   ├── routes/
│   │   ├── auth.js        # Registro y login
│   │   └── users.js       # CRUD de usuarios
│   └── index.js           # Entrada principal
├── .env                   # Variables secretas
└── package.json
```

### Tecnologías que usaremos

| Paquete | Para qué sirve |
|---|---|
| `express` | Servidor HTTP |
| `jsonwebtoken` | Crear y verificar tokens JWT |
| `bcryptjs` | Encriptar contraseñas |
| `dotenv` | Leer variables de entorno |
| `better-sqlite3` | Base de datos (sin instalar servidor) |

---

## Paso 1 — Configurar el proyecto

### Inicializar el proyecto

```bash
mkdir api
cd api
pnpm init -y
```

> `pnpm init -y` crea el archivo `package.json` que registra todas las dependencias.

### Instalar dependencias

```bash
pnpm add express jsonwebtoken bcryptjs dotenv better-sqlite3
```

### Crear el archivo `.env`

```env
# archivo: .env  ← NUNCA subir a Git
PORT=3000
JWT_SECRET=mi_clave_secreta_muy_larga_2024
DB_PATH=./database.db
```

> ⚠️ Agrega `.env` a tu `.gitignore` para no compartir información sensible.

### Archivo principal: `src/index.js`

```js
const express = require('express');
const dotenv  = require('dotenv');

dotenv.config(); // Carga las variables del .env

const app = express();
app.use(express.json()); // Permite leer JSON en el body

// Rutas (las agregaremos en los siguientes pasos)
app.use('/auth',  require('./routes/auth'));
app.use('/users', require('./routes/users'));

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Servidor corriendo en puerto ${PORT}`);
});
```

> 💡 `express.json()` es esencial. Sin él, el servidor no puede leer el cuerpo de las peticiones POST/PUT.

---

## Paso 2 — Conexión a base de datos

### ¿Por qué SQLite?

SQLite guarda toda la base de datos en un solo archivo. Es perfecta para aprender porque no necesitas instalar un servidor de base de datos. En producción usarías MySQL o PostgreSQL.

### Archivo: `src/config/db.js`

```js
const Database = require('better-sqlite3');
const path     = require('path');

// Conectar (o crear) la base de datos
const db = new Database(
  path.resolve(process.env.DB_PATH)
);

// Crear tabla si no existe
db.exec(`
  CREATE TABLE IF NOT EXISTS users (
    id        INTEGER PRIMARY KEY AUTOINCREMENT,
    name      TEXT    NOT NULL,
    email     TEXT    NOT NULL UNIQUE,
    password  TEXT    NOT NULL,
    role      TEXT    DEFAULT 'user',
    createdAt TEXT    DEFAULT CURRENT_TIMESTAMP
  )
`);

module.exports = db; // Exportar para usar en otras partes
```

### Conceptos clave de la tabla

| Columna | Descripción |
|---|---|
| `id` | Número único, se incrementa automáticamente |
| `name` | Texto obligatorio (`NOT NULL`) |
| `email` | Texto único, no puede repetirse |
| `password` | Siempre guardado encriptado, nunca en texto plano |
| `role` | Valor `'user'` o `'admin'` (por defecto `'user'`) |

> 💡 `CREATE TABLE IF NOT EXISTS` crea la tabla solo si no existe, así no da error al reiniciar el servidor.

---

## Paso 3 — Registro y Login (generación de JWT)

### ¿Qué es un JWT?

Un JWT (JSON Web Token) es una cadena de texto que contiene información del usuario, firmada con una clave secreta. El servidor la entrega al hacer login, y el cliente la envía en cada petición para demostrar su identidad.

```
# Estructura de un JWT (tres partes separadas por puntos):
eyJhbGciOiJIUzI1NiJ9    ← Header  (algoritmo usado)
.eyJ1c2VySWQiOjF9        ← Payload (datos del usuario)
.SflKxwRJSMeKKF2QT4fwp  ← Firma   (verificación de integridad)
```

### Archivo: `src/routes/auth.js`

```js
const express = require('express');
const bcrypt  = require('bcryptjs');
const jwt     = require('jsonwebtoken');
const db      = require('../config/db');
const router  = express.Router();

// ─── REGISTRO ───────────────────────────────────────────────
router.post('/register', (req, res) => {
  const { name, email, password } = req.body;

  // Verificar campos obligatorios
  if (!name || !email || !password)
    return res.status(400).json({ error: 'Faltan datos' });

  // Encriptar la contraseña (10 = nivel de seguridad)
  const hash = bcrypt.hashSync(password, 10);

  try {
    db.prepare(
      'INSERT INTO users (name, email, password) VALUES (?, ?, ?)'
    ).run(name, email, hash);

    res.status(201).json({ message: 'Usuario creado ✓' });
  } catch (e) {
    res.status(409).json({ error: 'El email ya está registrado' });
  }
});

// ─── LOGIN ──────────────────────────────────────────────────
router.post('/login', (req, res) => {
  const { email, password } = req.body;

  // Buscar usuario en la BD
  const user = db.prepare(
    'SELECT * FROM users WHERE email = ?'
  ).get(email);

  if (!user)
    return res.status(401).json({ error: 'Credenciales inválidas' });

  // Comparar contraseña con el hash guardado
  const valid = bcrypt.compareSync(password, user.password);
  if (!valid)
    return res.status(401).json({ error: 'Credenciales inválidas' });

  // Generar el token JWT
  const token = jwt.sign(
    { userId: user.id, role: user.role }, // Datos a incluir
    process.env.JWT_SECRET,               // Clave secreta
    { expiresIn: '24h' }                  // Duración del token
  );

  res.json({ token, message: 'Login exitoso ✓' });
});

module.exports = router;
```

> ⚠️ Nunca guardes la contraseña en texto plano. Siempre usa `bcrypt` para encriptarla antes de insertarla en la base de datos.

---

## Paso 4 — Middleware de autenticación (validar token)

### ¿Qué es un middleware?

Un middleware es una función que se ejecuta **entre** que llega la petición y que la ruta la responde. Actúa como un guardia de seguridad.

```
Cliente → [Middleware: valida token] → Ruta → Respuesta
                     ↓ si no tiene token válido
                  Error 401 (No autorizado)
```

### Archivo: `src/middleware/auth.js`

```js
const jwt = require('jsonwebtoken');

// Recibe (req, res, next)
// "next" = continuar al siguiente paso si todo está bien
function verifyToken(req, res, next) {

  // El token viene en el header así:
  // Authorization: Bearer eyJhbGci...
  const authHeader = req.headers['authorization'];

  if (!authHeader || !authHeader.startsWith('Bearer '))
    return res.status(401).json({ error: 'Token requerido' });

  // Extraer solo el token (quitar "Bearer ")
  const token = authHeader.split(' ')[1];

  try {
    // Verificar y decodificar el token
    const decoded = jwt.verify(token, process.env.JWT_SECRET);

    // Guardar los datos del usuario en req.user
    req.user = decoded; // → { userId, role }

    next(); // ✓ Token válido, continuar
  } catch (e) {
    res.status(401).json({ error: 'Token inválido o expirado' });
  }
}

module.exports = verifyToken;
```

### Cómo usar el middleware en las rutas

```js
const verifyToken = require('../middleware/auth');

// Sin protección — cualquiera puede acceder:
router.get('/publico', (req, res) => { ... });

// Con protección — solo usuarios con token válido:
router.get('/privado', verifyToken, (req, res) => {
  // req.user ya está disponible aquí
  res.json({ userId: req.user.userId });
});
```

> 💡 Al guardar los datos en `req.user`, las rutas pueden saber quién es el usuario sin necesidad de buscar en la base de datos de nuevo.

---

## Paso 5 — CRUD de Usuarios

### Archivo: `src/routes/users.js`

```js
const express     = require('express');
const db          = require('../config/db');
const verifyToken = require('../middleware/auth');
const router      = express.Router();

// Aplicar el middleware a TODAS las rutas de este archivo
router.use(verifyToken);

// ─── GET /users → Listar todos ──────────────────────────────
router.get('/', (req, res) => {
  const users = db.prepare(
    'SELECT id, name, email, role, createdAt FROM users'
  ).all(); // .all() devuelve un array
  res.json(users);
});

// ─── GET /users/:id → Ver uno ───────────────────────────────
router.get('/:id', (req, res) => {
  const user = db.prepare(
    'SELECT id, name, email, role FROM users WHERE id = ?'
  ).get(req.params.id); // .get() devuelve uno o undefined

  if (!user) return res.status(404).json({ error: 'No encontrado' });
  res.json(user);
});

// ─── POST /users → Crear ────────────────────────────────────
router.post('/', (req, res) => {
  const { name, email, role } = req.body;
  if (!name || !email)
    return res.status(400).json({ error: 'Nombre y email requeridos' });

  const result = db.prepare(
    'INSERT INTO users (name, email, password, role) VALUES (?, ?, ?, ?)'
  ).run(name, email, 'temporal', role || 'user');

  res.status(201).json({ id: result.lastInsertRowid, name, email });
});

// ─── PUT /users/:id → Editar ────────────────────────────────
router.put('/:id', (req, res) => {
  const { name, email } = req.body;
  const { changes } = db.prepare(
    'UPDATE users SET name = ?, email = ? WHERE id = ?'
  ).run(name, email, req.params.id);

  if (changes === 0)
    return res.status(404).json({ error: 'Usuario no encontrado' });

  res.json({ message: 'Actualizado ✓' });
});

// ─── DELETE /users/:id → Eliminar ───────────────────────────
router.delete('/:id', (req, res) => {
  const { changes } = db.prepare(
    'DELETE FROM users WHERE id = ?'
  ).run(req.params.id);

  if (changes === 0)
    return res.status(404).json({ error: 'Usuario no encontrado' });

  res.json({ message: 'Eliminado ✓' });
});

module.exports = router;
```

> 💡 Nunca devuelvas la columna `password` en las respuestas. Usa `SELECT` especificando solo las columnas que necesitas.

---

## Paso 6 — Pruebas con Postman / Thunder Client

### Iniciar el servidor

```bash
node src/index.js
# Deberías ver: Servidor corriendo en puerto 3000
```

### Prueba 1: Registrar un usuario

```
Método: POST
URL:    http://localhost:3000/auth/register
Body (JSON):
{
  "name":     "Ana López",
  "email":    "ana@correo.com",
  "password": "secreta123"
}

Respuesta esperada (201):
{ "message": "Usuario creado ✓" }
```

### Prueba 2: Hacer login y obtener el token

```
Método: POST
URL:    http://localhost:3000/auth/login
Body (JSON):
{
  "email":    "ana@correo.com",
  "password": "secreta123"
}

Respuesta (200):
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "message": "Login exitoso ✓"
}
```

> Copia el token. Lo necesitarás en todas las pruebas siguientes.

### Prueba 3: Listar usuarios (con token)

```
Método: GET
URL:    http://localhost:3000/users
Header: Authorization: Bearer eyJhbGciOiJIUzI1NiJ9...
                                ↑ "Bearer " + tu token

Respuesta (200):
[
  { "id": 1, "name": "Ana López", "email": "ana@correo.com", "role": "user" }
]
```

> ⚠️ Sin el header `Authorization`, la respuesta será `401: Token requerido`.

### Prueba 4: Editar un usuario

```
Método: PUT
URL:    http://localhost:3000/users/1
Header: Authorization: Bearer <tu_token>
Body (JSON):
{
  "name":  "Ana García",
  "email": "ana.garcia@correo.com"
}

Respuesta (200):
{ "message": "Actualizado ✓" }
```

### Prueba 5: Eliminar un usuario

```
Método: DELETE
URL:    http://localhost:3000/users/1
Header: Authorization: Bearer <tu_token>

Respuesta (200):
{ "message": "Eliminado ✓" }
```

---

## Paso 7 — Resumen y próximos pasos

### Lo que construiste

- ✅ Servidor Express con rutas organizadas en carpetas
- ✅ Base de datos SQLite con tabla de usuarios
- ✅ Registro con contraseña encriptada con bcrypt
- ✅ Login que devuelve un JWT firmado con clave secreta
- ✅ Middleware que valida el JWT en cada petición protegida
- ✅ CRUD completo: GET, POST, PUT, DELETE

### Mejoras para proyectos reales

```bash
# 1. Validación de datos
npm install joi

# 2. MySQL en lugar de SQLite
npm install mysql2 sequelize

# 3. Recarga automática al guardar (desarrollo)
npm install -D nodemon
# En package.json → "dev": "nodemon src/index.js"

# 4. Logging de peticiones
npm install morgan

# 5. Manejo de errores global
# Agregar al final de index.js:
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Error interno del servidor' });
});
```

### 🎯 Reto extra para practicar

Agrega una ruta `POST /auth/logout` que invalide el token usando una lista negra en memoria. Pista: guarda los tokens invalidados en un `Set` global y verifica en el middleware si el token recibido está en ese `Set`.

```js
// Pista de implementación:
const blacklist = new Set();

// En la ruta de logout:
router.post('/logout', verifyToken, (req, res) => {
  const token = req.headers['authorization'].split(' ')[1];
  blacklist.add(token);
  res.json({ message: 'Sesión cerrada ✓' });
});

// En el middleware auth.js, agregar esta verificación:
if (blacklist.has(token))
  return res.status(401).json({ error: 'Token invalidado' });
```

---
