# Flat Files DB Backend Principles

---

**Version** : 1.0  
**Last Updated** : June 26, 2026  
**Author** : Oswald Bernard (Steroids Studio)  
**Technologies** : Node.js, LokiJS, YAML, `proper-lockfile`

---

## 📌 1. Overview

### 1.1 System Description
This system allows you to **manage invoices** stored as **individual YAML files** (one file per invoice) with:
- **Automatic indexes** to speed up queries (via LokiJS).
- **Materialized views** for frequent queries (e.g., unpaid invoices).
- **Locking mechanism** to handle concurrent writes (via `proper-lockfile`).
- **Reliable persistence** of data on disk.

### 1.2 Architecture
```
backend/
├── factures/               # Directory for invoices (1 YAML file = 1 invoice)
│   ├── facture_1.yml
│   ├── facture_2.yml
│   └── ...
├── indexes/                # Directory for indexes (optional, if using external indexes)
│   └── ...
├── db.json                # LokiJS file (metadata + in-memory indexes)
├── server.js              # Entry point for the server (REST API)
└── package.json
```

### 1.3 Data Flow
```mermaid
graph TD
    A[Client] -->|HTTP Request| B[Node.js Server]
    B -->|Read| C[LokiJS]
    B -->|Write| C
    C -->|Load| D[YAML Files]
    C -->|Save| D
    C -->|Index| E[LokiJS Indexes]
    C -->|Views| F[LokiJS Collections]
    D -->|Lock| G[proper-lockfile]
```

---

## 🛠️ 2. Prerequisites

### 2.1 Environment
- **Node.js**: Version 18+ (recommended for modern features).
- **NPM/Yarn**: For dependency management.
- **File System**: Read/write access (for YAML files and locks).

### 2.2 Dependencies
| **Package**            | **Version** | **Role**                                                                 |
|------------------------|-------------|--------------------------------------------------------------------------|
| `lokijs`               | ^1.5.11     | In-memory NoSQL database with index support.               |
| `js-yaml`              | ^4.1.0      | Parsing and serialization of YAML files.                            |
| `proper-lockfile`      | ^4.1.2      | Locking mechanism for concurrent writes.                   |
| `express`              | ^4.18.2     | HTTP server to expose a REST API (optional).                   |

**Installation**:
```bash
npm install lokijs js-yaml proper-lockfile express
```

---

## 📦 3. Data Structure

### 3.1 Invoice Format (YAML)
Each invoice is stored in a **separate YAML file** (`factures/facture_{id}.yml`).

**Example** (`facture_1.yml`):
```yaml
id: 1
montant: 100.50
date: "2026-06-26"       # Format: YYYY-MM-DD
client: "Client A"
statut: "non payée"       # Possible values: "payée", "non payée", "annulée"
description: "Service de développement"
```

### 3.2 LokiJS Collections
| **Collection**          | **Role**                                                                 | **Indexes**                          | **Associated Views**               |
|-------------------------|--------------------------------------------------------------------------|------------------------------------|----------------------------------|
| `factures`              | Main collection containing all invoices.                     | `id` (unique), `client`, `date`, `statut` | `factures_non_payees`, `factures_par_client` |
| `factures_non_payees`  | Materialized view of invoices with `statut = "non payée"`.              | `date`                             | -                                |
| `factures_par_client`   | Materialized view grouping invoices by `client`.                 | `client`                          | -                                |

### 3.3 Indexes
- **Unique Indexes**:
  - `id`: Ensures each invoice has a unique identifier.
- **Standard Indexes**:
  - `client`, `date`, `statut`, `montant`: To speed up frequent queries.

---

## 🔧 4. System Functionality

### 4.1 Database Initialization
On server startup, LokiJS:
1. Loads **YAML files** from `factures/` into the `factures` collection.
2. **Rebuilds indexes** in memory.
3. **Populates materialized views** (`factures_non_payees`, `factures_par_client`).

