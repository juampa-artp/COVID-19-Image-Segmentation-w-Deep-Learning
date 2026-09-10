# COVID-19-Image-Segmentation-w-Deep-Learning

Repositorio para la fase de Extraccion, Limpieza y Transformacion de Datos (ETL) correspondiente al reto de segmentacion semantica de lesiones pulmonares por COVID-19 en imagenes tomograficas de torax (CT) en Kaggle.


- **Objetivo:** Identificar y delimitar pixeles con Vidrio Deslustrado (Ground-Glass Opacity - GGO) y Consolidaciones pulmonares.
- **Metrica de Evaluacion:** Pixel-wise Macro F1-Score.
- **Formato de Envio Kaggle:** Archivo CSV plano con 5,242,880 filas (10 imagenes de prueba x 512 x 512 x 2 canales).

---

## Justificacion del ETL para Imagenes Tomograficas

Las imagenes de tomografia axial computarizada (CT) almacenan densidades radiologicas en Unidades Hounsfield (HU), las cuales presentan retos especificos que requieren un preprocesamiento cuidadoso:

1. **Correccion de Colisiones Multietiqueta (Fondo vs Lesion):**
   - En las anotaciones crudas de 4 canales, se detecto que 510 pixeles en Medseg y 22,027 pixeles en Radiopedia estaban etiquetados simultaneamente como fondo (Canal 3) y como lesion patologica (Canales 0 o 1).
   - Se aplico el principio de jerarquia anatomica: un pixel diagnosticado con patologia pulmonar no puede ser fondo. El canal de fondo se recalcula como el complemento estricto del parenquima y las lesiones: `Fondo = ~(Lesiones | Pulmon)`.

2. **Ventaneo Tomografico Pulmonar (*Lung Windowing*):**
   - Las imagenes crudas contienen densidades desde -1600 HU (artefactos de gantry/aire externo) hasta mas de 3000 HU (calcio denso o metal).
   - Se aplica un ventaneo en el rango [-1000, 400] HU (ancho de ventana de 1400 HU centrado en -300 HU), preservando los detalles del parenquima pulmonar y de las lesiones infecciosas.

3. **Normalizacion Min-Max Continua:**
   - La escala [-1000, 400] HU se reescala afinementente al intervalo continuo [0.0, 1.0]. Esto asegura estabilidad numerica y evita la saturacion de gradientes durante el entrenamiento de redes convolucionales (como U-Net).

4. **Adaptacion a 2 Canales:**
   - De los 4 canales originales (0: Vidrio deslustrado, 1: Consolidacion, 2: Pulmon sano, 3: Fondo), se extraen unicamente los canales patologicos [0, 1] en formato binario uint8, con dimension `(N, 512, 512, 2)`.

5. **Particion Estratificada Train / Validation (80/20):**
   - De los 929 cortes disponibles (100 de Medseg y 829 de Radiopedia), 471 son positivos a COVID-19 y 458 no presentan lesiones visibles.
   - Se genera una division estratificada 80/20 que preserva la proporcion balanceada (~50.6% positivos) tanto en entrenamiento (743 cortes) como en validacion (186 cortes).

---

## Que hace el Notebook (`notebooks/01_ETL_Exploracion_Limpieza_Transformacion.ipynb`)

El notebook es 100% autocontenido y ejecuta el flujo completo de forma secuencial:

1. **Configuracion y Funciones Base:** Importa librerias estandar (NumPy, Pandas, Matplotlib, Scikit-learn) y define las funciones de limpieza, ventaneo, normalizacion, metrica F1 y generacion de envios.
2. **Extraccion e Inspeccion Dimensional:** Carga los arreglos `.npy` (`images_medseg`, `masks_medseg`, `images_radiopedia`, `masks_radiopedia`, `test_images_medseg`), comprueba dimensiones, tipos de datos y valida que existan 0 valores nulos (NaNs) y 0 infinitos (Infs).
3. **Limpieza de Datos:** Identifica las colisiones multietiqueta y ejecuta la resolucion anatomica, reportando la eliminacion total de solapamientos anomalos.
4. **Transformacion de Variables:** Aplica ventaneo pulmonar [-1000, 400] HU, normalizacion [0.0, 1.0] y extraccion de los canales objetivo de la competencia.
5. **Visualizacion Diagnostica:** Muestra un corte tomografico representativo con superposicion a color de las lesiones (verde: Vidrio Deslustrado, rojo: Consolidacion) y su mapa categorico correspondiente.
6. **Analisis Clinico y Division Train / Validation:** Construye la tabla de metadatos por corte y ejecuta la particion estratificada balanceada 80/20.
7. **Helper de Formato Kaggle:** Simula una prediccion y genera la estructura tabular exacta requerida para el archivo de envio a la plataforma (`submission.csv`).

---

## Ejecucion

1. Asegurarse de tener en el directorio los archivos de datos `.npy` o dentro de una carpeta `data/`.
2. Abrir el notebook interactivo:
```bash
jupyter notebook notebooks/01_ETL_Exploracion_Limpieza_Transformacion.ipynb
```
