# Node.js Crash Course — Zero to Interview-Ready

> Tailored for Delve tech screen. Django/C# background assumed.

---

## Level 0: Mental Model (Django/C# -> Node.js)

| Django / ASP.NET | Node.js |
|-----------------|---------|
| `manage.py runserver` | `node index.js` / `npm run dev` |
| `urls.py` + `views.py` | Express routes |
| `def my_view(request):` | `(req, res) => {}` |
| `request.GET['key']` | `req.query.key` |
| `request.POST['key']` | `req.body.key` |
| `request.user` | `req.user` (set by auth middleware) |
| Django middleware | Express middleware (same concept) |
| `JsonResponse({...})` | `res.json({...})` |
| `return HttpResponse(status=404)` | `res.status(404).send()` |
| Django ORM | Raw SQL / Prisma / Knex |
| `settings.py` | `.env` file + `process.env` |
| Virtual env | `node_modules` + `package.json` |

**The big difference:** Django is synchronous by default. Node is **async by default** — everything I/O-related (DB calls, HTTP requests, file reads) is non-blocking.

---

## Level 1: JavaScript Fundamentals for Node

### Variables
```js
const name = 'Aayush'    // can't be reassigned — use by default
let count = 0            // can be reassigned
// never use var — it has weird scoping rules
```

### Functions
```js
// Regular function
function greet(name) {
  return `Hello, ${name}`
}

// Arrow function (most common in modern Node)
const greet = (name) => `Hello, ${name}`

// Arrow with body
const greet = (name) => {
  const msg = `Hello, ${name}`
  return msg
}
```

### Destructuring (you'll see this everywhere)
```js
// Object destructuring
const user = { id: 1, name: 'Aayush', role: 'admin' }
const { id, name } = user          // id=1, name='Aayush'
const { id, ...rest } = user       // rest = { name, role }

// Array destructuring
const [first, second, ...others] = [1, 2, 3, 4]

// In function params (very common in Express)
app.get('/users/:id', async (req, res) => {
  const { id } = req.params        // instead of req.params.id
  const { page = 1, limit = 10 } = req.query  // with defaults
})
```

### Spread operator
```js
const defaults = { status: 'active', role: 'member' }
const newUser = { ...defaults, name: 'Aayush', role: 'admin' }
// { status: 'active', role: 'admin', name: 'Aayush' } — later keys win
```

### Modules
```js
// CommonJS (older, still common)
const express = require('express')
module.exports = { myFunction }

// ES Modules (modern — needs "type": "module" in package.json)
import express from 'express'
export const myFunction = () => {}
```

---

## Level 2: Async / Await (the most important concept)

### Why async matters in Node
Node is single-threaded. Instead of blocking while waiting for a DB call, it registers a callback and moves on. This is the **event loop**.

```
Request comes in
  -> Start DB query (non-blocking)
  -> Handle other requests while waiting
  -> DB responds
  -> Resume and send response
```

### Three ways to write async code (know all three)

**1. Callbacks (old — just recognize it)**
```js
fs.readFile('file.txt', 'utf8', (err, data) => {
  if (err) throw err
  console.log(data)
})
// Code here runs BEFORE the file is read — that's the async nature
```

**2. Promises**
```js
fetch('https://api.example.com/users')
  .then(res => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err))
  .finally(() => console.log('done'))

// Promise.all — run multiple in parallel
const [users, orders] = await Promise.all([
  fetchUsers(),
  fetchOrders()
])
```

**3. Async/Await (modern — use this)**
```js
async function getUser(id) {
  try {
    const res = await fetch(`/api/users/${id}`)
    const data = await res.json()
    return data
  } catch (err) {
    console.error('Failed:', err)
    throw err  // re-throw so caller knows it failed
  }
}
```

`await` can only be used inside an `async` function. It pauses that function (not the whole server) until the promise resolves.

### Common async mistake
```js
// WRONG: forEach doesn't await properly
users.forEach(async (user) => {
  await sendEmail(user)  // these all fire at once, not sequentially
})

// RIGHT: sequential
for (const user of users) {
  await sendEmail(user)
}

// RIGHT: parallel (all at once, then wait for all)
await Promise.all(users.map(user => sendEmail(user)))
```

### Event loop in one paragraph
Node runs your code on a single thread. When it hits an async operation (DB query, HTTP call, file I/O), it hands it off to the OS and registers a callback. While waiting, Node handles other incoming requests. When the OS is done, it puts the callback in the event queue. Node picks it up on the next loop iteration and runs it. **This is why you should never run heavy CPU work (sorting 10M items, image processing) directly in Node — it blocks the event loop for everyone.**

