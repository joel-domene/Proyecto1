# Arquitectura del Sistema

Este documento describe en detalle el pipeline de detección de paneles de autopista, las técnicas empleadas en cada etapa y las decisiones de diseño que las justifican. Todo el sistema se basa en **visión por computador clásica**, sin aprendizaje automático (restricción del enunciado de la práctica).

-----

## 1. Visión general

El sistema ofrece tres detectores seleccionables por CLI (`--detector mser | hough | combinado`). El detector **combinado** ejecuta las dos ramas independientes y fusiona sus salidas eliminando duplicados.

```
Imagen BGR
   │
   ├── Rama A: MSER + CLAHE + filtrado geométrico + puntuación HSV
   │
   ├── Rama B: Color HSV + morfología + contornos + Canny + Hough
   │
   └── Fusión NMS (IoU 0.3, IoS > 0.4) → detecciones finales
```

-----

## 2. Rama A — Detector MSER

### 2.1. Preprocesado

- **Conversión a escala de grises** (`cv2.cvtColor`, BGR→GRAY).
- **Realce de contraste local con CLAHE** (`cv2.createCLAHE`, `clipLimit=2.0`, `tileGridSize=8×8`).
  *Por qué CLAHE y no ecualización global:* la iluminación de las escenas viales es muy heterogénea (sombras, contraluces). El realce local mejora el contraste del panel sin amplificar el ruido del resto de la imagen.

### 2.2. Extracción de regiones candidatas

- **MSER** (`cv2.MSER_create`, `detectRegions`) con parámetros: `delta=5`, `min_area=100`, `max_area=35000`, `max_variation=0.25`, `min_diversity=0.2`.
  *Por qué MSER:* los paneles son regiones de intensidad estable y bien delimitadas frente al fondo; MSER las detecta sin necesidad de entrenamiento.

### 2.3. Filtrado geométrico

Cada región candidata se filtra por:

- **Tamaño** (área dentro de rango razonable).
- **Relación de aspecto** (aspect ratio entre **0.4 y 4.5**).
- **Ocupación** (proporción de píxeles de la región respecto a su *bounding box*).

### 2.4. Expansión de la *bounding box*

MSER tiende a capturar el interior azul del panel. Se **expande la caja** para incluir el **borde blanco** característico, que es parte distintiva del panel.

### 2.5. Puntuación (scoring)

- Se redimensiona la ventana candidata a **40×80** (`cv2.resize`).
- Se construye una **máscara HSV de azul saturado**: rango inferior `[100, 80, 40]`, superior `[130, 255, 255]` (`cv2.inRange`).
- Se compara con una **máscara ideal** mediante **F1-score**.
- **Umbral de aceptación: 0.3.**

-----

## 3. Rama B — Detector propio (Color + Hough)

Segundo detector diseñado como **iniciativa propia**, fuera de lo exigido por el enunciado. Aporta una señal complementaria a MSER.

1. **Máscara de color HSV** del azul del panel (`cv2.inRange`).
1. **Morfología** (`cv2.morphologyEx`): cierre para rellenar huecos y apertura para eliminar ruido.
1. **Detección de contornos** (`cv2.findContours`) y cálculo de cajas (`cv2.boundingRect`).
1. **Detección de bordes con Canny** (`cv2.Canny`, umbrales 50 y 150).
1. **Transformada de Hough probabilística** (`cv2.HoughLinesP`) para detectar líneas.
1. **Conteo de líneas horizontales y verticales**: si la región presenta **estructura rectilínea** (típica de un panel rectangular), se aplica un **bonus de +0.05** a su puntuación.

*Por qué Hough:* los paneles tienen bordes rectos fuertes. Premiar la estructura rectilínea ayuda a distinguir un panel real de una mancha azul cualquiera (cielo, vehículo, etc.).

-----

## 4. Fusión — Non-Maximum Suppression

Al combinar dos detectores aparecen **detecciones duplicadas** sobre el mismo panel. Se resuelven con **NMS** usando dos criterios:

- **IoU (Intersection over Union) > 0.3** — solapamiento clásico.
- **IoS (Intersection over Smaller) > 0.4** — captura el caso en que una caja está contenida dentro de otra mayor.

Se conserva la detección de mayor puntuación y se descartan las solapadas.

-----

## 5. Evaluación

Implementada en `evaluar_resultados.py`. Métricas calculadas sobre el conjunto de test (102 imágenes):

|Umbral   |Precisión|Recall|AP  |
|---------|---------|------|----|
|IoU > 0.5|~85%     |~65%  |47.5|
|IoU > 0.7|~80%     |~55%  |28.2|

Referencia de los profesores: **AP 68.7 (IoU 0.5) / 60.9 (IoU 0.7)**.

-----

## 6. Métodos de OpenCV utilizados

`cvtColor` · `createCLAHE` · `MSER_create` / `detectRegions` · `inRange` · `morphologyEx` · `findContours` · `boundingRect` · `Canny` · `HoughLinesP` · `rectangle` · `putText` · `imread` / `imwrite` · `resize`

-----

## 7. Limitaciones de diseño

- **Paneles pequeños o lejanos:** poca información de píxeles para puntuar con fiabilidad.
- **Oclusión parcial:** rompe la estructura rectilínea y la región estable.
- **Falsos positivos:** superficies azules grandes (cielo, vallas, vehículos) pueden superar los filtros.

Estas limitaciones son inherentes a un enfoque clásico y motivan la propuesta de migrar a *deep learning* (YOLO, Faster R-CNN) como trabajo futuro.