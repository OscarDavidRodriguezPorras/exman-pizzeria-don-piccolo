# 🍕 Pizzería Don Piccolo - Sistema de Gestión de Base de Datos (MySQL)

Este repositorio contiene la arquitectura, esquemas DDL/DML, funciones, procedimientos almacenados, disparadores (triggers), vistas y consultas analíticas de la base de datos relacional para **Pizzería Don Piccolo**.

---

## 📋 Tabla de Contenidos
- [📌 Descripción del Proyecto](#-descripción-del-proyecto)
- [🗄️ Explicación de las Tablas y Relaciones](#️-explicación-de-las-tablas-y-relaciones)
  - [Diagrama Conceptual de Relaciones](#diagrama-conceptual-de-relaciones)
  - [Detalle de las Tablas](#detalle-de-las-tablas)
- [⚡ Componentes Programables](#-componentes-programables)
  - [Funciones](#funciones)
  - [Procedimientos Almacenados](#procedimientos-almacenados)
  - [Triggers (Disparadores)](#triggers-disparadores)
  - [Vistas](#vistas)
- [🔍 Ejemplos de Consultas](#-ejemplos-de-consultas)
- [🚀 Instrucciones para Ejecutar el Script](#-instrucciones-para-ejecutar-el-script)

---

## 📌 Descripción del Proyecto

El proyecto **Pizzería Don Piccolo** es un sistema de gestión relacional diseñado para centralizar y automatizar la operación diaria de una pizzería comercial. Permite registrar y administrar:

1. **Gestión de Clientes y Pedidos:** Control de información de contacto, métodos de pago y estados de procesamiento de pedidos.
2. **Control de Inventario e Ingredientes:** Gestión del stock actual y mínimo con alertas automatizadas.
3. **Descuento de Stock por Receta:** Relación M:N entre pizzas e ingredientes con cálculo de insumos consumidos.
4. **Logística y Domicilios:** Asignación de repartidores por zona, control de tiempos de salida/entrega y cambio automático de disponibilidad de personal.
5. **Auditoría y Facturación:** Cálculo automático de montos con impuestos (IVA 19%), costo de envío e historial de cambios de precios de productos.

---

## 🗄️ Explicación de las Tablas y Relaciones

### Diagrama Conceptual de Relaciones

```text
+-----------------+       1:N       +-----------------+       1:N       +-------------------+
|    CLIENTES     |<--------------->|     PEDIDOS     |<--------------->|  DETALLE_PEDIDOS  |
+-----------------+                 +-----------------+                 +-------------------+
                                             |                                    |
                                    1:1      |                            N:1     |
                                    +--------+--------+                  +--------+--------+
                                    |                 |                  |                 |
                                    v                 v                  v                 v
                            +---------------+ +---------------+  +---------------+ +---------------+
                            |   DOMICILIOS  | |     PAGOS     |  |    PIZZAS     | | INGREDIENTES  |
                            +---------------+ +---------------+  +---------------+ +---------------+
                                    |                                    |                 ^
                                    | N:1                                | 1:N             |
                                    v                                    v                 | N:M
                            +---------------+                    +---------------+         |
                            | REPARTIDORES  |                    |HISTORIAL_PRECIOS|       |
                            +---------------+                    +---------------+---------+
                                                                         |
                                                                         v
                                                             +-----------------------+
                                                             |  PIZZA_INGREDIENTES   |
                                                             +-----------------------+
```

### Detalle de las Tablas

1. **`clientes`**: Almacena los datos personales de los clientes registrados.
   - **Campos principales:** `id_cliente` (PK), `nombre`, `telefono`, `direccion`, `email` (UNIQUE), `fecha_registro`.
2. **`ingredientes`**: Gestiona el catálogo de insumos y el control de inventario.
   - **Campos principales:** `id_ingrediente` (PK), `nombre`, `stock_actual`, `stock_minimo`, `costo_unitario`.
3. **`pizzas`**: Catálogo de pizzas con sus especificaciones y precios base.
   - **Campos principales:** `id_pizza` (PK), `nombre`, `tamano` (ENUM), `precio_base`, `tipo` (ENUM).
4. **`pizza_ingredientes`** *(Tabla intermedia N:M)*: Define la receta de cada pizza especificando la cantidad necesaria de cada ingrediente.
   - **Campos principales:** `(id_pizza, id_ingrediente)` (PK compuesta), `cantidad_requerida`.
   - **Relaciones:** FK a `pizzas` y `ingredientes` (con `ON DELETE CASCADE`).
5. **`repartidores`**: Registra al personal de entrega, su zona asignada y su disponibilidad.
   - **Campos principales:** `id_repartidor` (PK), `nombre`, `zona_asignada`, `estado` (`disponible`, `ocupado`).
6. **`pedidos`**: Tabla central que encabeza cada transacción de venta.
   - **Campos principales:** `id_pedido` (PK), `id_cliente` (FK), `fecha_hora`, `metodo_pago`, `estado`, `total`.
7. **`detalle_pedidos`** *(Tabla intermedia N:M)*: Desglosa las pizzas y cantidades incluidas en cada pedido.
   - **Campos principales:** `id_detalle` (PK), `id_pedido` (FK), `id_pizza` (FK), `cantidad`, `precio_unitario`.
8. **`domicilios`**: Almacena los datos logísticos de entrega del pedido *(Relación 1:1 con `pedidos`)*.
   - **Campos principales:** `id_domicilio` (PK), `id_pedido` (FK UNIQUE), `id_repartidor` (FK), `hora_salida`, `hora_entrega`, `distancia_km`, `costo_envio`.
9. **`pagos`**: Gestiona el registro contable e independientes de los pagos procesados por pedido.
   - **Campos principales:** `id_pago` (PK), `id_pedido` (FK), `metodo_pago`, `monto`, `estado_pago`, `referencia_transaccion`, `fecha_pago`.
10. **`historial_precios`**: Bitácora de auditoría para rastrear variaciones de precios en las pizzas.
    - **Campos principales:** `id_historial` (PK), `id_pizza` (FK), `precio_anterior`, `precio_nuevo`, `fecha_cambio`.

---

## ⚡ Componentes Programables

### Funciones
- **`fn_calcular_total_pedido(p_id_pedido)`**: Calcula el total a pagar sumando los ítems del detalle y el costo de envío de domicilios, aplicando el 19% de IVA.
- **`fn_ganancia_neta_diaria(p_fecha)`**: Devuelve el margen neto del día seleccionado (Ventas Totales menos Costos Directos de Ingredientes consumidos).

### Procedimientos Almacenados
- **`sp_registrar_entrega_domicilio(p_id_domicilio, p_hora_entrega)`**: Actualiza la hora de entrega del domicilio y marca el pedido correspondiente como `'entregado'`.

### Triggers (Disparadores)
- **`trg_descontar_stock`**: Tras insertar un ítem en `detalle_pedidos`, descuenta automáticamente del inventario la cantidad de ingredientes correspondiente según la receta (`pizza_ingredientes`).
- **`trg_auditoria_precios`**: Captura los cambios en `precio_base` dentro de la tabla `pizzas` e inserta el registro histórico en `historial_precios`.
- **`trg_ocupar_repartidor`**: Cambia el estado del repartidor a `'ocupado'` al crear un domicilio.
- **`trg_liberar_repartidor`**: Cambia el estado del repartidor a `'disponible'` cuando se registra la `hora_entrega`.
- **`trg_actualizar_total_detalle` / `trg_actualizar_total_domicilio`**: Recalculan automáticamente el valor total del pedido invocando la función `fn_calcular_total_pedido()`.

### Vistas
- **`vw_resumen_pedidos_cliente`**: Resume la cantidad de pedidos y el acumulado gastado por cliente.
- **`vw_desempeno_repartidores`**: Métricas de entregas realizadas y promedio en minutos del tiempo de entrega por repartidor.
- **`vw_ingredientes_bajo_stock`**: Muestra los ingredientes cuyo stock actual se encuentra por debajo del stock mínimo requerido.

---

## 🔍 Ejemplos de Consultas

A continuación se presentan las consultas SQL incluidas en el proyecto para análisis de datos y reportes:

### 1. Clientes activos en un periodo determinado (Agosto 2026)
```sql
SELECT DISTINCT c.id_cliente, c.nombre, c.telefono, c.email 
FROM clientes c 
JOIN pedidos p ON c.id_cliente = p.id_cliente
WHERE p.fecha_hora BETWEEN '2026-08-01 00:00:00' AND '2026-08-31 23:59:59';
```

### 2. Ranking de pizzas más vendidas (Unidades y Frecuencia)
```sql
SELECT p.nombre AS pizza, p.tamano, COUNT(dp.id_pizza) AS veces_pedida, SUM(dp.cantidad) AS total_unidades_vendidas 
FROM detalle_pedidos dp
JOIN pizzas p ON dp.id_pizza = p.id_pizza 
GROUP BY p.id_pizza, p.nombre, p.tamano 
ORDER BY total_unidades_vendidas DESC;
```

### 3. Seguimiento de repartidores y domicilios asignados
```sql
SELECT r.nombre AS repartidor, r.zona_asignada, p.id_pedido, p.fecha_hora, p.estado AS estado_pedido 
FROM domicilios d 
JOIN repartidores r ON d.id_repartidor = r.id_repartidor
JOIN pedidos p ON d.id_pedido = p.id_pedido;
```

### 4. Tiempo promedio de entrega en minutos por zona geográfica
```sql
SELECT r.zona_asignada, AVG(TIME_TO_SEC(TIMEDIFF(d.hora_entrega, d.hora_salida)) / 60) AS promedio_minutos
FROM domicilios d 
JOIN repartidores r ON d.id_repartidor = r.id_repartidor 
WHERE d.hora_entrega IS NOT NULL 
GROUP BY r.zona_asignada;
```

### 5. Clientes VIP (Gasto total acumulado mayor a $50.000)
```sql
SELECT c.id_cliente, c.nombre AS cliente, SUM(p.total) AS total_gastado 
FROM clientes c 
JOIN pedidos p ON c.id_cliente = p.id_cliente
GROUP BY c.id_cliente, c.nombre 
HAVING SUM(p.total) > 50000;
```

### 6. Buscador de pizzas por variedad o ingrediente (Hawaiana)
```sql
SELECT id_pizza, nombre, tamano, precio_base, tipo 
FROM pizzas 
WHERE nombre LIKE '%hawaiana%';
```

### 7. Clientes frecuentes del mes actual (Más de 5 pedidos en el mes)
```sql
SELECT id_cliente, nombre, telefono, email 
FROM clientes 
WHERE id_cliente IN (
    SELECT id_cliente 
    FROM pedidos 
    WHERE MONTH(fecha_hora) = MONTH(CURRENT_DATE()) 
      AND YEAR(fecha_hora) = YEAR(CURRENT_DATE()) 
    GROUP BY id_cliente 
    HAVING COUNT(id_pedido) > 5
);
```

---

## 🚀 Instrucciones para Ejecutar el Script

### Requisitos Previos
- MySQL Server 8.0 o superior (o MariaDB 10.5+).
- Cliente de gestión SQL (MySQL Workbench, DBeaver, phpMyAdmin o Terminal MySQL).

### Paso a Paso para la Instalación

#### Opción A: Desde MySQL Workbench / DBeaver / Entorno Gráfico
1. Abre tu cliente SQL y conéctate a tu servidor MySQL.
2. Abre un nuevo editor SQL (`File -> Open SQL Script...` o abre una nueva pestaña de query).
3. Pega los bloques del proyecto en el siguiente orden secuencial:
   - **Paso 1:** Estructura DDL (Creación de Base de Datos y Tablas) + Inserción de Datos Iniciales (DML).
   - **Paso 2:** Declaración de Funciones y Procedimientos Almacenados (`fn_calcular_total_pedido`, `fn_ganancia_neta_diaria`, `sp_registrar_entrega_domicilio`).
   - **Paso 3:** Declaración de Triggers (`trg_descontar_stock`, `trg_auditoria_precios`, etc.).
   - **Paso 4:** Declaración de Vistas (`vw_resumen_pedidos_cliente`, etc.).
   - **Paso 5:** Ejecución de las Consultas SQL de prueba.
4. Ejecuta todo el script (`Ctrl + Shift + Enter` en Workbench).

#### Opción B: Desde la Línea de Comandos (Terminal / CMD / Bash)
Si guardas todo el script en un archivo único llamado `schema_pizzeria.sql`:

```bash
# 1. Iniciar sesión en MySQL
mysql -u tu_usuario -p

# 2. O ejecutar directamente la importación del archivo .sql
mysql -u tu_usuario -p < schema_pizzeria.sql
```

#### Verificación de la Instalación
Una vez ejecutado el script, puedes verificar que todo se haya creado correctamente corriendo los siguientes comandos:

```sql
USE pizzeria_don_piccolo;

-- Verificar tablas
SHOW TABLES;

-- Verificar vistas
SHOW FULL TABLES WHERE Table_type = 'VIEW';

-- Verificar procedimientos y funciones
SHOW PROCEDURE STATUS WHERE Db = 'pizzeria_don_piccolo';
SHOW FUNCTION STATUS WHERE Db = 'pizzeria_don_piccolo';

-- Verificar triggers
SHOW TRIGGERS FROM pizzeria_don_piccolo;
```

---
*Desarrollado para la gestión eficiente de **Pizzería Don Piccolo**.* 🍕
