# MongoDB Class Notes — CRUD, Arrays, Projections, Updates & Aggregations (Cheat-Sheet)

**Author:** Erick González  
**Last updated:** 2025-09-14

---

## Table of Contents
1. [Shell Basics](#shell-basics)  
2. [Reads: `find` & Projections](#reads-find--projections)  
3. [Sorting, Limiting, Counting](#sorting-limiting-counting)  
4. [Dot Notation (Objects & Arrays)](#dot-notation-objects--arrays)  
5. [Updates](#updates)  
6. [Arrays: Replace vs Append](#arrays-replace-vs-append)  
7. [Deletions](#deletions)  
8. [Aggregation Pipeline](#aggregation-pipeline)  
9. [Practical Patterns](#practical-patterns)  
10. [Common Pitfalls](#common-pitfalls)  
11. [Quick Reference](#quick-reference)

---

## Shell Basics
```javascript
use("dbName");          // Switch DB (created on first write)
show dbs;               // List DBs
show collections;       // List collections
db.getSiblingDB("otherDB").collection.find(); // Access other DB without switching
```

---

## Reads: `find` & Projections
```javascript
db.empleados.find();  // All documents
db.empleados.find({ depto: "TI" });  // Equality
db.empleados.find({ salario: { $gte: 4000, $lte: 5000 } }); // Range (inclusive)
db.empleados.find({}, { nombre: 1, salario: 1, _id: 0 });   // Projection
db.empleados.find({ nombre: /^A/ }); // Starts-with (anchored regex)
db.empleados.find({ depto: { $in: ["TI", "Marketing"] } }); // Membership
```
- `$gt/$gte/$lt/$lte` are strict vs inclusive comparisons.  
- Projection: do **not** mix include and exclude (except `_id:0`).

---

## Sorting, Limiting, Counting
```javascript
db.empleados.find().sort({ antiguedad: 1 }).limit(3); // Top-3 by lowest antigüedad
db.empleados.countDocuments();                         // Accurate count (with optional filter)
db.empleados.find({ salario: { $gte: 4000, $lte: 5000 } }).pretty(); // Pretty output (shell-only)
```
- `1` ascending, `-1` descending.  
- `limit(n)` avoids dumping huge result sets.

---

## Dot Notation (Objects & Arrays)
```javascript
db.col.find({ "direccion.ciudad": "Grecia" }); // Object field
db.col.find({ "telefonos.0": "+506..." });     // Array index
db.col.find({ "cursos.nombre": "MongoDB" });   // Array of objects
```
Be explicit—MongoDB won’t infer nested paths.

---

## Updates
```javascript
// Assign exact values
db.empleados.updateOne({ empleadoId: 47 }, { $set: { depto: "Consultoría" } });

// Numeric change without read-modify-write on client
db.empleados.updateMany({ depto: "Finanzas" }, { $inc: { salario: 500 } }); // +500
db.empleados.updateMany({ depto: "Ventas"   }, { $mul: { salario: 1.10 } }); // +10%

// Bulk status by filter
db.empleados.updateMany({ antiguedad: { $lt: 2 } }, { $set: { activo: false } });

// Dates
db.empleados.updateOne({ empleadoId: 36 }, { $set: { ultimaReevaluacion: new Date() } });

// Server-side string update (atomic; MongoDB ≥4.2)
db.empleados.updateOne(
  { empleadoId: 7 },
  [ { $set: { nombre: { $concat: ["S. ", "$nombre"] } } } ]
);

// Remove field
db.empleados.updateOne({ empleadoId: 1 }, { $unset: { deprecated: "" } });
```
- Use `$set` for explicit assignment; `$inc` for additive; `$mul` for percentages.  
- Prefer pipeline updates for transformations that depend on current values.

---

## Arrays: Replace vs Append
```javascript
// Replace specific index
db.products.updateOne({ _id: 1 }, { $set: { "storage.0": 16 } });

// Append values
db.products.updateOne({ _id: 1 }, { $push: { storage: { $each: [512, 1024] } } });

// Replace the whole array
db.products.updateOne({ _id: 1 }, { $set: { storage: [16, 32] } });
```

---

## Deletions
```javascript
db.empleados.deleteOne({ empleadoId: 4 });                     // First match
db.empleados.deleteMany({ depto: "Marketing" });               // All matches
db.empleados.deleteMany({ activo: false, salario: { $lt: 4000 } });
```
Preview with `find()` before destructive ops.

---

## Aggregation Pipeline
```javascript
db.collection.aggregate([
  { $match:  { depto: "Legal" } },                  // Filter
  { $group:  { _id: "$depto", total: { $sum: "$salario" } } }, // Aggregate
  { $sort:   { total: -1 } },                       // Order
  { $project:{ _id: 0, depto: "$_id", total: 1 } }  // Shape output
]);
```
- `_id` in `$group` can be a field (e.g., `"$depto"`) or a constant (single bucket).  
- Aggregations run server-side and are efficient with indexes and selective `$match`.

---

## Practical Patterns
- Top-N by metric: `.find(filter).sort({ metric: -1 }).limit(N)`  
- Percentage raises: `$mul: { salario: 1.xx }`  
- Fixed bonuses/counters: `$inc: { campo: N }`  
- Multi-category filters: `{ campo: { $in: [...] } }`  
- Transform in place safely: aggregation **pipeline** updates `[{ $set: ... }]`  
- Remove flags/fields cleanly: `$unset`

---

## Common Pitfalls
- Filters in one object are **AND**ed: `{ a:1, b:2 }` (use `$or` when needed).  
- Projection: don’t mix include/exclude (except `_id`).  
- Unanchored regexes (`/foo/` without `^`) often skip indexes.  
- Client-side read-concat-write is race-prone—prefer server-side pipeline updates.  
- Always verify bulk updates/deletes with a prior `find()`.

---

## Quick Reference
```javascript
// READ
db.col.find(query, projection);
db.col.find().sort({ field: 1 }).limit(n);
db.col.countDocuments(query);

// UPDATE
db.col.updateOne(filter, { $set: { "a.b": 1 } });
db.col.updateMany(filter, { $inc: { n: 5 } });
db.col.updateMany(filter, { $mul: { n: 1.1 } });
db.col.updateOne(filter, [ { $set: { s: { $concat: ["x", "$s"] } } } ]);
db.col.updateOne(filter, { $unset: { oldField: "" } });

// ARRAYS
db.col.updateOne(filter, { $set:  { "arr.0": 123 } });
db.col.updateOne(filter, { $push: { arr: { $each: [1,2,3] } } });

// DELETE
db.col.deleteOne(filter);
db.col.deleteMany(filter);

// AGGREGATE
db.col.aggregate([
  { $match: { ... } },
  { $group: { _id: "$field", total: { $sum: 1 } } },
  { $sort:  { total: -1 } },
  { $project: { _id: 0, field: "$_id", total: 1 } }
]);
```
