# Tratamiento y Mejora de Imágenes Digitales en Visión Artificial

**Autor:** Carlos Mario Salvatore Ocampo Aguilar  
**Programa:** Especialización en Inteligencia Artificial — Universidad de Cundinamarca  
**Curso:** Percepción Computacional (CADI IMAGEN, REA 2 / R2-A2-S8S9)  
**Informe en Overleaf:** [Proyecto Overleaf (Norma APA 7ma edición)](https://www.overleaf.com/project/6ad566f910500e05f7c238dd)  
**Repositorio GitHub:** [Salv4tore27/carlos-tratamiento-imagen](https://github.com/Salv4tore27/carlos-tratamiento-imagen)

---

## ¿De qué trata este trabajo?

Para esta entrega del curso de Percepción Computacional construí un dataset experimental de 5 capturas fotográficas de un mismo objeto bajo distintas condiciones físicas del entorno. El objeto elegido fue un control inalámbrico para videojuegos (gamepad Machenike G5 Pro), el cual presenta superficies plásticas blancas, botones mecánicos, cruceta direccional (D-pad), joysticks electromagnéticos Hall effect e iluminación LED sutil.

Las 5 situaciones capturadas fueron:

1. **Referencia:** Toma frontal a distancia media con iluminación homogénea y difusa, donde se aprecian con claridad los botones de acción, palancas y contorno general.
2. **Luz desigual:** Fuente direccional rasante proveniente de un costado, generando una transición lumínica marcada con pérdida de contraste en la zona opuesta.
3. **Sombra intensa (la imagen seleccionada para tratamiento):** Una sombra densa y proyectada que cubre casi por completo la cruceta direccional y el sector izquierdo del mando, sumiendo los píxeles en niveles muy bajos de gris.
4. **Movimiento:** Captura efectuada con desplazamiento relativo manual entre el sensor y el mando, evidenciando un desenfoque cinético (*motion blur*) con caída en la nitidez espacial.
5. **Cambio de ángulo / perspectiva:** Vista isométrica lateral en escorzo, registrando la profundidad tridimensional, el perfil de los gatillos y las juntas de ensamblaje.

El objetivo fue diagnosticar cuantitativamente el estado fotométrico y espacial de cada captura mediante métricas reproducibles (luminancia media, contraste por desviación estándar, nitidez por varianza del Laplaciano y densidad de bordes con Canny), justificar de forma objetiva cuál toma requería intervención y diseñar un algoritmo de realce adaptativo que recuperara la visibilidad de los controles sin arruinar la colorimetría de la escena.

---

## ¿Por qué seleccioné la foto con sombra y no la de movimiento?

Al enfrentarse a un dataset con defectos visuales, la primera tentación suele ser intentar "arreglar" la imagen borrosa por movimiento. Sin embargo, en visión por computador existe una diferencia fundamental entre ambos fenómenos:

- **La captura con movimiento** sufre una degradación cinemática. Matemáticamente, el movimiento relativo integra la trayectoria espacial del objeto sobre el plano del sensor durante el tiempo de exposición, lo que equivale a una convolución bidimensional con una función de dispersión del punto (PSF) desconocida. Como no se tiene acceso al vector exacto de aceleración y velocidad angular durante el disparo, cualquier intento de filtrado espacial (filtros pasa-altas o deconvolución ciega) introduce fuertes artefactos de oscilación (*ringing* o efecto halo) y dispara el ruido de alta frecuencia. En términos prácticos: no hay información de gradiente nítido en el sensor que se pueda rescatar sin inventar datos.
- **La captura con sombra intensa**, en contraste, no sufre pérdida de fase óptica ni desenfoque geométrico. Los detalles mecánicos del mando (relieve de la cruceta, serigrafía y junturas) siguen físicamente registrados en la matriz de píxeles, pero quedaron comprimidos en la parte más baja de la curva de cuantización del sensor (el 81.67% de los píxeles está por debajo del umbral de 50 en la escala de 8 bits). Aquí un algoritmo de rango dinámico local sí tiene la capacidad matemática de expandir los niveles de gris y hacer visible la geometría oculta.

---

## La solución técnica: CLAHE en espacio CIE $L^*a^*b^*$

Si se aplica una ecualización de histograma tradicional sobre una imagen RGB:
1. Las zonas que ya estaban iluminadas se sobreexponen de inmediato (se saturan los blancos a 255).
2. Se producen aberraciones cromáticas graves en la carcasa y en las sutiles tonalidades doradas y grises del mando.
3. Se amplifica de manera descontrolada el grano y ruido térmico en las sombras profundas.

Para evitar estas distorsiones, el procesamiento se estructuró de la siguiente forma:

1. **Desacoplamiento tonal:** Se realiza la transformación del espacio de color BGR nativo de OpenCV al espacio uniforme perceptualmente CIE $L^*a^*b^*$. En este espacio, el canal $L^*$ codifica de manera aislada la luminancia (brillo), mientras que $a^*$ (eje verde-rojo) y $b^*$ (eje azul-amarillo) representan la información cromática.
2. **Ecualización adaptativa local (CLAHE):** Se aplica el algoritmo CLAHE (*Contrast Limited Adaptive Histogram Equalization*) únicamente sobre la matriz $L^*$. La imagen se subdivide en una malla de $8 \times 8$ bloques contextuales (*tiles*). En cada bloque se computa el histograma local y se aplica un límite de corte estricto (`clipLimit = 2.0`), redistribuyendo el exceso de frecuencia uniformemente en todos los bines antes de calcular la función de distribución acumulada (CDF). Con esto se impide que las zonas homogéneas oscuras amplifiquen el ruido.
3. **Interpolación y recombinación:** Para eliminar costuras entre bloques adyacentes, se interpola bilinealmente entre vecindades. Por último, se fusiona el canal $L^*$ mejorado con los canales cromáticos $a^*$ y $b^*$ originales (que permanecieron completamente intactos) y se reconvierte a BGR para guardar el resultado final en `resultados/sombra_tratada.png`.

---

## Resultados cuantitativos

El script `analisis_tratamiento.py` calcula las métricas de forma automatizada sobre las imágenes en disco.

### 1. Diagnóstico del dataset completo (Tabla 1 del informe)

| Captura | Formato | Tamaño | Resolución | Canales | Brillo medio ($\mu$) | Contraste ($\sigma$) | Nitidez ($\sigma^2_{\nabla^2}$) | Bordes Canny |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| 1. Referencia | PNG | 963 KB | $768\times1024$ | 3 BGR | 68.81 | 45.33 | 937.13 | 2.07% |
| 2. Luz desigual | PNG | 574 KB | $768\times1024$ | 3 BGR | 38.32 | 41.12 | 642.82 | 1.25% |
| 3. Sombra intensa | PNG | 390 KB | $768\times1024$ | 3 BGR | 25.53 | 36.92 | 429.37 | 0.83% |
| 4. Movimiento | JPEG | 912 KB | $768\times1024$ | 3 BGR | 67.28 | 47.37 | 361.32 | 1.14% |
| 5. Perspectiva | PNG | 862 KB | $768\times1024$ | 3 BGR | 74.80 | 58.98 | 556.55 | 1.15% |

### 2. Comparación antes y después del tratamiento CLAHE (Tabla 2 del informe)

Para una evaluación rigurosa, no solo se midió la imagen global sino una región de interés fija (ROI de $550\times560$ píxeles, coordenadas `[200:750, 220:780]`) centrada específicamente en la cruceta y botones mecánicos del gamepad:

| Región analizada | Estado | Brillo ($\mu$) | Contraste ($\sigma$) | Percentil 5 ($P_5$) | Píxeles oscuros ($<50$) | Nitidez ($\sigma^2_{\nabla^2}$) | Bordes Canny |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **Foto completa** | Antes | 25.53 | 36.92 | 1.00 | 81.67% | 429.37 | 0.83% |
| **Foto completa** | Después (CLAHE) | 37.57 | 45.29 | 6.00 | 76.15% | 1003.05 | 1.39% |
| **Zona del mando (ROI)** | Antes | 44.83 | 44.10 | 3.00 | 64.34% | 981.81 | 2.01% |
| **Zona del mando (ROI)** | Después (CLAHE) | 61.31 | 53.29 | 9.00 | 53.39% | 2331.25 | 3.41% |

### Principales hallazgos del tratamiento:
- En la zona funcional del control (ROI), la luminancia promedio ascendió de **44.83 a 61.31** (+16.48 puntos de ganancia lumínica).
- La proporción de píxeles profundamente subexpuestos descendió casi **11 puntos porcentuales** (de 64.34% a 53.39%), y el percentil 5 pasó de 3.0 a 9.0, sacando la textura de la penumbra.
- La nitidez estimada por la varianza del Laplaciano en la cruceta creció de **981.81 a 2331.25** (se multiplicó por más de 2.3 veces), y la densidad de contornos Canny subió de **2.01% a 3.41%**, permitiendo que un algoritmo de segmentación identifique con nitidez los bordes de los pulsadores.
- Todo lo anterior se consiguió sin alterar el balance de color de la carcasa ni saturar los reflejos luminosos del objeto.

---

## Estructura del repositorio

```text
carlos-tratamiento-imagen/
├── fotos/                           # Capturas fotográficas originales (sin modificar)
│   ├── referencia.png
│   ├── luz_desigual.png
│   ├── sombra_fuerte.png
│   ├── movimiento.jpg
│   └── angulo_perspectiva.png
├── resultados/                      # Salidas generadas por el pipeline
│   ├── metricas.csv                # Mediciones cuantitativas de brillo, contraste, nitidez y bordes
│   ├── metricas.json               # Datos en formato JSON estructurado con coordenadas del ROI
│   ├── cinco_capturas.png          # Panel comparativo con las 5 fotos analizadas
│   ├── sombra_tratada.png          # Captura con sombra tratada mediante CLAHE en Lab
│   └── comparacion_tratamiento.png # Mosaico comparativo tripartito (Referencia / Antes / Después)
├── analisis_tratamiento.py          # Script principal en Python (reproducible y documentado)
├── requirements.txt                 # Dependencias mínimas de ejecución (opencv-python, numpy)
├── main.tex                         # Código fuente del informe académico en LaTeX (APA 7)
└── README.md                        # Descripción y guía de reproducción
```

---

## Instrucciones para reproducir las pruebas

1. Clonar el repositorio:
   ```bash
   git clone https://github.com/Salv4tore27/carlos-tratamiento-imagen.git
   cd carlos-tratamiento-imagen
   ```
2. Instalar dependencias en el entorno virtual de Python:
   ```bash
   pip install -r requirements.txt
   ```
3. Ejecutar el pipeline de análisis:
   ```bash
   python analisis_tratamiento.py
   ```
   El script procesará automáticamente las imágenes en `fotos/`, calculará las matrices de diagnóstico y actualizará las tablas y figuras en la carpeta `resultados/`.
