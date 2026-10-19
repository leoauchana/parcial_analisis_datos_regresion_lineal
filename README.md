# Regresión Lineal sobre el Mercado Inmobiliario de Moscú

**Primer Parcial – Análisis de Datos (UTN)**  
**Integrantes:**
- Auchana, Matías Leonel
- Casanueva, Facundo

---

## Introducción y Objetivos

Este proyecto aborda el análisis y la predicción de precios en el mercado inmobiliario residencial de Moscú (Rusia), cubriendo tanto operaciones de **venta** en el mercado secundario como de **alquiler mensual**.

El enfoque metodológico se basa en **Mínimos Cuadrados Ordinarios (OLS)**, priorizando:
1. **Interpretabilidad cuantitativa:** cuantificar de manera directa el impacto porcentual y marginal de cada atributo del inmueble (ubicación geográfica, distancia al metro, tipología constructiva, nivel de renovación, etc.).
2. **Capacidad predictiva y generalización:** minimizar el error cuadrático y relativo sin incurrir en sobreajuste (*overfitting*), validando con esquemas de división entrenamiento/prueba (80/20) y validación cruzada (*Repeated K-Fold*).
3. **Reproducibilidad y trazabilidad:** registro sistemático de parámetros, matrices de diseño, métricas de test/train y diagnósticos de residuos mediante **MLflow**.

---

## Datasets utilizados

