# Método de Triangulación en Muografía — CCTVal · UTFSM

**Autor:** Eduardo Andrés Novoa Carrasco  
**Afiliación:** Centro Científico Tecnológico de Valparaíso (CCTVal), Universidad Técnica Federico Santa María (UTFSM)  
**Notebook de referencia:** `Analisis (2).ipynb`

---

## Resumen

Este trabajo presenta un **pipeline reproducible** para **detección y reconstrucción 3D de anomalías de densidad mediante muografía**, combinando filtrado en el dominio de la frecuencia (Wiener), separación de canales **positivo/negativo** (asociados a **cavidades/agua** vs **alta densidad/cobre** con **roca** como referencia), **clustering** (K‑Means) y **triangulación volumétrica** por acumulación de rayos desde múltiples detectores. Finalmente, se evalúa la superposición entre la nube reconstruida y un **cono/frustum de referencia (ground truth)**, reportando **Cobertura** y **Precisión** voxelar.

**Palabras clave:** muografía, reconstrucción 3D, filtrado de Wiener, K‑Means, muones, voxelización, CCTVal, UTFSM.

---

## Objetivos

1. Construir mapas 2D de señal a partir de trazas de muones por detector.
2. Atenuar ruido y realzar estructura angular mediante **filtro de Wiener** y **suavizado gaussiano**.
3. Separar canales físicos **positivos (cavidades/agua)** y **negativos (metales/cobre)** respecto de **roca** (referencia).
4. Detectar **objetos principales** por **K‑Means** y selección de extremos (máximos o mínimos).
5. **Triangular en 3D** acumulando rayos entre detectores y voxelizando el espacio.
6. **Cuantificar** desempeño con **Cobertura** (GT cubierto por la nube) y **Precisión** (nube dentro del GT).

---

## Datos de entrada

- Archivos **ROOT** de **señal** y **background** con árbol `tree_muonTrk` y ramas:
  - `mu_sph_X`, `mu_sph_Y` (coordenadas angulares por evento)
  - `det_idx` (índice del detector por evento)
- Conjunto de detectores `det_ids` (p. ej., `0..5`) y sus **posiciones 3D** (`detector_positions`).
- Sistema de referencia: eje **z** hacia abajo (subsuelo), con volumen ROI rectangular en `x, y, z`.

> El notebook carga y proyecta por detector con `_load_detector_data(...)` y `_xy_to_angles(...)`.

---

## Método (Materiales y Procedimiento)

### 1) Preprocesamiento y mapas 2D por detector
- Se construyen histogramas 2D (mapas angulares) por detector a partir de `mu_sph_X, mu_sph_Y`.
- Funciones clave:  
  - `process_detector_simple(...)`, `process_all_detectors(...)`  
  - Visualización en grilla: `plot_detectors_grid(...)`

### 2) Filtrado y realce
- **Wiener en frecuencia**: `_wiener_lowpass_freq(...)`, `apply_wiener_to_results(...)`
  - Parámetros: `cutoff_frac` (bajo‑pasa gaussiano) y `noise_ring` (estimador de ruido fuera de anillo).
- **Z‑score**: `add_zscore_to_results(...)`
- **Gaussiano espacial** (opcional): `apply_gaussian_to_results(...)`
- **Wavelets** (opcional): `_wavelet_denoise_2d(...)`, `apply_wavelet_to_results(...)`

### 3) Separación física por canal
- `split_pos_neg_physical(...)` divide mapas en **positivo** y **negativo** y los **escala** respecto a densidades típicas:
  ```python
  densidades = {'agua': 1.0, 'roca': 2.6, 'cobre': 8.96}
  ```
- **Convención**: roca ≈ 0 (referencia); **positivo → menor densidad** (cavidad/agua), **negativo → mayor densidad** (cobre).

### 4) Detección de objeto principal (2D)
- **K‑Means** en cada canal: `kmeans_on_channel_signed(...)` (por detector).
- **Selección por extremo**: `select_object_by_extreme(map2d, mode="max|min")`
  - Para **agua/cavidad**: `mode="max"`  
  - Para **cobre**: `mode="min"`
- Extracción y métricas por detector: `extract_objects_from_detector(...)`, `show_main_objects_grid(...)`.

### 5) Triangulación y voxelización 3D
- Se acumulan **rayos** de cada detector mediante **muestreo** angular (phi, theta) y longitud de rayo:
  - `accumulate_distinct_detectors(...)` genera un cubo **(xs, ys, zs)** y un **volumen `counts[x,y,z]`** de intersecciones.
  - Suavizado volumétrico opcional: `smooth_volume_3x3x3(...)`.
- Proyecciones diagnósticas: `plot_projections_xy_xz(...)`.

