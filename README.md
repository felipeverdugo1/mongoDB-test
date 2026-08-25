# Trabajo Práctico 3: Bases de Datos NoSQL y Relacionales (MongoDB)

## Parte 1: Bases de Datos NoSQL y Relacionales

### 1. Mapeo de Conceptos RDBMS a MongoDB
Los conceptos del modelo relacional (RDBMS) tienen su equivalencia en MongoDB de la siguiente manera:

| Concepto RDBMS | Equivalente en MongoDB |
| :--- | :--- |
| Base de Datos | Base de Datos |
| Tabla / Relación | Colección (*Collection*) |
| Fila / Tupla | Documento (*Document*) |
| Columna | Campo (*Field*) |

---

### 2. Transacciones y Propiedades ACID
Antes de la versión 4.0, MongoDB garantizaba transacciones ACID **a nivel de documento único**. Esto implicaba que todas las operaciones realizadas sobre subdocumentos o campos dentro de un mismo documento eran atómicas: si ocurría un error, la base de datos realizaba automáticamente un *rollback* al estado anterior.

A partir de **MongoDB 4.0**, se implementó el soporte para **transacciones ACID multi-documento**, extendiendo la atomicidad, consistencia, aislamiento y durabilidad a través de múltiples colecciones y documentos mediante una interfaz sencilla.

#### Métodos Principales para Manejo de Transacciones
* **Gestión de Sesión:**
  * `session.startSession()`: Abre una sesión transaccional (las transacciones están obligatoriamente ligadas a una sesión).
  * `session.endSession()`: Cierra la sesión. Si hay una transacción activa sin confirmar, esta se abortará automáticamente.
* **Control de Transacción:**
  * `session.startTransaction()`: Inicia la transacción multi-documento dentro de la sesión activa.
  * `session.commitTransaction()`: Confirma y aplica de forma permanente los cambios realizados.
  * `session.abortTransaction()`: Aborta la transacción y revierte todos los cambios ejecutados durante la misma.

---

### 3. Tipos de Índices en MongoDB
MongoDB permite crear índices sobre cualquier campo o subcampo dentro de un documento:

* **Single Field (Campo Único):** Índice sobre un solo campo. MongoDB puede recorrer estos índices en dirección ascendente o descendente según la consulta.
* **Compound Index (Índice Compuesto):** Índice formado por múltiples campos. El orden en el que se declaran los campos dentro del índice es relevante para la optimización.
* **Multikey Index (Índice Multiclave):** Se utiliza para indexar arrays. MongoDB genera entradas de índice independientes para cada elemento del array.
* **Geospatial Index (Índice Geoespacial):** Optimiza consultas de coordenadas geográficas.
  * `2d`: Utiliza geometría plana.
  * `2dsphere`: Utiliza geometría esférica (ideal para datos en la Tierra).
* **Text Index:** Soporta búsquedas de texto plano dentro de campos de tipo *String*, ignorando automáticamente palabras vacías de cada idioma (*stop words* como "the", "a", "or").
* **Hashed Index:** Indexa el valor *hash* de un campo para dar soporte al fragmentado de datos (*sharding*) basado en hash.

---

### 4. Claves Foráneas en MongoDB
En MongoDB **no existen las claves foráneas** con restricciones de integridad referencial como en los sistemas RDBMS tradicionales. En su lugar, se reemplazan por **referencias manuales** (guardando el `_id` del documento relacionado en un nuevo campo) o mediante el estándar **DBRef** (`$ref`, `$id`, `$db`).

---

## Parte 2: Primeros Pasos con MongoDB

### 5. Creación de Base de Datos, Colección e Inserción Única

```javascript
// Selección y creación implícita de la base de datos
use vaccination

// Creación explícita de la colección
db.createCollection("nurses")

// Inserción de un documento
db.nurses.insertOne({ name: "Morella Crespo", experience: 9 })

// Consultas
db.nurses.find()
db.nurses.find().pretty()
```

> **Nota:** La función `find()` retorna el campo primario `_id` generado automáticamente por MongoDB (`ObjectId`), además de los atributos insertados.

---

### 6. Inserción Múltiple y Consultas

```javascript
// Inserción de múltiples documentos
db.nurses.insertMany([
  { name: 'Gale Molina', experience: 8, vaccines: ['AZ', 'Moderna'] },
  { name: 'Honoria Fernández', experience: 5, vaccines: ['Pfizer', 'Moderna', 'Sputnik V'] },
  { name: 'Gonzalo Gallardo', experience: 3 },
  { name: 'Altea Parra', experience: 6, vaccines: ['Pfizer'] }
])

// Enfermeros con experiencia menor o igual a 5 años
db.nurses.find({ experience: { $lte: 5 } })

// Enfermeros que aplicaron la vacuna Pfizer
db.nurses.find({ vaccines: "Pfizer" })

// Enfermeros sin el campo 'vaccines'
db.nurses.find({ vaccines: { $exists: false } })

// Búsqueda por expresión regular (apellido 'Fernández')
db.nurses.find({ name: /Fernández/ })

// Enfermeros con experiencia > 6 años y vacuna 'Moderna'
db.nurses.find({ experience: { $gt: 6 }, vaccines: "Moderna" })

// Proyección: Mostrar solo el nombre (excluir _id)
db.nurses.find(
  { experience: { $gt: 6 }, vaccines: "Moderna" },
  { name: 1, _id: 0 }
)
```

---

### 7. Actualización de un Campo Simple (`$set`)

```javascript
// Actualizar la experiencia de Gale Molina a 9 años
db.nurses.updateOne(
  { name: "Gale Molina" },
  { $set: { experience: 9 } }
)
```

---