Se utilizó el dataset público de Kaggle: [`sergionefedov/moscow-real-estate-sales-and-rentals-20202026`](https://www.kaggle.com/datasets/sergionefedov/moscow-real-estate-sales-and-rentals-20202026), que abarca operaciones y series mensuales entre enero de 2020 y abril de 2026.

| Archivo | Observaciones | Variables clave | Rol en el Proyecto |
| :--- | :--- | :--- | :--- |
| `secondary_market.csv` | 50.000 anuncios | Precio total, m², ubicación, año de construcción, piso, estado | **Modelo principal de ventas** |
| `rentals.csv` | 20.000 anuncios | Renta mensual, m², distrito, línea de metro, amoblado, mascotas | **Modelo de alquileres** |
| `district_prices_monthly.csv` | Serie panel mensual | Histórico de precios y rentas promedio por distrito | **Variable exógena de mercado (rezago mensual sin data leakage)** |
| `new_builds.csv` | 10.000 anuncios | Obra en pozo / departamentos a estrenar | Explorado en EDA (segmento con dinámica de precios desacoplada) |
| `metro_stations.csv` | Catálogo de estaciones | Coordenadas y correspondencias de líneas | Explorado (información resumida en distancia y línea) |

---

## Metodología y Decisiones de Modelado

### 1. Análisis Exploratorio de Datos (EDA)
- **Asimetría del target:** Fuerte asimetría positiva (*skewness*) y curtosis elevada en los precios nominales en rublos (`price_rub`), con colas pesadas de propiedades suntuosas.
- **Heterocedasticidad estructural:** La dispersión de precios aumenta proporcionalmente con el valor del inmueble.
- **Colinealidad de superficies:** Alta correlación entre `total_area`, `living_area`, `kitchen_area` y cantidad de ambientes (`rooms`). Para evitar inflación de varianza (VIF elevado), se priorizó la superficie total.

### 2. Detección y Tratamiento de Outliers (Criterio Tukey Zonal)
- Se descartó un filtro global de precios porque eliminaría departamentos grandes legítimos en zonas exclusivas.
- **Criterio adoptado:** Se filtraron valores atípicos que superen $Q_3 + 1.5 \times \text{IQR}$ calculado **dentro de cada zona administrativa (*okrug*)**, aislando anomalías reales de mercado por m² dentro de su entorno barrial.

### 3. El Cambio a Precio por Metro Cuadrado (`price_per_sqm`)
- El precio total depende mecánicamente de la superficie, distorsionando el efecto de las demás variables explicativas.
- Modelar `price_per_sqm` desacopla el tamaño del departamento, homogeneiza la varianza y produce residuos más estables.
- **Comparabilidad de errores:** Para contrastar modelos con target en `price_rub` frente a modelos en `price_per_sqm`, las predicciones de estos últimos se reescalan a rublos totales.

### 4. Transformación Logarítmica
- Aplicar logaritmo natural al target ($\ln(\text{target})$) convierte el modelo en una especificación **semi-logarítmica**, donde los coeficientes representan cambios porcentuales directos.

### 5. Incorporación del Panel Mensual sin Fuga de Información (*No Data Leakage*)
- Desde `district_prices_monthly.csv` se computó el logaritmo del precio promedio del distrito correspondiente al mes calendario previo.
- Permite capturar la inflación inmobiliaria, los shocks macroeconómicos y la prima geográfica específica del momento, utilizando exclusivamente información disponible al publicarse el anuncio.

### 6. Modelo de Alquileres
- Partiendo de todas las covariables disponibles en `rentals.csv`, se realizó una selección progresiva hacia atrás evaluando la significancia estadística ($t$-statistic y $p$-values < 0.05).
- Se incorporaron las variables de amenidades clave (`furnished`, `pets_allowed`), línea de metro (`metro_line`), distritos y el rezago de renta zonal mensual.

---

## Resultados Principales y Comparativa

Las métricas reportadas corresponden a conjuntos de prueba independientes (**Test Set 20%**) y validación cruzada:

| Modelo / Especificación | R² | RMSE (Escala original) | MAE (Escala original) | Error Relativo (MAPE) | Diagnóstico de Residuos |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Venta – Baseline inicial** (`price_rub`, lineal) | 0.831 | ≈ 3.200.000 RUB | ≈ 2.235.000 RUB | ~16.5 % | Residuos muy asimétricos (*skew* 0.93, *kurtosis* 8.6) |
| **Venta – Modelo Final** ($\ln(\text{ppsqm})$, distrito, lag mensual) | **0.962** | **≈ 1.705.000 RUB** | **≈ 1.117.000 RUB** | **8.47 %** | Residuos cuasi-normales (*skew* -0.10, *kurtosis* 3.09) |
| **Alquiler – Modelo Final** ($\ln(\text{rent/m²})$, distrito, lag mensual) | **0.943** | **≈ 7.656 RUB/mes** | **≈ 5.122 RUB/mes** | **9.45 %** | Residuos simétricos (*skew* 0.06, *kurtosis* 3.15) |

### Conclusiones Técnicas
- **Reducción del error en ventas en casi un 47%:** El pasaje de `price_rub` mediante la aplicación de log junto al contexto del distrito y su rezago mensual redujo el RMSE de ~3,2M a ~1,7M RUB.
- **Ausencia de sobreajuste (*Overfitting*):** El error en entrenamiento y en prueba resulta prácticamente idéntico (Ventas: Train RMSE 1,725M vs Test RMSE 1,705M; Alquileres: Train RMSE 7.641 vs Test RMSE 7.656).
- **Alcance del modelo:** Las especificaciones obtenidas no pretenden emular la negociación final o factores micro subjetivos, pero proporcionan un tasador de referencia altamente preciso para detectar sobreprecios u oportunidades en anuncios de mercado.

---

## Estructura del Repositorio

El proyecto se encuentra organizado dentro del directorio [`regresion_lineal`]:

```
.
├── README.md
├── regresion_lineal/
│   ├── regresion_venta_baseline.ipynb   # EDA completo (univariado, bivariado), outliers y baseline en price_rub
│   ├── regresion_ppsqm_baseline.ipynb   # Modelo ppsqm con log, panel mensual y modelo de alquileres (Sec. 5)
│   ├── mlflow.db                        # Base de datos SQLite con los experimentos y métricas de MLflow
│   ├── mlartifacts/                     # Artefactos, gráficos de residuos y resúmenes OLS
│   ├── district_prices_monthly.csv      # Serie temporal panel por distrito (Kaggle)
│   ├── secondary_market.csv             # Dataset de ventas (Kaggle)
│   ├── rentals.csv                      # Dataset de alquileres (Kaggle)
│   ├── new_builds.csv                   # Dataset de obra nueva (Kaggle)
│   └── metro_stations.csv               # Dataset de estaciones de metro (Kaggle)
```

### Experimentos registrados en MLflow
1. **`Parcial_Regresion_Ventas`**: Comparación de baselines sobre precio total en rublos, filtrado de outliers y primeras aproximaciones con distrito.
2. **`Parcial_ppsqm`**: Modelos sobre precio por m².
3. **`Parcial_Alquileres`**: Proceso de selección secuencial de variables para alquileres y modelo final optimizado.

---

## Guía de Instalación y Ejecución

### 1. Clonar el repositorio y preparar el entorno
Se recomienda utilizar un entorno virtual con Python 3.10 o superior:

```bash
git clone <url-del-repositorio>
cd primer_parcial
```

### 2. Instalar las dependencias
```bash
pip install pandas numpy statsmodels scikit-learn scipy matplotlib seaborn mlflow
```

### 3. Ejecución de los Notebooks
1. Abrir Jupyter Lab o VS Code en el directorio del proyecto.
2. **Para el análisis exploratorio y baseline de ventas:** Ejecutar [`regresion_lineal/regresion_venta_baseline.ipynb`]
3. **Para los modelos finales de ventas y alquileres:** Ejecutar [`regresion_lineal/regresion_ppsqm_baseline.ipynb`]

### 4. Explorar los Experimentos en la UI de MLflow
Para visualizar interactivamente las curvas de residuos, tablas de parámetros y métricas comparativas:

```bash
mlflow ui --backend-store-uri sqlite:///regresion_lineal/mlflow.db
```

Luego, acceder desde el navegador a: `http://localhost:5000`.