---

## Level 3: Express.js

### Project setup
```
npm init -y
npm install express
npm install -D nodemon   # auto-restart on file change
```

```json
// package.json scripts
"scripts": {
  "start": "node index.js",
  "dev": "nodemon index.js"
}
```

### Basic server
```js
const express = require('express')
const app = express()

app.use(express.json())  // parse JSON request bodies

app.listen(3000, () => {
  console.log('Server running on port 3000')
})
```

### Routes
```js
// GET /users
app.get('/users', async (req, res) => {
  res.json({ users: [] })
})

// GET /users/:id — URL params
app.get('/users/:id', async (req, res) => {
  const { id } = req.params   // string! convert if needed: parseInt(id)
  res.json({ id })
})

// Query params: GET /users?role=admin&page=2
app.get('/users', async (req, res) => {
  const { role, page = 1 } = req.query
  res.json({ role, page })
})

// POST — body data
app.post('/users', async (req, res) => {
  const { name, email } = req.body
  // ... create user
  res.status(201).json({ message: 'Created', user: { name, email } })
})

// PUT — replace entire resource
app.put('/users/:id', async (req, res) => {
  const { id } = req.params
  const updates = req.body
  res.json({ updated: id })
})

// PATCH — partial update
app.patch('/users/:id', async (req, res) => {
  const { id } = req.params
  const { status } = req.body
  res.json({ id, status })
})

// DELETE
app.delete('/users/:id', async (req, res) => {
  const { id } = req.params
  res.status(204).send()  // 204 = No Content
})
```

### HTTP status codes to know
```
200 OK            — successful GET / PUT / PATCH
201 Created       — successful POST
204 No Content    — successful DELETE
400 Bad Request   — invalid input from client
401 Unauthorized  — not authenticated (no token)
403 Forbidden     — authenticated but not allowed
404 Not Found     — resource doesn't exist
409 Conflict      — resource already exists (duplicate email)
500 Internal Server Error — something crashed on your side
```

### Middleware
Middleware is a function that runs between the request and your route handler. It has access to `req`, `res`, and `next`.

```js
// Built-in middleware
app.use(express.json())          // parse JSON bodies
app.use(express.urlencoded())    // parse form data

// Custom middleware — runs on every request
app.use((req, res, next) => {
  console.log(`${req.method} ${req.path}`)
  next()  // MUST call next() or the request hangs
})

// Auth middleware
function requireAuth(req, res, next) {
  const token = req.headers.authorization?.replace('Bearer ', '')
  if (!token) return res.status(401).json({ error: 'No token' })

  try {
    req.user = verifyToken(token)  // attach user to request
    next()
  } catch {
    res.status(401).json({ error: 'Invalid token' })
  }
}

// Apply to specific routes
app.get('/protected', requireAuth, (req, res) => {
  res.json({ user: req.user })
})

// Apply to a group of routes
const router = express.Router()
router.use(requireAuth)  // all routes in this router need auth
router.get('/settings', (req, res) => { ... })
app.use('/api', router)
```

### Error handling
```js
// Wrap async routes to catch errors
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next)
}

app.get('/users/:id', asyncHandler(async (req, res) => {
  const user = await db.getUser(req.params.id)
  if (!user) return res.status(404).json({ error: 'Not found' })
  res.json(user)
}))

// Global error handler — must have 4 params (err, req, res, next)
app.use((err, req, res, next) => {
  console.error(err.stack)
  res.status(err.status || 500).json({
    error: err.message || 'Internal Server Error'
  })
})
```

---

## Level 4: Connecting to Postgres

### Using the `pg` library (node-postgres)
```
npm install pg
```

```js
const { Pool } = require('pg')

const pool = new Pool({
  connectionString: process.env.DATABASE_URL
  // or:
  // host: 'localhost', port: 5432, database: 'mydb',
  // user: 'postgres', password: 'secret'
})

// Basic query
const result = await pool.query('SELECT * FROM users WHERE id = $1', [userId])
const user = result.rows[0]   // .rows is an array
const users = result.rows     // multiple rows

// Insert and return
const { rows } = await pool.query(
  'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
  [name, email]
)
const newUser = rows[0]
```

**Important:** Always use `$1`, `$2` placeholders — never string interpolation. String interpolation = SQL injection.