**Code**:
```javascript
const loki = require('lokijs');
const fs = require('fs');
const path = require('path');
const yaml = require('js-yaml');

const FACTURES_DIR = path.join(__dirname, 'factures');
if (!fs.existsSync(FACTURES_DIR)) fs.mkdirSync(FACTURES_DIR);

const db = new loki('db.json');
const factures = db.addCollection('factures', {
  indices: ['client', 'date', 'statut', 'montant'],
  unique: ['id']
});

// Materialized views
const facturesNonPayees = db.addCollection('factures_non_payees');
facturesNonPayees.createIndex('date');

const facturesParClient = db.addCollection('factures_par_client');
facturesParClient.createIndex('client');

// Load invoices from YAML files
const files = fs.readdirSync(FACTURES_DIR).filter(f => f.endsWith('.yml'));
files.forEach(file => {
  const filePath = path.join(FACTURES_DIR, file);
  const data = yaml.load(fs.readFileSync(filePath, 'utf-8'));
  if (data && data.id) {
    const doc = factures.insert(data);
    // Populate views
    if (data.statut === "non payée") facturesNonPayees.insert(doc);
    let clientDocs = facturesParClient.findOne({ client: data.client });
    if (!clientDocs) {
      clientDocs = { client: data.client, factures: [] };
      facturesParClient.insert(clientDocs);
    }
    clientDocs.factures.push(doc);
    facturesParClient.update(clientDocs);
  }
});
```

---

### 4.2 Writing Data (with Locking)
To prevent **write conflicts** (e.g., two processes modifying `facture_1.yml` simultaneously), we use **`proper-lockfile`** to create a **lock per file**.

#### Locking Mechanism
1. **Before any write**:
   - A **lock file** (`facture_{id}.yml.lock`) is created for the concerned file.
   - If the lock already exists, the process **waits** (or fails after a timeout).
2. **After writing**:
   - The lock is **removed**.
3. **Cleanup stale locks**:
   - On startup, remove locks that are **too old** (e.g., > 10 seconds).

**Code**:
```javascript
const lockfile = require('proper-lockfile');

async function saveFacture(facture) {
  const lockFile = path.join(FACTURES_DIR, `${facture.id}.yml.lock`);
  const filePath = path.join(FACTURES_DIR, `${facture.id}.yml`);

  try {
    // Acquire lock (5-second timeout)
    await lockfile.lock(lockFile, { stale: 5000 });
    // Write YAML file
    fs.writeFileSync(filePath, yaml.dump(facture, { sortKeys: true }));
    // Release lock
    await lockfile.unlock(lockFile);
    return { success: true };
  } catch (err) {
    // Release lock in case of error
    await lockfile.unlock(lockFile).catch(() => {});
    return { success: false, error: err.message };
  }
}

// Cleanup stale locks on startup
function cleanupStaleLocks() {
  const now = Date.now();
  fs.readdirSync(FACTURES_DIR)
    .filter(f => f.endsWith('.yml.lock'))
    .forEach(lockFile => {
      const filePath = path.join(FACTURES_DIR, lockFile);
      const stats = fs.statSync(filePath);
      // Remove locks older than 10 seconds
      if (now - stats.mtimeMs > 10000) {
        fs.unlinkSync(filePath);
      }
    });
}

cleanupStaleLocks();
```

---

### 4.3 Updating Views
Materialized views (`factures_non_payees`, `factures_par_client`) are **automatically updated** via LokiJS event listeners:

