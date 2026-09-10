# Cuantificación de brechas clínicas en la enfermedad cardíaca valvular: uso del Índice de Carga Medicamentosa (MBI) para el triaje de candidatos a intervenciones de alta complejidad

[![es](https://img.shields.io/badge/lang-es-red.svg)](https://github.com/enaromd/Cardio-MBI-Triage-Engine/blob/main/README.es.md)
[![en](https://img.shields.io/badge/lang-en-blue.svg)](https://github.com/enaromd/Cardio-MBI-Triage-Engine)

## Antecedentes del proyecto

Este análisis evalúa datos clínicos del mundo real procedentes de una misión cardiológica humanitaria de alto volumen en León, Nicaragua. Al operar como un modelo de prestación de atención especializada aguda, la misión cierra la brecha entre el cribado comunitario primario y la intervención cardíaca terciaria avanzada mediante un flujo de trabajo clínico estructurado en tres fases:

* **Fase 1 — Clínica de Cardiología General:** Tamizaje diagnóstico, evaluación ecocardiográfica y priorización de procedimientos.

* **Fase 2 — Electrofisiología:** Manejo de arritmias, incluyendo ablaciones cardíacas e implantación de marcapasos permanentes/DAI.

* **Fase 3 — Cardiología Intervencionista:** Procedimientos cardíacos estructurales percutáneos y revisión colaborativa de casos.

Desde la perspectiva de un Analista de Datos integrado en la misión, este proyecto evalúa la Cohorte 2025 de la Fase 1. Para enfocar el análisis en la enfermedad cardíaca estructural del adulto y maximizar el rendimiento de las intervenciones, la población inicial ($N = 187$) se filtró para excluir a pacientes menores de 15 años y a aquellos con hallazgos ecocardiográficos completamente normales, lo que arrojó una muestra analítica final de $N = 152$ pacientes.

Se ofrecen hallazgos y recomendaciones en las siguientes áreas clave:

- **[Auditoría forense de datos](#auditoría-forense-de-datos)**
- **[Imputación gaussiana normal natural y análisis de sensibilidad](#imputación-gaussiana-normal-natural-y-análisis-de-sensibilidad)**
- **[Índice de Carga Medicamentosa (MBI) como herramienta de triaje](#índice-de-carga-medicamentosa-mbi-como-herramienta-de-triaje)**

### Recursos y Enlaces del Proyecto

- **[Cuaderno de transformación e ingeniería de datos](https://github.com/enaromd/Cardio-MBI-Triage-Engine/blob/main/notebooks/01_extraction_transformation.ipynb)**
- **[Configuración centralizada ("Cerebro Médico")](https://github.com/enaromd/Cardio-MBI-Triage-Engine/blob/main/modules/config.py)**
- **[Cuaderno de análisis de datos](https://github.com/enaromd/Cardio-MBI-Triage-Engine/blob/main/notebooks/03_analysis.ipynb)**
- **[Dashboard interactivo de triaje de la misión](https://public.tableau.com/app/profile/enyel.a.rodr.guez.g./viz/MBIstratification/MBIstratification)**

## Estructura de datos y comprobaciones iniciales

La estructura subyacente de la base de datos MySQL del proyecto, diseñada a partir de expedientes clínicos digitalizados para alimentar el dashboard de Tableau, consta de cinco tablas centradas en $N = 152$ registros de pacientes. A continuación se describe cada tabla:

- `fact_patient`: Tabla principal que contiene las líneas de base demográficas individuales, signos vitales de triaje y parámetros ecocardiográficos cuantitativos continuos ($N = 152$).

- `dim_diagnoses`: Tabla de dimensión de referencia que aplica nomenclatura diagnóstica normalizada, tipos de lesiones valvulares y clasificaciones de severidad.

- `dim_medications`: Tabla de dimensión de referencia que cataloga agentes farmacológicos, clases terapéuticas de medicamentos y límites de dosis diaria máxima requeridos para la normalización de la dosificación.

- `bridge_diagnoses`: Tabla puente asociativa que resuelve relaciones de muchos a muchos entre pacientes y presentaciones complejas de enfermedad multivalvular.

- `bridge_medications`: Tabla puente asociativa que mapea regímenes de polifarmacia con pacientes, lo que permite cálculos granulares de la puntuación MBI sin duplicar filas demográficas principales.

![Esquema de base de datos](assets/database_schema.png)

## Resumen ejecutivo

### Visión general de los hallazgos

Una auditoría de la Clínica de Cardiología General 2025 ($N=152$) revela que la ausencia de datos en los parámetros ecocardiográficos es Faltante No al Azar (MNAR), impulsada por la omisión selectiva de mediciones cuantitativas por parte de los clínicos cuando las evaluaciones cualitativas muestran características no patológicas. Para evitar descartar el $80\%$ de los casos de pacientes incompletos, el flujo de procesamiento aplica la imputación gaussiana normal natural a los campos ecográficos no medidos, preservando la varianza real de la población sin distorsionar el riesgo clínico.

Al combinar la severidad ecocardiográfica estructural con el Índice de Carga Medicamentosa (MBI) diseñado, este flujo de procesamiento introduce un Marco de Discordancia Diagnóstica. Revelar la divergencia entre la enfermedad anatómica (eco) y la intensidad farmacológica (MBI) expone una Constante Nativa de línea base ($MBI = 2.27$) y un Umbral Crítico de Alerta Roja ($MBI = 5.25$), operacionalizando el MBI como una puntuación objetiva de triaje en el punto de registro para priorizar candidatos a intervenciones de alto rendimiento y capturar la descompensación hemodinámica en etapa terminal.

![Dashboard](assets/dashboard.png)

## Análisis profundo de hallazgos

### Auditoría forense de datos

- **Los signos vitales de triaje alcanzan un 99.67% de exhaustividad:** Las métricas de seguridad física y fisiológica registraron una tasa de datos faltantes del $0.33\%$ ($n=151.5$ registros completos), verificando que el triaje de enfermería de primera línea captura los parámetros de línea base de manera confiable en toda la cohorte.

- **Las dimensiones estructurales exhiben un 81.77% de dispersión selectiva:** Los parámetros de dimensión anatómica ($\text{IVSd, LVIDd, LVPWd, LVIDs}$) mostraron una ausencia promedio del $81.77\%$, lo que confirma que los clínicos documentan mediciones precisas con calibrador principalmente cuando se observa un agrandamiento evidente de las cavidades.

- **Pérdida de métricas hemodinámicas del 66.89%:** Las variables ecocardiográficas funcionales ($\text{TR Vmax, RVSP, MS MG}$) mostraron un $66.89\%$ de datos faltantes, registrándose gradientes de presión máxima casi exclusivamente durante insuficiencias o estenosis valvulares activas.

![Matriz de ecocardiograma](assets/output_heatmap.png)

### Imputación gaussiana normal natural y análisis de sensibilidad

- **Retención completa de la cohorte ($N=152$):** La imputación gaussiana normal natural preservó el $100\%$ de los registros de pacientes ($N=152$), rescatando $122$ casos de alta complejidad que los algoritmos de eliminación tradicionales habrían descartado.

- **Estabilidad de varianza en la estimación de densidad por kernel (KDE):** Los gráficos de densidad verifican que la imputación normal natural puebla los campos anatómicos no medidos alrededor de centros fisiológicos normales sin colapsar artificialmente la varianza en una media de estimación puntual única.

- **Desplazamientos lógicos de la media en auditorías de sensibilidad:** La comparación de la subcohorte medida original frente a la población totalmente imputada demostró desplazamientos esperados y no sesgados de la media (p. ej., el promedio de $LVIDd$ desplazándose hacia intervalos de referencia saludables) mientras se mantenía intacta la estratificación general de riesgo.

![Natural normal imputation](assets/output_kde_imputation.png)

### Índice de Carga Medicamentosa (MBI) como herramienta de triaje

- **Establecimiento de la "Constante Nativa" ($MBI = 2.27$):** El modelado estadístico identifica $2.27$ como la carga medicamentosa de línea base de la cohorte. Un paciente con esta puntuación representa un caso valvular estándar y estable.

- **El piso de complejidad ($MBI = 4.0$):** Establecer el umbral de entrada para intervenciones de alta prioridad en 4.0 captura el 25% superior de la severidad de la cohorte. Por ejemplo, combinar la Constante Nativa de línea base ($2.27$) con el coeficiente de riesgo aditivo de una lesión mitral mixta ($2.22$) da como resultado $MBI \approx 4.5$, lo que demuestra que un MBI de $4.0$ aisla matemáticamente fenotipos complejos de lesiones múltiples que requieren un soporte farmacológico intenso.

- **La alerta roja del umbral crítico ($MBI = 5.25$):** Superar un MBI de $5.25$ marca el punto donde la terapia médica ya no logra enmascarar la enfermedad subyacente. Este punto de corte predice hipertensión pulmonar crítica ($RVSP > 60 mmHg$) con una precisión del $85\%$, capturando una cohorte con alta media de RVSP ($48.0 \pm 34.3 mmHg$) y una elevada prevalencia femenina ($88.2\%$).

![Zonas de MBI](assets/output_triage_zones.png)

## Recomendaciones

Con base en los hallazgos descritos anteriormente, se recomienda al equipo de la misión de cardiología considerar lo siguiente:

- **Operacionalizar $MBI = 5.25$ como una bandera roja en el punto de registro.** Calcular el MBI en la recepción. Los pacientes con puntuación $\ge 5.25$ deben omitir la fila general y ser derivados prioritariamente a ecocardiografía y evaluación por un especialista sénior.

- **Priorizar la Zona 2 ($MBI = 4.0 - 5.5$) para la selección primaria de casos de intervención.** Priorizar a los pacientes de la Zona 2 ($MBI = 4.0 - 5.5$) en las listas de candidatos a procedimientos intervencionistas y quirúrgicos, maximizando el rendimiento del procedimiento antes de que los pacientes pasen a una insuficiencia en etapa terminal ($MBI > 5.5$).

- **Desplegar alertas de "subtratamiento" para barreras de acceso de alta complejidad.** Cruzar la severidad ecocardiográfica cualitativa con la carga farmacológica para activar consultas prioritarias de farmacia y trabajo social para pacientes con lesiones valvulares graves pero con $MBI < 1.0$, dejando al descubierto barreras críticas de acceso a medicamentos.

## Supuestos y limitaciones

A lo largo del análisis, se establecieron varios supuestos para gestionar los desafíos de los datos. Estos supuestos y limitaciones se detallan a continuación:

- **Filtro de exclusión de cohorte:** Se excluyó a los pacientes pediátricos (<15 años) y a los pacientes con ecocardiogramas completamente normales ($N=187 \to 152$) para centrar el flujo de trabajo en la enfermedad cardíaca estructural y valvular del adulto.

- **Lógica de imputación MNAR:** Se asumió que los parámetros ecocardiográficos continuos no registrados se omitieron debido a una apariencia no patológica. Los campos faltantes se imputaron mediante distribuciones gaussianas parametrizadas por constantes normales de línea base ($\mu, \sigma$) definidas en ClinicalConfig.

- **Mapeo cualitativo ecocardiográfico:** Las descripciones cualitativas en las notas clínicas ("leve", "moderado", "grave") se estandarizaron a una escala numérica continua ($0.0$ a $3.0$) correspondiente a los grados de severidad de la Sociedad Americana de Ecocardiografía (ASE).