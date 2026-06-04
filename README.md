# 🏆 Taller Práctico: Data Analytics con DuckDB (Mundial 2026 Projections)

¡Bienvenidos al repositorio oficial del taller práctico del grupo **SalsaLizano** para el curso de Bases de Datos (CE3101)!

En este laboratorio demostramos cómo realizar flujos de **Data Analytics** de alto rendimiento en entornos locales utilizando tecnologías embebidas modernas.

---

## 📝 Descripción del Taller y Objetivos

El propósito de este taller es procesar y analizar de forma masiva variables estadísticas de las selecciones nacionales y jugadores clasificados al **Mundial 2026** para resolver tres grandes interrogantes estratégicas:

1. **Selección Potencial Ganadora:** Basado en rendimiento colectivo y efectividad de goles.
2. **Bota de Oro:** Proyección del máximo goleador del torneo.
3. **Guante de Oro:** Proyección del arquero menos batido (vallas invictas).

### 🛠️ Herramienta Utilizada: DuckDB

Para este análisis dejamos de lado los motores relacionales tradicionales orientados a filas (como MySQL o PostgreSQL), los cuales sufren graves caídas de rendimiento al procesar analítica masiva. En su lugar, utilizamos **DuckDB**: un motor de base de datos **OLAP** y embebido (el _"SQLite para Analytics"_).

**¿Por qué DuckDB?**

- **Almacenamiento Columnar:** Lee únicamente las columnas necesarias para las fórmulas analíticas, reduciendo drásticamente el uso de memoria RAM (I/O).
- **Procesamiento Vectorial:** Ejecuta operaciones en bloques masivos aprovechando las instrucciones de hardware de las CPUs modernas para resolver consultas en milisegundos.
- **Arquitectura Embebida:** Corre directamente dentro del proceso de Node.js, sin necesidad de instalar servidores externos pesados.

---

## 🛠️ Paso 1: Configuración del Entorno desde Cero

DuckDB es un motor analítico embebido, lo que significa que no requiere la instalación previa de servidores complejos ni dependencias externas en el sistema operativo. Todo el ciclo de vida se gestiona directamente desde el entorno de ejecución de nuestra aplicación.

A continuación, se detallan los comandos necesarios para inicializar un proyecto limpio en Node.js utilizando TypeScript:

### 1. Inicializar el proyecto Node.js

Genera el archivo base de control de dependencias `package.json` en la raíz actual del repositorio:

```bash
npm init -y
```

### 2. Instalar el motor analítico y dependencias de desarrollo

Instala el paquete oficial de DuckDB, junto con el compilador de TypeScript y las herramientas para ejecutar el código directamente en la consola:

```bash
npm install duckdb typescript @types/node ts-node
```

### 3. Configuración del compilador TypeScript

Genera el archivo `tsconfig.json` para habilitar el tipado estricto y el uso de módulos modernos de JavaScript:

```bash
npx tsc --init
```

## 📊 Paso 2: Definición de la Estructura e Ingesta Analítica (ETL)

En las arquitecturas OLAP, la flexibilidad para absorber datos crudos es fundamental. DuckDB cuenta con la función nativa `read_csv_auto`, la cual escanea de forma directa los archivos existentes de nuestra carpeta data/, infiere los tipos de datos columnares (cadenas, enteros) y genera las estructuras internas de forma automática en una sola instrucción.

Crea el archivo principal `app.ts` en la raíz del proyecto e incorpora las instrucciones para levantar la base de datos persistente (`mundial2026.db`) e importar los CSV:

```ts
import { Database } from 'duckdb';
import * as path from 'path';

// Instanciación de la base de datos local persistente
const db = new Database('mundial2026.db');
const connection = db.connect();

connection.serialize(() => {
    console.log("================================================================");
    console.log("📥 Fase 1: Iniciando proceso de Ingesta Analítica (ETL)...");
    console.log("================================================================");

    // Resolución de rutas absolutas para los archivos de la carpeta /data existente
    const rutaSelecciones = path.join(__dirname, 'data', 'selecciones.csv');
    const rutaJugadores = path.join(__dirname, 'data', 'jugadores.csv');

    // Ingesta automática y estructuración de la tabla de Selecciones
    connection.run(`
        CREATE TABLE IF NOT EXISTS selecciones AS
        SELECT * FROM read_csv_auto('${rutaSelecciones.replace(/\\/g, '/')}');
    `, (err) => {
        if (err) console.error("❌ Error al procesar selecciones.csv:", err);
        else console.log("✅ Tabla 'selecciones' estructurada con éxito desde CSV.");
    });

    // Ingesta automática y estructuración de la tabla de Jugadores
    connection.run(`
        CREATE TABLE IF NOT EXISTS jugadores AS
        SELECT * FROM read_csv_auto('${rutaJugadores.replace(/\\/g, '/')}');
    `, (err) => {
        if (err) console.error("❌ Error al procesar jugadores.csv:", err);
        else console.log("✅ Tabla 'jugadores' estructurada con éxito desde CSV.");
    });
```