```js
// NEVER do this
const query = `SELECT * FROM users WHERE email = '${email}'`  // SQL injection!

// Always do this
const query = 'SELECT * FROM users WHERE email = $1'
const result = await pool.query(query, [email])
```

### A complete CRUD module
```js
// db/users.js
const pool = require('./pool')

async function getUserById(id) {
  const { rows } = await pool.query(
    'SELECT id, name, email, role FROM users WHERE id = $1',
    [id]
  )
  return rows[0] || null
}

async function createUser({ name, email, companyId }) {
  const { rows } = await pool.query(
    'INSERT INTO users (name, email, company_id) VALUES ($1, $2, $3) RETURNING *',
    [name, email, companyId]
  )
  return rows[0]
}

async function updateUser(id, updates) {
  const { rows } = await pool.query(
    'UPDATE users SET name = $1, role = $2 WHERE id = $3 RETURNING *',
    [updates.name, updates.role, id]
  )
  return rows[0] || null
}

async function deleteUser(id) {
  await pool.query('DELETE FROM users WHERE id = $1', [id])
}

module.exports = { getUserById, createUser, updateUser, deleteUser }
```

---

## Level 5: Structuring a Node/Express App

### Folder structure (what interviewers expect you to know)
```
src/
  index.js          — app entry point, starts server
  app.js            — express app setup, middleware, routes
  routes/
    users.js        — route definitions
    companies.js
  controllers/
    users.js        — request handling logic
  services/
    users.js        — business logic (calls DB)
  db/
    pool.js         — db connection
    users.js        — db queries
  middleware/
    auth.js
    errorHandler.js
```

### Environment variables
```js
// .env file (never commit this)
DATABASE_URL=postgres://localhost:5432/mydb
JWT_SECRET=supersecretkey
PORT=3000

// Load with dotenv
require('dotenv').config()
const dbUrl = process.env.DATABASE_URL
const port = process.env.PORT || 3000
```

---

## Complete Example: Compliance Evidence API

```js
// Full working endpoint — what you might write in the interview

const express = require('express')
const { Pool } = require('pg')
const app = express()
app.use(express.json())

const pool = new Pool({ connectionString: process.env.DATABASE_URL })

// GET /companies/:companyId/evidence
// Returns all evidence for a company with control details
app.get('/companies/:companyId/evidence', async (req, res) => {
  const { companyId } = req.params
  const { status, framework } = req.query

  try {
    let query = `
      SELECT
        e.id,
        e.status,
        e.collected_at,
        c.code AS control_code,
        c.description AS control_description,
        f.name AS framework
      FROM evidence e
      JOIN controls c ON e.control_id = c.id
      JOIN frameworks f ON c.framework_id = f.id
      WHERE e.company_id = $1
    `
    const params = [companyId]

    if (status) {
      params.push(status)
      query += ` AND e.status = $${params.length}`
    }
    if (framework) {
      params.push(framework)
      query += ` AND f.name = $${params.length}`
    }

    query += ' ORDER BY e.collected_at DESC'

    const { rows } = await pool.query(query, params)
    res.json({ evidence: rows, count: rows.length })
  } catch (err) {
    console.error(err)
    res.status(500).json({ error: 'Failed to fetch evidence' })
  }
})

// POST /companies/:companyId/evidence
// Create new evidence record
app.post('/companies/:companyId/evidence', async (req, res) => {
  const { companyId } = req.params
  const { controlId, status, data } = req.body

  if (!controlId || !status) {
    return res.status(400).json({ error: 'controlId and status are required' })
  }

  try {
    const { rows } = await pool.query(
      `INSERT INTO evidence (company_id, control_id, status, data)
       VALUES ($1, $2, $3, $4)
       RETURNING *`,
      [companyId, controlId, status, data || null]
    )
    res.status(201).json({ evidence: rows[0] })
  } catch (err) {
    console.error(err)
    res.status(500).json({ error: 'Failed to create evidence' })
  }
})

app.listen(3000, () => console.log('Running on :3000'))
```

---

## Interview Tips

1. **Explain the async model** when asked — "Node handles I/O asynchronously via the event loop, so the server stays responsive while waiting for DB calls"
2. **Always validate input** — check for required fields, return 400 if missing
3. **Use parameterized queries** — mention SQL injection prevention
4. **Mention error handling** — even if you don't write it in full, say "I'd add a try/catch and a global error handler"
5. **Bridge your Django experience** — "Middleware in Express is the same concept as Django middleware — it intercepts requests before they hit the handler"
