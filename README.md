ETL Test – Castor (SSIS + SQL Server)
1) Arquitectura general

El proyecto se implementó en SQL Server Integration Services (SSIS) y SQL Server 2019, bajo una arquitectura de tres niveles:

Staging: recepción, validación y limpieza de datos.

DW (Data Warehouse): integración y carga de dimensiones y hechos.

Orquestador: control del flujo total de ejecución (de STG a DW).

2) Fuentes de datos

sales.csv → archivo estructurado con las ventas (order_id, fecha, cliente, producto, precio, cantidad, etc.)

customers.json → archivo semi-estructurado con la información de clientes (contacto, ubicación, programa de lealtad).

3) Base de datos

Nombre: CastorDW

Esquemas:

stg → staging temporal y control de errores.

dw → modelo dimensional final.

4) Estructura de carpetas del proyecto
Castor_ETL_Test/
│
├── data/
│   ├── sales.csv
│   └── customers.json
│
├── sql/
│   ├── 00_create_db.sql
│   ├── 01_create_schemas.sql
│   ├── 02_create_tables.sql
│   ├── 03_populate_dimdate.sql
│   ├── 04_load_from_stg.sql
│   ├── 05_2_openjson_to_stg_customers.sql
│   ├── CastorDW.bak
│   ├── Script_CastorDW.sql
│   └── ScriptTask_LoadJson.cs

│
├── ssis_project/
│   ├── Cargar_STG.dtsx
│   ├── Cargar_DW.dtsx
│   ├── Orquestador.dtsx
│   └── Castor.sln
│
├── docs/
│   ├── diagrama CastorDW.png
│   └── Model_ER_Castor.png
│
└── README.md

6) Procesos implementados
Paquete 1 – Cargar_STG.dtsx

Encargado de la extracción y preparación de datos en el entorno staging.

Flujo principal

Carga de datos SalesCsv (Data Flow)

Fuente: Flat File Source (sales.csv).

Transformaciones: Derived Column, Data Conversion, Conditional Split, error handling.

Destinos:

stg.Sales → registros válidos.

stg.Sales_Errors → registros rechazados o con campos obligatorios nulos.

Contenedor “Carga datos JSON” (Sequence Container)

Tarea 1: exec cargaJsonRaw

Ejecuta el SP dbo.cargaJsonRaw que lee el archivo customers.json directamente desde SQL Server usando OPENROWSET y lo inserta en stg.JsonRaw.

La ruta se pasa como parámetro @Path desde SSIS.

Tarea 2: TRUNCATE TABLE stg.Customers (o EXEC stg.truncate_Customers).

Tarea 3: exec stg.openjson_to_Customers

Ejecuta el SP que usa OPENJSON para aplanar el contenido del JSON hacia stg.Customers.

Paquete 2 – Cargar_DW.dtsx

Encargado de poblar las tablas del modelo dimensional con lógica incremental.

Flujo principal

exec cargar_DimCustomer

Hace MERGE entre stg.Customers y dw.DimCustomer.

exec cargar_DimProduct

Carga incremental en dw.DimProduct.

exec cargar_FactSales

Inserta nuevos registros en dw.FactSales, evitando duplicados mediante NOT EXISTS.

Paquete 3 – Orquestador.dtsx

Controla el flujo global del proceso.

Flujo principal

Ejecutar STG → llama al paquete Cargar_STG.dtsx

Ejecutar DW → llama al paquete Cargar_DW.dtsx
Ambas tareas se enlazan por precedencia On Success.

6) Procedimientos almacenados
Procedimiento	Descripción
dbo.cargaJsonRaw	Carga el archivo JSON a stg.JsonRaw usando OPENROWSET.
stg.truncate_Customers	Limpia la tabla staging de clientes.
stg.openjson_to_Customers	Aplana los datos del JSON y los inserta en stg.Customers.
dbo.cargar_DimCustomer	MERGE incremental en la dimensión de clientes.
dbo.cargar_DimProduct	MERGE incremental en la dimensión de productos.
dbo.cargar_FactSales	Inserta nuevas transacciones en el hecho FactSales evitando duplicados.
7) Reglas de negocio y transformaciones

Filas con order_id, customer_id o order_date nulos → rechazadas a stg.Sales_Errors.

Descuentos nulos (discount_rate) → reemplazados por 0.

Duplicados (order_id + product_code) → eliminados antes de la carga.

Medidas calculadas persistidas (PERSISTED) en dw.FactSales:

GrossAmount = Quantity * UnitPrice

DiscountAmt = GrossAmount * DiscountRate

NetAmount = GrossAmount - DiscountAmt

TotalCost = Quantity * UnitCost

Margin = NetAmount - TotalCost

8) Control de duplicados

En el proceso cargar_FactSales se evita insertar registros ya existentes mediante NOT EXISTS.

Además, se sugiere crear un índice único para reforzar la integridad:

CREATE UNIQUE INDEX UX_FactSales_BK
ON dw.FactSales(DateKey, CustomerKey, ProductKey, Channel, Quantity, UnitPrice, DiscountRate, UnitCost);

9) Validaciones finales
-- Conteos por etapa
SELECT COUNT(*) AS StgSales FROM stg.Sales;
SELECT COUNT(*) AS StgCustomers FROM stg.Customers;
SELECT COUNT(*) AS FactRows FROM dw.FactSales;

-- KPIs finales
SELECT dd.Year, dd.Month, SUM(NetAmount) AS Revenue, SUM(Margin) AS Margin
FROM dw.FactSales fs
JOIN dw.DimDate dd ON fs.DateKey = dd.DateKey
GROUP BY dd.Year, dd.Month
ORDER BY dd.Year, dd.Month;

10) Ejecución del proyecto

Abre Orquestador.dtsx en SSIS.

Configura los parámetros de conexión y el @Path del JSON.

Ejecuta el paquete completo.

Verifica logs, conteos y KPIs finales.

11) Requisitos técnicos

SQL Server 2016+ con Ad Hoc Distributed Queries habilitado.

Permisos de lectura sobre la ruta del archivo JSON para el servicio SQL Server.

Nivel de compatibilidad ≥ 130.

12) Video de entrega

 Duración máxima: 10 minutos
Debe mostrar:

Tu rostro y pantalla completa.

Explicación del flujo completo (STG → DW → Orquestador).

Justificación técnica de cada componente (transformaciones, SPs, control de errores).

Evidencia de la carga correcta y consultas de validación.
