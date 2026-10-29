# 📊 Corporate Financial Intelligence: P&L Statement & Budget Variance Analysis

[![SQL Server](https://img.shields.io/badge/Database-SQL%20Server%202022-CC292B?logo=microsoftsqlserver&logoColor=white)](https://www.microsoft.com/sql-server)
[![Docker](https://img.shields.io/badge/Container-Docker%20Engine-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
[![Ubuntu](https://img.shields.io/badge/OS-Ubuntu%20Server%20(KVM)-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Power BI](https://img.shields.io/badge/BI-Power%20BI%20Desktop-F2C811?logo=powerbi&logoColor=black)](https://powerbi.microsoft.com/)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 📌 Executive Summary

Este proyecto implementa una solución de **Inteligencia Financiera y Control Presupuestal (FP&A)** para una empresa multinacional manufacturera (*Adventure Works Cycles*). A diferencia de los modelos analíticos convencionales basados en hojas de cálculo estáticas o tablas planas recortadas, esta arquitectura se diseñó desde cero utilizando infraestructura corporativa real sobre virtualización Linux, contenedores Docker y un motor de base de datos relacional **Microsoft SQL Server 2022**.

La solución transforma millones de registros transaccionales del Libro Mayor (*General Ledger*) en un **modelo analítico en estrella** optimizado, resolviendo la jerarquía contable del **Estado de Resultados (P&L)** y habilitando una auditoría precisa de desviaciones presupuestales (*Actual vs. Budget Variance*) por centro de costos y cuenta operativa.

---

## 🏗️ Technical Architecture & Infrastructure

El flujo de ingeniería de datos desacopla por completo la capa de almacenamiento transaccional de la capa semántica de consumo analítico:

```
[ KVM / Ubuntu Server ]
       │
       ▼
[ Docker Container ] ──► [ SQL Server 2022 Engine ] ──► [ AdventureWorksDW2022 ]
                                                              │
                                                              ▼
                                                    [ Semantic Layer (SQL Views) ]
                                                              │
                                                              ▼
                                                    [ Power BI Tabular Model ]
                                                    (Star Schema + DAX Measures)
```

### 1. Entorno de Virtualización y Motor Relacional
* **Host & Virtualización:** Servidor virtualizado con **Ubuntu Server** sobre hipervisor KVM.
* **Contenedorización:** Contenedor oficial `mcr.microsoft.com/mssql/server:2022-latest` desplegado en Docker, configurado con políticas de reinicio automático (`unless-stopped`) y persistencia mediante volúmenes.
* **Seguridad y Acceso Remoto:** Gestión de accesos remotos y consultas de ingeniería de datos a través de JetBrains DataGrip mediante autenticación TCP/IP en el puerto `1433`.

### 2. Capa Semántica en SQL (Vistas de Desacoplamiento)
Para prevenir dependencias circulares en Power Query y garantizar compresión columnar óptima en el motor VertiPaq, se implementaron vistas SQL limpias para la tabla de hechos y cada dimensión analítica:

```sql
-- 1. Tabla de Hechos Financieros
CREATE OR ALTER VIEW vw_FactFinance AS
SELECT 
    FinanceKey,
    DateKey,
    OrganizationKey,
    DepartmentGroupKey,
    ScenarioKey,
    AccountKey,
    Amount
FROM FactFinance;

-- 2. Dimensión Plan de Cuentas (Recursividad Padre-Hijo)
CREATE OR ALTER VIEW vw_DimAccount AS
SELECT 
    AccountKey,
    ParentAccountKey,
    AccountCodeAlternateKey AS AccountCode,
    AccountDescription,
    AccountType,
    Operator,
    CustomMembers
FROM DimAccount;

-- 3. Dimensión Centros de Costo / Departamentos
CREATE OR ALTER VIEW vw_DimDepartmentGroup AS
SELECT 
    DepartmentGroupKey,
    DepartmentGroupName
FROM DimDepartmentGroup;

-- 4. Dimensión Escenarios (Real vs Presupuesto)
CREATE OR ALTER VIEW vw_DimScenario AS
SELECT 
    ScenarioKey,
    ScenarioName
FROM DimScenario;

-- 5. Dimensión Calendario
CREATE OR ALTER VIEW vw_DimDate AS
SELECT 
    DateKey,
    FullDateAlternateKey AS [Date],
    EnglishDayNameOfWeek AS DayName,
    EnglishMonthName AS MonthName,
    MonthNumberOfYear AS MonthNumber,
    CalendarQuarter AS Quarter,
    CalendarYear AS [Year]
FROM DimDate;
```

---

## 📐 Data Modeling (Star Schema)

El modelo en Power BI Desktop sigue estrictamente la metodología Kimball de esquema en estrella, eliminando relaciones bidireccionales y dependencias transitivas:

* **Fact Table:** `vw_FactFinance`
* **Dimension Tables:** 
  * `vw_DimAccount` (enlace $1:*$ vía `AccountKey`)
  * `vw_DimScenario` (enlace $1:*$ vía `ScenarioKey`)
  * `vw_DimDepartmentGroup` (enlace $1:*$ vía `DepartmentGroupKey`)
  * `vw_DimDate` (enlace $1:*$ vía `DateKey`)

Para resolver la visualización jerárquica del catálogo de cuentas padre-hijo sin desnormalizar la tabla de hechos, se estructuró la jerarquía contable en DAX:

```dax
AccountLevel1 = 
IF(
    ISBLANK(vw_DimAccount[ParentAccountKey]),
    vw_DimAccount[AccountDescription],
    LOOKUPVALUE(
        vw_DimAccount[AccountDescription],
        vw_DimAccount[AccountKey],
        vw_DimAccount[ParentAccountKey]
    )
)
```

```dax
AccountLevel2 = vw_DimAccount[AccountDescription]
```

---

## 🧮 Financial Calculations & DAX Formulation

Las medidas analíticas fueron estructuradas de forma atómica y modular para garantizar trazabilidad y rendimiento en el motor de cálculo:

### 1. Medidas Atómicas de Escenario
```dax
Actual = 
CALCULATE(
    SUM(vw_FactFinance[Amount]),
    vw_DimScenario[ScenarioName] = "Actual"
)
```

```dax
Budget = 
CALCULATE(
    SUM(vw_FactFinance[Amount]),
    vw_DimScenario[ScenarioName] = "Budget"
)
```

### 2. Medidas de P&L (Estado de Resultados)
```dax
Total Revenues = 
CALCULATE(
    [Actual],
    vw_DimAccount[AccountType] = "Revenue"
)
```

```dax
Total Expenditures = 
CALCULATE(
    [Actual],
    vw_DimAccount[AccountType] = "Expenditures"
)
```

```dax
Operating Profit = [Total Revenues] - [Total Expenditures]
```

```dax
Operating Margin % = 
DIVIDE([Operating Profit], [Total Revenues], 0)
```

### 3. Medidas de Control Presupuestal (FP&A)
```dax
Variance ($) = [Actual] - [Budget]
```

```dax
Variance (%) = 
DIVIDE([Variance ($)], [Budget], 0)
```

---

## 🖥️ Dashboard Architecture & Visual Design

El tablero directivo se diseñó con un enfoque ejecutivo minimalista, empleando contenedores flotantes con bordes redondeados y una paleta sobria corporativa basada en la identidad visual de la marca.

### Página 1: Estado de Resultados (P&L Jerárquico)
* **Objetivo:** Monitorear el rendimiento operativo neto y la estructura de márgenes de la empresa.
* **Componentes Principales:**
  * **Fila de KPIs Superiores:** Monitoreo instantáneo de `Total Revenues ($108.76M)`, `Total Expenditures ($88.08M)`, `Operating Profit ($20.68M)` y `Operating Margin % (19.01%)`.
  * **Matriz Contable Dinámica:** Estructurada jerárquicamente por `AccountType` $\rightarrow$ `AccountLevel1` $\rightarrow$ `AccountLevel2` a lo largo de los ejercicios fiscales (`2010` - `2013`), excluyendo cuentas patrimoniales del Balance General para reflejar utilidad contable pura.
  * **Segmentación Temporal:** Control dinámico de ejercicio fiscal anual.

### Página 2: Control Presupuestal y Análisis de Desviaciones (Actual vs. Budget)
* **Objetivo:** Auditar la ejecución presupuestal e identificar sobrecostos departamentales.
* **Componentes Principales:**
  * **KPIs de Auditoría:** Resumen de gasto ejecutado (`Actual`), presupuesto aprobado (`Budget`), desviación nominal (`Variance $`) y desviación porcentual (`Variance %`).
  * **Ranking de Desviación por Departamento:** Gráfico de barras horizontales que identifica de inmediato las unidades con mayor desvío absoluto (liderado por *Research and Development* y *Corporate*).
  * **Matriz Detallada de Centros de Costo:** Auditoría tabular que cruza el presupuesto asignado contra el gasto real por grupo operativo.
  * **Tendencia Temporal de Gasto:** Curva analítica de evolución interanual para auditar el ritmo de consumo presupuestal.

---

## 🚀 How to Clone This Repository
    git clone https://github.com/ismaellopez8933-boop/Corporate-Financial-Intelligence.git
