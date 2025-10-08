ETL Test – Castor (SSIS + SQL Server)
1) Fuentes

data/sales.csv → fuente estructurada con ventas (fecha, cliente, producto, cantidad, precio, descuento, costo, canal)

data/customers.json → fuente semi-estructurada con clientes (contacto, ubicación, lealtad)

2) Base de datos y esquemas

Base: CastorDW

Esquemas:

stg → staging (datos intermedios y control de errores)

dw → modelo dimensional (datos finales para análisis)

3) Scripts (SQL Server)
Orden	Archivo	Descripción
1	00_create_db.sql	Crea la base de datos CastorDW
2	01_create_schemas.sql	Crea los esquemas stg y dw
3	02_create_tables.sql	Crea tablas stg.Sales, stg.Customers, stg.Sales_Errors, stg.AuditLog, stg.JsonRaw y el modelo dimensional dw.DimDate, dw.DimCustomer, dw.DimProduct, dw.FactSales
4	03_populate_dimdate.sql	Pobla la tabla dw.DimDate con fechas 2024–2026
5	load_json_raw.sql (nuevo)	Inserta el contenido de customers.json directamente a stg.JsonRaw usando OPENROWSET
6	05_openjson_to_stg_customers.sql	Aplana el JSON desde stg.JsonRaw hacia stg.Customers usando OPENJSON
7	04_load_from_stg.sql	Carga DimCustomer, DimProduct y FactSales (con control de duplicados)
4) Reglas de transformación

Fechas en formato ISO (YYYY-MM-DD)

Duplicados: eliminados por (order_id, product_code)

Campos obligatorios: order_id, customer_id, order_date

Filas con datos faltantes se registran en stg.Sales_Errors

Descuentos nulos (discount_rate) → 0

Columnas calculadas PERSISTED en dw.FactSales:
GrossAmount, DiscountAmt, NetAmount, TotalCost, Margin

Control de duplicados en FactSales mediante NOT EXISTS o MERGE
(puedes incluir un índice único si lo deseas)

5) SSIS (resumen de proceso)
Control Flow

DF_Load_Sales_CSV → carga stg.Sales desde sales.csv

Flat File Source → Derived Column → Conditional Split

Carga filas válidas → stg.Sales

Registra errores → stg.Sales_Errors

Execute SQL – Load_JSON_Raw → lee customers.json desde SQL con OPENROWSET e inserta en stg.JsonRaw

Execute SQL – OPENJSON_to_stg_Customers → aplana JSON a stg.Customers

Execute SQL – Load_Dims_and_Fact → inserta datos en dw con control de duplicados

Execute SQL – Validations_KPIs → valida conteos y muestra KPIs básicos

6) Validaciones
-- Conteos
SELECT COUNT(*) AS StgSales FROM stg.Sales;
SELECT COUNT(*) AS StgCustomers FROM stg.Customers;
SELECT COUNT(*) AS FactRows FROM dw.FactSales;

-- KPIs
SELECT dd.Year, dd.Month, SUM(NetAmount) AS Revenue, SUM(Margin) AS Margin
FROM dw.FactSales fs
JOIN dw.DimDate dd ON fs.DateKey = dd.DateKey
GROUP BY dd.Year, dd.Month
ORDER BY dd.Year, dd.Month;

7) Requisitos SQL

SQL Server 2016+ (compatibilidad nivel 130 o superior)

Permitir consultas ad-hoc:

EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'Ad Hoc Distributed Queries', 1; RECONFIGURE;


La ruta del JSON debe ser accesible por el servicio de SQL Server, por ejemplo:

C:\Castor_ETL_Test\data\customers.json

8) Estructura del proyecto
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
│   ├── 05_openjson_to_stg_customers.sql
│   └── load_json_raw.sql
│
├── ssis_project/
│   ├── Main.dtsx
│   └── Castor.PruebaETL.sln
│
├── docs/
│   ├── ETL_Flow_Castor.png
│   └── Model_ER_Castor.png
│
└── README.md