### 8. Agregar un Campo Vacío de Tipo Array

```javascript
// Inicializar el campo 'vaccines' como un array vacío para Gonzalo Gallardo
db.nurses.updateOne(
  { name: "Gonzalo Gallardo" },
  { $set: { vaccines: [] } }
)
```

---

### 9. Añadir Elemento a un Array (`$push`)

```javascript
// Añadir 'AZ' a la lista de vacunas de Altea Parra
db.nurses.updateOne(
  { name: "Altea Parra" },
  { $push: { vaccines: "AZ" } }
)
```

---

### 10. Multiplicación de Valores (`$mul`)

```javascript
// Duplicar la experiencia de todos los enfermeros que aplican 'Pfizer'
db.nurses.updateMany(
  { vaccines: "Pfizer" },
  { $mul: { experience: 2 } }
)
```

---

## Parte 3: Índices y Rendimiento

### Carga de Datos de Prueba

```javascript
// Vaciar la colección de enfermeros
db.nurses.deleteMany({})

// Carga de script externo (Ejecutado en Studio 3T por incompatibilidad de load() en mongosh)
load('C:\Users\Felipe\Desktop\generador.js')
```

---

### 11. Obtener Índices de una Colección

```javascript
db.doses.getIndexes()
```

---

### 12. Creación, Análisis y Eliminación de Índices

```javascript
// 1. Crear índice en el campo 'nurse'
db.doses.createIndex({ nurse: 1 })

// 2. Analizar rendimiento de consulta con índice
db.doses.find({ nurse: /11/ }).explain("executionStats")
/*
  executionTimeMillis: 1133 ms
  totalKeysExamined: 209648
  totalDocsExamined: 12796
*/

// 3. Eliminar el índice
db.doses.dropIndex("nurse_1")

// 4. Analizar rendimiento sin índice (COLLSCAN)
db.doses.find({ nurse: /11/ }).explain("executionStats")
/*
  executionTimeMillis: 1746 ms
  totalKeysExamined: 0
  totalDocsExamined: 209648
*/
```

---

### 13. Consultas Geoespaciales y Optimización con `2dsphere`

```javascript
// Definición del polígono de CABA
var caba = {
  "type": "MultiPolygon",
  "coordinates": [[[
    [-58.46305847167969, -34.53456089748654],
    [-58.49979400634765, -34.54983198845187],
    [-58.532066345214844, -34.614561581608186],
    [-58.528633117675774, -34.6538270014492],
    [-58.48674774169922, -34.68742794931483],
    [-58.479881286621094, -34.68206400648744],
    [-58.46855163574218, -34.65297974261105],
    [-58.465118408203125, -34.64733112904415],
    [-58.4585952758789, -34.63998735602951],
    [-58.45344543457032, -34.63603274732642],
    [-58.447265625, -34.63575026806082],
    [-58.438339233398445, -34.63038297923296],
    [-58.38100433349609, -34.62162507826766],
    [-58.38237762451171, -34.59251960889388],
    [-58.378944396972656, -34.5843230246475],
    [-58.46305847167969, -34.53456089748654]
  ]]]
}

// Consulta de intersección geoespacial
db.patients.find({ address: { $geoIntersects: { $geometry: caba } } })
db.patients.countDocuments({ address: { $geoIntersects: { $geometry: caba } } }) // Resultado: 46045 de 209648

// Análisis SIN índice geoespacial
db.patients.find({ address: { $geoIntersects: { $geometry: caba } } }).explain("executionStats")
// executionTimeMillis: 3030 ms | totalDocsExamined: 209648

// Crear índice geoespacial 2dsphere
db.patients.createIndex({ address: '2dsphere' })

// Análisis CON índice geoespacial
db.patients.find({ address: { $geoIntersects: { $geometry: caba } } }).explain("executionStats")
// executionTimeMillis: 2813 ms | totalDocsExamined: 58125
```

---

## Parte 4: Aggregation Framework

### 14. Muestreo Aleatorio (`$sample`)

```javascript
// Obtener 5 documentos aleatorios de la colección patients
db.patients.aggregate([
  { $sample: { size: 5 } }
])
```

---

### 15. Búsqueda por Proximidad y Exportación (`$geoNear` + `$out`)

```javascript
// Pacientes a menos de 1km de un punto y guardado en una nueva colección
db.patients.aggregate([
  {
    $geoNear: {
      near: { type: "Point", coordinates: [-58.4586, -34.5968] },
      distanceField: "distancesToPoint",
      maxDistance: 1000,
      spherical: true
    }
  },
  { $out: "pacientes1km" }
])
```

---

### 16. Unión de Colecciones (`$lookup`)

```javascript
var pacientesBs = {
  $geoNear: {
    near: { type: "Point", coordinates: [-58.4586, -34.5968] },
    distanceField: "distancesToPoint",
    maxDistance: 1000,
    spherical: true
  }
}

var lookup = {
  $lookup: {
    from: "doses",
    localField: "name",
    foreignField: "patient",
    as: "doses"
  }
}

db.patients.aggregate([pacientesBs, lookup, { $out: "dosis1km" }])
```

---

### 17. Pipeline Complejo con Filtrado y Subconsultas

```javascript
db.nurses.aggregate([
  { $match: { name: { $regex: "111" } } },
  { $addFields: { doses: [] } },
  {
    $lookup: {
      from: "doses",
      localField: "name",
      foreignField: "nurse",
      as: "doses",
      pipeline: [
        { $match: { date: { $gt: ISODate("2021-05-01") } } }
      ]
    }
  },
  { $out: "pacientes111" }
])

// Verificación de registros procesados
db.pacientes111.countDocuments() // Resultado: 22
```