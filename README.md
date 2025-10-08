# ETL Test – Castor (SSIS + SQL Server)

## 1) Fuentes
- data/sales.csv (estructurada): ventas (fecha, cliente, producto, cantidad, precio, descuento, costo, canal)
- data/customers.json (semi-estructurada): clientes (contacto, ubicación, lealtad)

## 2) Base de datos y esquemas
- Base: CastorDW
- Esquemas: stg (staging) y dw (modelo dimensional) — ambos dentro de CastorDW

## 3) Scripts (SQL Server)
1. sql/00_create_db.sql — crea CastorDW
2. sql/01_create_schemas.sql — crea stg y dw
3. sql/02_create_tables.sql — crea stg.Sales, stg.Customers, stg.Sales_Errors, stg.AuditLog, stg.JsonRaw; dw.DimDate, dw.DimCustomer, dw.DimProduct, dw.FactSales
4. sql/03_populate_dimdate.sql — rellena dw.DimDate (2024–2026)
5. sql/05_openjson_to_stg_customers.sql — aplanar JSON de stg.JsonRaw → stg.Customers
6. sql/04_load_from_stg.sql — MERGE DimCustomer/DimProduct e INSERT FactSales

## 4) Reglas de transformación
- Fechas ISO (YYYY-MM-DD)
- Duplicados en Sales por (order_id, product_code) → remover
- Nulos: discount_rate → 0; filas sin order_id/customer_id/fecha → rechazar (stg.Sales_Errors)
- Calculadas (en Fact): GrossAmount, DiscountAmt, NetAmount, TotalCost, Margin (columnas calculadas PERSISTED)

## 5) SSIS (resumen)
1) Cargar CSV → stg.Sales (limpieza, duplicados, errores)
2) Script Task C# → insertar JSON en stg.JsonRaw (var User::CustomersJsonPath)
3) OPENJSON → stg.Customers
4) MERGE/INSERT a DW (04_load_from_stg.sql)
5) Validaciones: conteos y KPIs (Revenue/Margin por mes)

## 6) Video (≤10 min)
- Cara + pantalla todo el tiempo
- Explicar: fuentes → SSIS (Data Flows + Script Task + OPENJSON) → carga DW → KPIs
