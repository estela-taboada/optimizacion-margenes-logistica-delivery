# 📊 Optimización de Márgenes Netos y Reducción de Fricción Logística en Delivery Gastronómico

![Python](https://img.shields.io/badge/Python-Pandas%20%7C%20SQLite3-blue)
![SQL](https://img.shields.io/badge/SQL-Analytics-orange)
![Power BI](https://img.shields.io/badge/Power%20BI-DAX%20%7C%20Dashboard-yellow)

## 📌 Planteamiento del Caso de Negocio
Una cadena/plataforma de restaurantes multi-ciudad opera bajo diversos canales de venta (`order_channel`), niveles de precio (`restaurant_price_tier`) y estrategias promocionales (`discount_percentage`). Aunque el volumen de pedidos crece, la rentabilidad neta por restaurante (`estimated_restaurant_net_margin_usd`) presenta alta volatilidad y las calificaciones (`customer_rating`) sufren caídas críticas en escenarios operativos específicos.

### 🎯 Mandato Ejecutivo
1. **Detener la erosión de margen:** Identificar qué promociones, canales y rangos de distancia generan pérdida neta por pedido (*"vender más para perder dinero"*).
2. **Optimizar el cuello de botella cocina–reparto:** Cuantificar cómo el tiempo de preparación (`estimated_prep_time_min`) y la fricción logística (`logistics_friction_index`) impactan la calificación del cliente en horas pico (`is_peak_hour`) y mal clima (`weather_condition`).
3. **Focalizar la inversión comercial:** Determinar qué categorías gastronómicas (`cuisine_category`) aportan rentabilidad sostenible sin depender de subsidios excesivos.

---

## 🛠️ Arquitectura del Proyecto (End-to-End)
1. **Python (Pandas):** Ingesta, auditoría de calidad de datos (nulos y tipos) y cálculo de KPIs base sobre **8,000 órdenes**.
2. **SQL (SQLite):** Ejecución de 4 consultas analíticas para aislar órdenes en pérdida, cuantificar el efecto de clima/hora pico y evaluar categorías de cocina, generando el dataset transformado `delivery_clean_powerbi.csv`.
3. **Power BI & DAX:** Modelado de medidas de negocio (`_Medidas_Negocio`) y diseño de un reporte ejecutivo interactivo de 2 páginas con segmentación multicanal.

---

## 🔍 Fase 1: Auditoría de Calidad y Línea Base (Python)

```python
import pandas as pd
import numpy as np

# 1. Carga de datos
df = pd.read_csv('global_food_delivery_enriched_market_2026.csv') 

# 2. Auditoría de calidad y valores nulos
nulos = df.isnull().sum()
print(nulos[nulos > 0] if nulos.sum() > 0 else "Cero valores nulos: Dataset limpio para análisis.")

# 3. Radiografía Financiera y Operativa
ventas_brutas = df['order_value_usd'].sum()
ventas_netas = df['net_effective_order_price'].sum()
margen_total = df['estimated_restaurant_net_margin_usd'].sum()
ordenes_perdida = (df['estimated_restaurant_net_margin_usd'] <= 0).sum()
pct_perdida = (ordenes_perdida / len(df)) * 100
```
Hallazgos Iniciales
Volumen y Limpieza: 8,000 órdenes auditadas a través de 26 variables con 0% de valores nulos.

Salud Financiera: Ventas Brutas de $241,782.71 USD, Ventas Netas Efectivas de $238,854.63 USD y Margen Neto Total de $86,318.38 USD (36.14% sobre venta neta).

Foco Rojo: 91 órdenes (1.14%) operan con margen cero o negativo.

Salud Operativa: Calificación promedio de 4.28 / 5.0 y tiempo total promedio (cocina + entrega) de 29.3 min.

🗄️ Fase 2: Diagnóstico Quirúrgico de Fugas y Fricción (SQL)
Se estructuraron 4 consultas SQL en memoria (sqlite3) para responder preguntas críticas de negocio:

```SQL
-- QUERY 1: Auditoría de Rentabilidad y Órdenes en Pérdida por Segmento de Precio y Canal
SELECT 
    restaurant_price_tier AS segmento_precio,
    order_channel AS canal,
    COUNT(order_id) AS total_ordenes,
    ROUND(AVG(net_effective_order_price), 2) AS ticket_neto_prom,
    ROUND(AVG(estimated_restaurant_net_margin_usd), 2) AS margen_neto_prom,
    SUM(CASE WHEN estimated_restaurant_net_margin_usd <= 0 THEN 1 ELSE 0 END) AS ordenes_en_perdida,
    ROUND(100.0 * SUM(CASE WHEN estimated_restaurant_net_margin_usd <= 0 THEN 1 ELSE 0 END) / COUNT(order_id), 2) AS pct_perdida
FROM delivery_orders
GROUP BY restaurant_price_tier, order_channel
ORDER BY ordenes_en_perdida DESC, margen_neto_prom ASC;

-- QUERY 2: Impacto Operativo de Hora Pico y Clima en Tiempos y Satisfacción
SELECT 
    is_peak_hour AS hora_pico,
    weather_condition AS clima,
    COUNT(order_id) AS ordenes,
    ROUND(AVG(estimated_prep_time_min), 1) AS prep_cocina_min,
    ROUND(AVG(actual_delivery_time_min), 1) AS reparto_min,
    ROUND(AVG(total_fulfillment_time_min), 1) AS tiempo_total_min,
    ROUND(AVG(logistics_friction_index), 2) AS indice_friccion,
    ROUND(AVG(customer_rating), 2) AS rating_prom
FROM delivery_orders
GROUP BY is_peak_hour, weather_condition
ORDER BY rating_prom ASC;

-- QUERY 3: Desempeño Comercial y Logístico por Categoría de Cocina
SELECT 
    cuisine_category AS categoria_cocina,
    COUNT(order_id) AS ordenes,
    ROUND(SUM(estimated_restaurant_net_margin_usd), 2) AS margen_total_usd,
    ROUND(AVG(estimated_restaurant_net_margin_usd), 2) AS margen_por_orden,
    ROUND(AVG(discount_percentage), 1) AS descuento_prom_pct,
    ROUND(AVG(total_fulfillment_time_min), 1) AS tiempo_total_min,
    ROUND(AVG(customer_rating), 2) AS rating_prom
FROM delivery_orders
GROUP BY cuisine_category
ORDER BY margen_total_usd DESC;

-- QUERY 4: Autopsia de Órdenes en Pérdida ($ Budget + Online Delivery App)
SELECT 
    CASE WHEN estimated_restaurant_net_margin_usd <= 0 THEN 'En Pérdida (<= $0)' ELSE 'Rentable (> $0)' END AS estado_margen,
    COUNT(order_id) AS ordenes,
    ROUND(AVG(order_value_usd), 2) AS venta_bruta_prom,
    ROUND(AVG(discount_percentage), 1) AS descuento_prom_pct,
    ROUND(AVG(delivery_distance_km), 2) AS distancia_prom_km,
    ROUND(AVG(delivery_fee_ratio), 2) AS ratio_tarifa_entrega,
    ROUND(AVG(estimated_restaurant_net_margin_usd), 2) AS margen_neto_prom
FROM delivery_orders
WHERE restaurant_price_tier = '$ Budget' 
  AND order_channel = 'Online Delivery App'
GROUP BY estado_margen;
```

## *Resultados del Diagnóstico SQL*

Culpable único de pérdidas (SQL 1 y 4): El 96.7% de las órdenes en pérdida (88 de 91) se concentran en Restaurantes $ Budget vendiendo por Online Delivery App. Se trata de micro-tickets subsidiados con venta bruta promedio de $3.60 USD (vs. $15.13 USD en órdenes rentables), un 19.8% de descuento promedio y un ratio de tarifa de entrega de 1.31 en distancias de 5.13 km, generando una pérdida neta promedio de -$0.17 USD por pedido.

La tormenta perfecta operativa (SQL 2): Cuando coinciden Hora Pico + Tormenta (Heavy Rain / Storm), el índice de fricción se multiplica casi 9x (de 0.68 a 6.11), el tiempo total sube de 25.2 min a 39.7 min (+4.8 min en cocina y +9.7 min en reparto) y la calificación promedio cae por debajo del umbral crítico de 4 estrellas (3.90).

Rentabilidad por cocina (SQL 3): Burgers & Fast Food lidera el volumen total ($18,740.67 USD en 1,788 órdenes y rating de 4.31), mientras que Japanese & Sushi ofrece el mayor margen unitario ($11.54 USD por orden).

📈 Fase 3: Dashboard Ejecutivo en Power BI
A partir del dataset enriquecido (delivery_clean_powerbi.csv), se construyó un tablero interactivo de 2 vistas con segmentación dinámica por canal (Dine-In POS, Online Delivery App, Takeaway / Pickup):

<img width="1082" height="614" alt="dashboard_pagina1_salud_financiera" src="https://github.com/user-attachments/assets/7f5ac5b9-5d41-48c8-a23c-cc1198f19ad5" />

<img width="1113" height="636" alt="dashboard_pagina2_cuello_botella" src="https://github.com/user-attachments/assets/185139b8-b0a9-47b7-a0e0-c879c15f56f2" />


### 1. Salud Financiera:
Auditoría de Rentabilidad y Fugas de Margen
### 2. Cuello de Botella Operativo: 
Tiempos de Preparación y Satisfacción
     
## 💡 Recomendaciones Estratégicas Accionables

Regla de Pedido Mínimo y Tope de Descuento: Establecer un monto mínimo de $8.00 – $10.00 USD en Online Delivery App para el segmento $ Budget y bloquear descuentos superiores al 15% en tickets menores a $12.00 USD. Impacto esperado: eliminación del 96.7% de las órdenes con margen negativo sin afectar el volumen rentable.

Protocolo de Contingencia en Hora Pico y Mal Clima: Reforzar estaciones de cocina durante horas pico (donde la preparación sube invariablemente +5 min) y ajustar dinámicamente el radio de cobertura en delivery bajo condiciones de tormenta para proteger el SLA (< 35 min) y evitar el riesgo de abandono (Churn Risk).

Reasignación de Presupuesto Promocional: Reducir los descuentos promedio del 12.5% en categorías con ticket alto y demanda orgánica sólida (Healthy & Bowls e Indian & Curries) para reinvertir en la promoción de Japanese & Sushi (mayor margen unitario).