## 🧹 Paso 3: Operaciones CRUD Analíticas (Saneamiento de Datos)

En los flujos de Data Analytics, los datos crudos extraídos de fuentes externas suelen contener inconsistencias o valores nulos que alteran las proyecciones estadísticas. En este paso, aplicamos transformaciones lógicas (UPDATE y DELETE) utilizando el motor columnar de DuckDB para limpiar el dataset antes de ejecutar las agregaciones.

A diferencia de los motores orientados a filas (OLTP), que deben reconstruir registros completos en disco para modificar un solo atributo, DuckDB aísla únicamente los vectores de las columnas afectadas, acelerando la depuración masiva.

Agrega las siguientes funciones de limpieza en tu archivo `app.ts`:

```typescript
function ejecutarCRUDLimpieza() {
    console.log("🧹 Ejecutando Limpieza y Depuración (CRUD Analítico)...");

    // 1. UPDATE: Corregir inconsistencias de caracteres especiales en la columna 'nombre'
    connection.run(`
        UPDATE jugadores 
        SET nombre = 'Kylian Mbappe' 
        WHERE nombre = 'Kylian Mbappé' AND pais = 'Francia';
    `, (err) => {
        if (err) console.error("❌ Error en actualización:", err);
        else console.log("✅ Operación UPDATE completada sobre la columna estructurada.");
    });

    // 2. DELETE: Filtrar y eliminar registros con valores nulos en métricas críticas
    connection.run(`
        DELETE FROM jugadores 
        WHERE goles_clasificatoria IS NULL OR posicion = 'N/A';
    `, (err) => {
        if (err) console.error("❌ Error en borrado analítico:", err);
        else console.log("✅ Operación DELETE de depuración completada con éxito.");
    });
}

🏆 Paso 4: Consultas Analíticas (Proyecciones Estadísticas en Tiempo Real)

El núcleo estratégico del taller consiste en ejecutar consultas complejas utilizando funciones de agregación (SUM, AVG, GROUP BY, CAST) y ordenamientos masivos para dar respuesta en milisegundos a las tres interrogantes de investigación planteadas, aprovechando el procesamiento vectorial de DuckDB sobre el dataset del Mundial 2026.

Agrega las funciones de proyección al final de tu flujo de ejecución:

```javascript
function ejecutarConsultasAnaliticas() {
    console.log("🏆 Ejecutando Consultas de Agregación y Proyecciones...");

    // 1. Proyección de la Selección Potencial Ganadora
    connection.all(`
        SELECT pais, confederacion, ranking_fifa_abril2026 AS ranking, goles_favor, 
               CAST((partidos_ganados * 100 / partidos_jugados_clasificatoria) AS INTEGER) AS efectividad
        FROM selecciones
        WHERE partidos_jugados_clasificatoria > 0
        ORDER BY ranking_fifa_abril2026 ASC
        LIMIT 5;
    `, (err, rows) => {
        if (err) console.error(err);
        else console.log("⚽ Resultados de Proyección Colectiva:", rows);
    });

    // 2. Proyección del Ganador de la Bota de Oro
    connection.all(`
        SELECT nombre, pais, posicion, goles_clasificatoria 
        FROM jugadores
        WHERE posicion = 'Delantero'
        ORDER BY goles_clasificatoria DESC
        LIMIT 3;
    `, (err, rows) => {
        if (err) console.error(err);
        else console.log("🥇 Resultados de Proyección Bota de Oro:", rows);
    });

    // 3. Proyección del Ganador del Guante de Oro
    connection.all(`
        SELECT nombre, pais, partidos_invicto, goles_recibidos_portero, paradas_portero
        FROM jugadores
        WHERE posicion = 'Portero'
        ORDER BY partidos_invicto DESC, paradas_portero DESC
        LIMIT 3;
    `, (err, rows) => {
        if (err) console.error(err);
        else {
            console.log("🧤 Resultados de Proyección Guante de Oro:", rows);
            console.log("================================================================");
            console.log("✅ Taller evaluado con éxito. Infraestructura OLAP completada.");
            console.log("================================================================");
        }
    });
}