```javascript
// Listen for insertions
factures.on('insert', (doc) => {
  if (doc.statut === "non payée") facturesNonPayees.insert(doc);
  let clientDocs = facturesParClient.findOne({ client: doc.client });
  if (!clientDocs) {
    clientDocs = { client: doc.client, factures: [] };
    facturesParClient.insert(clientDocs);
  }
  clientDocs.factures.push(doc);
  facturesParClient.update(clientDocs);
});

// Listen for updates
factures.on('update', (doc) => {
  // Update "non payées" view
  const oldInView = facturesNonPayees.findOne({ id: doc.id });
  if (oldInView) facturesNonPayees.remove(oldInView);
  if (doc.statut === "non payée") facturesNonPayees.insert(doc);

  // Update "par client" view (client may have changed)
  const oldDoc = factures.findOne({ id: doc.id });
  if (oldDoc && oldDoc.client !== doc.client) {
    const oldClientDocs = facturesParClient.findOne({ client: oldDoc.client });
    if (oldClientDocs) {
      oldClientDocs.factures = oldClientDocs.factures.filter(f => f.id !== doc.id);
      facturesParClient.update(oldClientDocs);
    }
  }
  let clientDocs = facturesParClient.findOne({ client: doc.client });
  if (!clientDocs) {
    clientDocs = { client: doc.client, factures: [] };
    facturesParClient.insert(clientDocs);
  }
  if (!clientDocs.factures.some(f => f.id === doc.id)) {
    clientDocs.factures.push(doc);
    facturesParClient.update(clientDocs);
  }
});

// Listen for deletions
factures.on('delete', (doc) => {
  const docInView = facturesNonPayees.findOne({ id: doc.id });
  if (docInView) facturesNonPayees.remove(docInView);
  const clientDocs = facturesParClient.findOne({ client: doc.client });
  if (clientDocs) {
    clientDocs.factures = clientDocs.factures.filter(f => f.id !== doc.id);
    facturesParClient.update(clientDocs);
  }
});
```

---

### 4.4 Saving Data
The `saveDatabase()` method in LokiJS is **overridden** to:
1. Save each modified invoice to its YAML file.
2. Use locks to prevent conflicts.

**Code**:
```javascript
class YamlPerDocAdapter {
  constructor(dir) {
    this.dir = dir;
    this.modifiedDocs = new Set();
  }

  loadDatabase(dbname, callback) {
    // ... (see section 4.1)
  }

  async saveDatabase(dbname, db, callback) {
    const factures = db.getCollection('factures');
    const errors = [];

    for (const id of this.modifiedDocs) {
      const lockFile = path.join(this.dir, `${id}.yml.lock`);
      const filePath = path.join(this.dir, `${id}.yml`);
      try {
        await lockfile.lock(lockFile, { stale: 5000 });
        const doc = factures.findOne({ id: id });
        if (doc) {
          fs.writeFileSync(filePath, yaml.dump(doc, { sortKeys: true }));
        } else {
          if (fs.existsSync(filePath)) fs.unlinkSync(filePath);
        }
        await lockfile.unlock(lockFile);
      } catch (err) {
        errors.push({ id, error: err.message });
        await lockfile.unlock(lockFile).catch(() => {});
      }
    }

    this.modifiedDocs.clear();
    callback(errors.length > 0 ? new Error(`Errors: ${JSON.stringify(errors)}`) : null);
  }
}

// Initialize LokiJS with the adapter
const adapter = new YamlPerDocAdapter(FACTURES_DIR);
const db = new loki('db.json', { adapter: adapter });
```

---

## 🔌 5. REST API (Optional)
To expose functionality via an **HTTP API**, use **Express**:

```javascript
const express = require('express');
const app = express();
app.use(express.json());

// Get all invoices
app.get('/factures', (req, res) => {
  const allFactures = factures.find();
  res.json(allFactures);
});

// Get an invoice by ID
app.get('/factures/:id', (req, res) => {
  const facture = factures.findOne({ id: parseInt(req.params.id) });
  if (!facture) return res.status(404).json({ error: "Invoice not found" });
  res.json(facture);
});

// Get unpaid invoices
app.get('/factures/non-payees', (req, res) => {
  const nonPayees = facturesNonPayees.find();
  res.json(nonPayees);
});

// Get invoices by client
app.get('/factures/par-client/:client', (req, res) => {
  const clientDocs = facturesParClient.findOne({ client: req.params.client });
  if (!clientDocs) return res.status(404).json({ error: "Client not found" });
  res.json(clientDocs.factures);
});

// Add an invoice
app.post('/factures', async (req, res) => {
  try {
    const facture = req.body;
    if (!facture.id || !facture.montant || !facture.date || !facture.client) {
      return res.status(400).json({ error: "Missing fields" });
    }
    const existing = factures.findOne({ id: facture.id });
    if (existing) {
      return res.status(409).json({ error: "Invoice already exists" });
    }
    factures.insert(facture);
    await new Promise((resolve, reject) => {
      db.saveDatabase((err) => err ? reject(err) : resolve());
    });
    res.json({ success: true, id: facture.id });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Update an invoice
app.put('/factures/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id);
    const updates = req.body;
    const facture = factures.findOne({ id });
    if (!facture) return res.status(404).json({ error: "Invoice not found" });
    Object.assign(facture, updates);
    factures.update(facture);
    await new Promise((resolve, reject) => {
      db.saveDatabase((err) => err ? reject(err) : resolve());
    });
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// Delete an invoice
app.delete('/factures/:id', async (req, res) => {
  try {
    const id = parseInt(req.params.id);
    const facture = factures.findOne({ id });
    if (!facture) return res.status(404).json({ error: "Invoice not found" });
    factures.remove(facture);
    await new Promise((resolve, reject) => {
      db.saveDatabase((err) => err ? reject(err) : resolve());
    });
    res.json({ success: true });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

app.listen(3000, () => {
  console.log('Server started on http://localhost:3000');
});
```

