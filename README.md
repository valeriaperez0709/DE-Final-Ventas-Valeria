# Examen Final — Ingeniería de Datos

**Estudiante:** Valeria Pérez Álvarez  
**Proyecto:** Procesamiento de ventas diarias con dbt, DuckDB y Apache Airflow

## 1. Descripción del proyecto

Este proyecto implementa un flujo de ingeniería de datos para procesar información de ventas de una empresa de comercio electrónico.

El objetivo es generar un resumen diario de ventas y garantizar que la información utilizada esté actualizada antes de ejecutar las transformaciones.

Para ello se utilizan las siguientes tecnologías:

- **DuckDB:** almacenamiento y consulta de datos.
- **dbt:** transformación de datos, pruebas de calidad y validación de actualidad.
- **Apache Airflow:** orquestación de las tareas.
- **Docker:** ejecución del entorno.

## 2. Objetivo

Construir una tabla analítica llamada `analytics.ventas_diarias` que permita consultar el número de ventas, las unidades vendidas y los ingresos totales por fecha.

Adicionalmente, implementar un mecanismo de control que impida reconstruir el resumen cuando los datos de origen estén desactualizados.

## 3. Transformación de datos con dbt

Se desarrolló el modelo `ventas_diarias.sql`, que utiliza `ref('stg_ventas')` para obtener la información de ventas y realizar las siguientes operaciones:

- Contar el número de ventas por día.
- Sumar las unidades vendidas.
- Calcular los ingresos totales.
- Agrupar los resultados por fecha de venta.
- Conservar la precisión decimal de los ingresos.

### Resultados obtenidos

| Fecha de venta | Número de ventas | Unidades vendidas | Ingreso total |
|---|---:|---:|---:|
| 2026-01-10 | 2 | 3 | 46.00 |
| 2026-01-11 | 3 | 6 | 51.25 |
| 2026-01-12 | 1 | 4 | 12.40 |

Los resultados coinciden con los valores esperados para el ejercicio.

## 4. Control de actualidad de datos

Se configuró `dbt source freshness` sobre la fuente `ventas_raw.ventas`, utilizando la columna `_loaded_at`.

Se establecieron los siguientes umbrales:

| Antigüedad | Estado | Acción |
|---|---|---|
| Hasta 12 horas | Normal | Permitir construcción |
| Más de 12 y hasta 24 horas | Advertencia | Permitir construcción |
| Más de 24 horas | Error | Bloquear construcción |

Esta validación permite evitar que el resumen analítico sea reconstruido cuando la última carga de datos supera el tiempo máximo permitido.

## 5. Orquestación con Apache Airflow

Se configuró el DAG `examen_ventas` con dos tareas:

**Tarea 1: `verificar_actualidad`**

Ejecuta `dbt source freshness` para comprobar la antigüedad de los datos.

**Tarea 2: `construir_resumen`**

Ejecuta `dbt build` para construir el modelo analítico y comprobar las pruebas de calidad.

La dependencia implementada es:

`verificar_actualidad → construir_resumen`

La segunda tarea solamente se ejecuta cuando la primera termina correctamente.

## 6. Pruebas realizadas

### Escenario 1: Fuente actualizada

Se simuló una fuente con una última carga realizada aproximadamente dos horas antes de la ejecución.

**Resultados:**

- `verificar_actualidad`: success.
- `construir_resumen`: success.
- El resumen de ventas se construyó correctamente.

### Escenario 2: Fuente desactualizada

Se simuló una fuente con una última carga realizada aproximadamente 30 horas antes de la ejecución.

**Resultados:**

- `verificar_actualidad`: failed.
- `construir_resumen`: upstream_failed.
- La construcción del resumen fue bloqueada.

El flujo evita reconstruir la tabla analítica con una fuente que supera el límite de actualidad establecido.

## 7. Preguntas conceptuales

### ¿Por qué usar `_loaded_at` y no `fecha_venta` para este control?

Se utiliza `_loaded_at` porque indica el momento en que los datos fueron cargados al sistema, mientras que `fecha_venta` representa la fecha en que ocurrió la transacción.

Una venta antigua puede ser cargada recientemente, por lo que la fecha de venta no permite determinar si la fuente está actualizada.

La columna `_loaded_at` permite medir el tiempo transcurrido desde la última carga y compararlo con los umbrales definidos.

Sin embargo, verificar la última carga no garantiza por sí solo que se hayan recibido todas las ventas esperadas.

### ¿Por qué las pruebas de integridad no garantizan actualidad?

Las pruebas de integridad verifican que los datos cumplan condiciones de calidad, como unicidad, ausencia de valores nulos y validez de los registros.

Sin embargo, una tabla puede cumplir todas estas condiciones y contener información desactualizada.

Por eso, las pruebas de integridad deben complementarse con controles de actualidad como `dbt source freshness`.

## 8. Archivos desarrollados

Los archivos modificados para resolver el examen fueron:

- `dbt/models/ventas_diarias.sql`: construcción del resumen diario.
- `dbt/models/sources.yml`: configuración de freshness.
- `airflow/dags/examen_ventas.py`: orquestación y control de ejecución.

## 9. Evidencias

Las evidencias del proyecto incluyen:

- Ejecución exitosa del DAG con datos actualizados.
- Ejecución fallida de freshness con datos desactualizados.
- Bloqueo de la construcción del resumen.
- Consulta SQL con los resultados del modelo.
- Log de la validación de actualidad.

## 10. Conclusión

El proyecto demuestra la importancia de incorporar controles de calidad y actualidad en los procesos de ingeniería de datos.

La integración entre dbt y Apache Airflow permite automatizar la transformación de ventas y evitar que se reconstruyan modelos analíticos cuando la fuente no cumple los requisitos establecidos.

De esta manera, se mejora la confiabilidad de la información utilizada para tomar decisiones relacionadas con promociones, inventario y seguimiento comercial.