### 6) Métricas contra Ground Truth (cono/frustum)
- Se define un **frustum** (cono truncado) de referencia para **agua**/**cobre**:
  - `make_frustum_mask(xs, ys, zs, cx, cy, z_start, height, r_top, r_bottom, direction)`
- **Comparación nube vs frustum**: `plot_cloud_vs_cone_with_metrics(...)`  
  Reporta:
  - **Cobertura (GT)** = |nube ∩ GT| / |GT|
  - **Precisión (nube)** = |nube ∩ GT| / |nube|
- Visualización 3D (Plotly): isosuperficie del núcleo + malla del cono/frustum.

---

## Resultados (lugares para figuras)



- **Figura 1.** Mapas 2D por detector — sin filtro vs. con Wiener/gaussiano.
- <img width="1210" height="711" alt="image" src="https://github.com/user-attachments/assets/6501c4dc-813e-4f41-a0eb-e4adecdd7f6c" />
<img width="1223" height="711" alt="image" src="https://github.com/user-attachments/assets/cd0c739b-bafd-448a-969d-27eb5693ed03" />
<img width="1210" height="711" alt="image" src="https://github.com/user-attachments/assets/904a9ed4-d19b-4de4-a405-afa262518ec4" />


- **Figura 2.** Separación de canales positivo/negativo y clusters K‑Means (objetos).
  <img width="1210" height="711" alt="image" src="https://github.com/user-attachments/assets/62a525bc-8aa4-462c-b8e5-022b5aadd426" />
 <img width="1210" height="711" alt="image" src="https://github.com/user-attachments/assets/daac0288-92a2-4d0a-9d2d-75572a0069b3" />
<img width="1114" height="593" alt="image" src="https://github.com/user-attachments/assets/1bab94db-a70e-4db4-8239-882d7fbaf8c5" />
<img width="1104" height="593" alt="image" src="https://github.com/user-attachments/assets/082234ac-c954-45f4-b24f-96fe79c5d58d" />
<img width="1076" height="593" alt="image" src="https://github.com/user-attachments/assets/1648dd81-8a4b-4758-bf2e-26fae4e9c320" />
<img width="1084" height="593" alt="image" src="https://github.com/user-attachments/assets/fb1cbbb8-6f6a-429b-a9f1-d4877a3e0217" />


- **Figura 3.** Nube 3D por acumulación de rayos (vista isométrica).
<img width="1003" height="480" alt="image" src="https://github.com/user-attachments/assets/19645dd7-b23c-4316-8d46-548e081aee83" />
<img width="1018" height="486" alt="image" src="https://github.com/user-attachments/assets/42002ef3-2de9-41f1-a81b-53eaf89eea21" />

- **Figura 4.** Proyecciones XY / XZ de la nube.
<img width="1740" height="567" alt="image" src="https://github.com/user-attachments/assets/9efb3436-4737-48a9-b653-606051edd1ff" />
<img width="1736" height="561" alt="image" src="https://github.com/user-attachments/assets/3ab396af-fc8b-41d3-959f-3fbbda1671d0" />

- **Figura 5.** Comparación con frustum (agua/cobre) con **Cobertura** y **Precisión**.

<img width="1693" height="512" alt="image" src="https://github.com/user-attachments/assets/87310036-b2b3-4161-bb75-a18b7795ad28" />
<img width="1677" height="560" alt="image" src="https://github.com/user-attachments/assets/37cae0fb-ce09-4e82-8cb1-fad2d3952490" />

---

## Discusión y limitaciones

- La **calibración angular** y la **geometría exacta** de los detectores impactan fuertemente la triangulación.
- El **ruido no gaussiano** y la **dispersión múltiple** pueden sesgar los mapas y la acumulación volumétrica.
- La separación **pos/neg** asume una referencia de **roca** y rangos de densidad representativos; requiere validación local.
- **K‑Means** es sensible a escalamiento y a la elección de `n_clusters`; puede complementarse con DBSCAN/GMM.
- La métrica **isovoxel** depende de umbrales (e.g., mínimo de detectores por voxel, percentiles para isosuperficies).

---

## Reproducibilidad

### Requisitos de entorno
- Python ≥ 3.10
- Dependencias principales:
  - `numpy`, `scipy` (`ndimage`, `fft`), `matplotlib`, `plotly`
  - `awkward`, `uproot` (lectura ROOT)
  - `scikit-learn` (K‑Means)
  - `pywt` (opcional: denoise por wavelets)
  
Instalación rápida (ejemplo):
```bash
python -m venv .venv
source .venv/bin/activate  # (Windows: .venv\Scripts\activate)
pip install numpy scipy matplotlib plotly awkward uproot scikit-learn pywt
```

Si empleas este pipeline o partes de él, por favor cita:
```
Novoa-Carrasco, E. A., CCTVal – UTFSM (2025). Método de triangulación en muografía:
pipeline de filtrado, clustering y voxelización con métricas de cobertura/precisión. 
Repositorio y notebook: Analisis (2).ipynb.
```

---

## Agradecimientos

A **CCTVal – Universidad Técnica Federico Santa María** por el apoyo técnico y computacional.  
A los colaboradores del proyecto y al equipo por la discusión metodológica.