---

## 📜 6. Usage Examples

### 6.1 Add an Invoice
**HTTP Request**:
```bash
curl -X POST http://localhost:3000/factures \
  -H "Content-Type: application/json" \
  -d '{
    "id": 1,
    "montant": 100.50,
    "date": "2026-06-26",
    "client": "Client A",
    "statut": "non payée"
  }'
```

**Result**:
- A file `factures/facture_1.yml` is created.
- The views `factures_non_payees` and `factures_par_client` are updated.

---

### 6.2 Update an Invoice
**HTTP Request**:
```bash
curl -X PUT http://localhost:3000/factures/1 \
  -H "Content-Type: application/json" \
  -d '{
    "statut": "payée"
  }'
```

**Result**:
- The file `factures/facture_1.yml` is updated.
- The invoice is **removed** from the `factures_non_payees` view.

---

### 6.3 Get Unpaid Invoices
**HTTP Request**:
```bash
curl http://localhost:3000/factures/non-payees
```

**Result**:
```json
[
  {
    "id": 2,
    "montant": 200,
    "date": "2026-06-25",
    "client": "Client B",
    "statut": "non payée"
  },
  {
    "id": 3,
    "montant": 150,
    "date": "2026-06-24",
    "client": "Client C",
    "statut": "non payée"
  }
]
```

---

### 6.4 Get Invoices by Client
**HTTP Request**:
```bash
curl http://localhost:3000/factures/par-client/Client%20A
```

**Result**:
```json
[
  {
    "id": 1,
    "montant": 100.50,
    "date": "2026-06-26",
    "client": "Client A",
    "statut": "payée"
  }
]
```

---

## ⚠️ 7. Error Handling and Best Practices

### 7.1 Common Errors and Solutions
| **Error**                          | **Cause**                                  | **Solution**                                                                 |
|------------------------------------|-------------------------------------------|------------------------------------------------------------------------------|
| `Error: ENOENT: no such file or directory` | Missing `factures/` directory.              | Create the directory before starting (`fs.mkdirSync`).                      |
| `Error: Lock stale`                 | Lock file blocked for too long.             | Increase timeout (`stale: 10000`) or clean up stale locks.                    |
| `Error: Duplicate key`              | Duplicate `id`.                            | Check for uniqueness of `id` before insertion.                              |
| `Error: YAML parse error`           | Malformed YAML file.                       | Validate YAML before loading (e.g., with `js-yaml.load`).                   |
| `Error: Database not loaded`         | `autoloadCallback` not called.            | Ensure `db.loadDatabase` is called before any operation.                  |

---

### 7.2 Best Practices
1. **Always use locks**:
   - **Every write operation** (add, update, delete) must be protected by a lock.
   - Use `proper-lockfile` to avoid conflicts.

2. **Clean up stale locks**:
   - On startup, remove locks that are **too old** (e.g., > 10 seconds).

3. **Validate data**:
   - Ensure required fields (`id`, `montant`, `date`, `client`) are present before insertion.

4. **Save regularly**:
   - Call `db.saveDatabase()` after each modification to persist changes.

5. **Optimize indexes**:
   - Only create indexes on **frequently queried fields**.

6. **Handle errors**:
   - Use `try/catch` to capture locking or write errors.

7. **Test concurrent scenarios**:
   - Test with **multiple simultaneous PUT requests** on the same invoice.

---

## 📊 8. Performance and Limits

### 8.1 Performance
| **Operation**               | **Complexity** | **Estimated Time (1000 invoices)** | **Optimizations**                          |
|-----------------------------|----------------|-----------------------------------|-------------------------------------------|
| Insert an invoice           | O(1)           | ~5 ms                             | Lock per file.                             |
| Update an invoice           | O(1)           | ~5 ms                             | Lock per file.                             |
| Delete an invoice           | O(1)           | ~5 ms                             | Lock per file.                             |
| Query with index            | O(log n)       | ~1-2 ms                           | Use LokiJS indexes.                        |
| Query without index         | O(n)           | ~10-20 ms                         | Avoid queries without indexes.             |
| Initial load                | O(n)           | ~50-100 ms                        | Load files in parallel.                    |

---

### 8.2 Limits
| **Limit**                          | **Description**                                                                 | **Solution**                                                                 |
|------------------------------------|---------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| **Maximum data size**              | Depends on available memory (LokiJS loads everything in memory).              | Use a backend with pagination or streaming for large datasets.             |
| **High concurrency**                | File locks may slow down writes if many conflicts occur.                     | Use a global lock for critical operations.                                  |
| **No transactions**                 | LokiJS does not support multi-document transactions.                         | Use a SQL backend (e.g., SQLite) if transactions are required.             |
| **Persistence**                     | Data is first loaded in memory before being saved to disk.                     | Call `db.saveDatabase()` frequently to avoid data loss.                     |

---

## 🔧 9. Configuration and Deployment

### 9.1 Basic Configuration
Create a `config.js` file to centralize settings:
```javascript
module.exports = {
  facturesDir: path.join(__dirname, 'factures'),
  lockTimeout: 5000, // Lock timeout in ms
  dbFile: 'db.json', // LokiJS file
  port: 3000 // Express server port
};
```

---

### 9.2 Deployment
1. **Install dependencies**:
   ```bash
   npm install
   ```

2. **Start the server**:
   ```bash
   node server.js
   ```

3. **Check logs**:
   - The server should display:
     ```
     Server started on http://localhost:3000
     Database initialized successfully!
     ```

4. **Test the API**:
   - Use `curl` or Postman to test the endpoints (see section 6).

---

### 9.3 Environments
| **Environment** | **Configuration**                          | **Usage**                          |
|-------------------|--------------------------------------------|------------------------------------------|
| **Development**   | `NODE_ENV=development`                     | Detailed logs, no minification.          |
| **Production**    | `NODE_ENV=production`                      | Reduced logs, optimizations enabled.    |
| **Test**          | `NODE_ENV=test`                            | Ephemeral database.                     |

---

## 📚 10. Additional Documentation

### 10.1 LokiJS
- **Official Documentation**: [LokiJS GitHub](https://github.com/techfort/LokiJS)
- **Indexing**: [LokiJS Indexing](https://github.com/techfort/LokiJS/wiki/Indexing)
- **Views**: [LokiJS Views](https://github.com/techfort/LokiJS/wiki/Dynamic-Views)

### 10.2 proper-lockfile
- **Documentation**: [proper-lockfile GitHub](https://github.com/moxystudio/node-proper-lockfile)
- **Usage**: [Usage](https://github.com/moxystudio/node-proper-lockfile#usage)

### 10.3 YAML
- **Specification**: [YAML 1.2](https://yaml.org/spec/1.2/spec.html)
- **Validation**: Use `js-yaml` to validate files before loading.

---

## 🎯 11. Useful Code Snippets

### 11.1 Validate an Invoice Before Insertion
```javascript
function validateInvoice(invoice) {
  const requiredFields = ['id', 'montant', 'date', 'client', 'statut'];
  for (const field of requiredFields) {
    if (!(field in invoice)) {
      throw new Error(`Missing field: ${field}`);
    }
  }
  if (typeof invoice.id !== 'number') {
    throw new Error("ID must be a number.");
  }
  if (typeof invoice.montant !== 'number' || invoice.montant <= 0) {
    throw new Error("Montant must be a positive number.");
  }
  if (!/^\d{4}-\d{2}-\d{2}$/.test(invoice.date)) {
    throw new Error("Date must be in YYYY-MM-DD format.");
  }
  const validStatuses = ['payée', 'non payée', 'annulée'];
  if (!validStatuses.includes(invoice.statut)) {
    throw new Error(`Invalid status. Valid values: ${validStatuses.join(', ')}`);
  }
}
```

---

### 11.2 Export Invoices to JSON
```javascript
function exportInvoicesToJSON() {
  const invoices = db.getCollection('factures').find();
  const json = JSON.stringify(invoices, null, 2);
  fs.writeFileSync('export_invoices.json', json);
  return json;
}
```

---

### 11.3 Import Invoices from JSON
```javascript
function importInvoicesFromJSON(json) {
  const invoices = JSON.parse(json);
  invoices.forEach(invoice => {
    try {
      validateInvoice(invoice);
      const existing = db.getCollection('factures').findOne({ id: invoice.id });
      if (existing) {
        db.getCollection('factures').update(invoice);
      } else {
        db.getCollection('factures').insert(invoice);
      }
    } catch (err) {
      console.error(`Error importing invoice ${invoice.id}:`, err.message);
    }
  });
  db.saveDatabase();
}
```

---

## 🚀 12. Roadmap and Possible Improvements
| **Improvement**               | **Description**                                                                 | **Priority** | **Complexity** |
|--------------------------------|---------------------------------------------------------------------------------|--------------|----------------|
| **Automatic backup**           | Backup YAML files to a `backup/` directory at regular intervals.              | ⭐⭐⭐        | ⭐⭐           |
| **File compression**           | Compress YAML files to save space.                                             | ⭐⭐          | ⭐⭐⭐         |
| **Encryption**                 | Encrypt sensitive YAML files.                                                  | ⭐⭐          | ⭐⭐⭐⭐       |
| **GraphQL API**                | Replace REST API with GraphQL for more flexibility.                           | ⭐           | ⭐⭐⭐⭐       |
| **Webhooks**                    | Notify an external service on updates.                                         | ⭐⭐          | ⭐⭐           |
| **Redis cache**                 | Use Redis to cache frequent views.                                             | ⭐⭐          | ⭐⭐⭐         |
| **Unit tests**                  | Add tests for critical functions.                                              | ⭐⭐⭐⭐       | ⭐⭐           |

---

## 📜 Appendix: Glossary
| **Term**               | **Definition**                                                                 |
|-------------------------|---------------------------------------------------------------------------------|
| **LokiJS**              | In-memory NoSQL database for Node.js and the browser.                |
| **Index**               | Optimized data structure to speed up queries on a field.       |
| **View**                 | Precomputed or dynamic result of a query (e.g., unpaid invoices).     |
| **Lock**                 | Mechanism to prevent concurrent writes to a file.              |
| **YAML**                | Human-readable data serialization format.                       |
| **Adapter**             | Class that handles data persistence for LokiJS (e.g., IndexedDB, YAML). |
| **Collection**          | Set of documents in LokiJS (equivalent to a table in SQL).             |

---

## 📞 13. Support and Contribution

### 13.1 Report a Bug
- **Open an issue** on the repository (if applicable).
- **Include**:
  - Steps to reproduce the bug.
  - Error logs.
  - Node.js and dependency versions.

### 13.2 Contribute
- **Fork** the project and submit a **Pull Request**.
- **Follow** coding conventions (e.g., ESLint, Prettier).
- **Add tests** for new features.

---

## 🏁 Conclusion
This system provides a **robust and scalable** solution for managing invoices in **backend** with:
- **YAML file storage** (1 file = 1 invoice).
- **Automatic indexes** for optimized queries.
- **Materialized views** for frequent queries.
- **Locking mechanism** for concurrent writes.
- **REST API** for data interaction.

**Next Steps**:
1. **Test** the system with real data.
2. **Optimize** performance (e.g., caching, additional indexes).
3. **Secure** the API (e.g., authentication, input validation).
4. **Deploy** to production (e.g., Docker, PM2).

---

**Author**: Oswald Bernard (Steroids Studio)  
**Contact**: oswald.bernard@steroids.studio  
**License**: MIT (adapt as needed)