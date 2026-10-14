# DISEÑO, CONSTRUCCIÓN Y EVALUACIÓN EXPERIMENTAL DE UN PROTOTIPO DE INCUBADORA NEONATAL A ESCALA CON CONTROL TÉRMICO PID Y SUBSISTEMA DE PESAJE DIGITAL

**Autores:** Estudiantes de Ingeniería Biomédica  
**Filiación:** Facultad de Ingeniería, Universidad Militar Nueva Granada, VII Semestre  
**Asignatura:** Instrumentación Biomédica y Biosensores  

---

## RESUMEN
Se presenta la implementación técnica y el análisis experimental de un prototipo de incubadora neonatal a escala supervisado por un microcontrolador ESP32. El desarrollo integra un lazo de control térmico en lazo cerrado tipo Proporcional-Integral-Derivativo (PID) orientado a mantener una temperatura objetivo de 36.5 °C, empleando un termistor NTC de 10 kΩ como elemento censor y una etapa de potencia activada por Modulación por Ancho de Pulso (PWM) invertida para la conmutación de un elemento calefactor resistivo de 10 Ω / 20 W. En paralelo, el prototipo incorpora un sistema de monitoreo de peso continuo mediante una celda de carga de 2 kg acondicionada con el amplificador analógico-digital de 24 bits HX711, proyectando las lecturas en una pantalla LCD de 16x2 líneas sobre bus I2C. Durante la fase de pruebas de laboratorio, el sistema exhibió desviaciones en el seguimiento de la variable térmica y variaciones en la estabilidad de la masa medida. Se determinó que estas condiciones se derivaron de la respuesta no lineal del convertidor analógico-digital (ADC) interno del microprocesador, perturbaciones por desacoplamiento de masa común y ruido de conmutación en la etapa de potencia. En este documento se examina el comportamiento físico del sistema, la formulación matemática implementada, la causa de las desviaciones encontradas y la reestructuración requerida para la optimización del circuito.

**Palabras clave:** Incubadora neonatal, ESP32, Control PID, Termistor NTC, HX711, Instrumentación Biomédica.

---

## I. INTRODUCCIÓN

Las incubadoras neonatales constituyen equipos de soporte vital en unidades de cuidados intensivos, diseñados con la función de generar un microambiente aislado que reproduzca las condiciones fisiológicas uterinas. La termorregulación es un factor crítico en el recién nacido, dado que los prematuros presentan un desarrollo tegumentario incompleto, menor proporción de tejido graso pardo y una inmadurez del centro termorregulador en el sistema nervioso central. Esta condición reduce la capacidad de compensar pérdidas de calor por conducción, radiación, convección y evaporación. El margen térmico idóneo en neonatos se sitúa de forma estricta entre 36.0 °C y 37.5 °C. Desviaciones de este rango conllevan riesgos de hipotermia o hipertermia, afectando la tasa metabólica, el consumo de oxígeno y la saturación sistémica.

Por otra parte, la estimación continua de la masa corporal en el paciente es necesaria para controlar la evolución ponderal, el balance hídrico y la dosificación farmacológica, evitando retirar al infante de la cabina para prevenir episodios de estrés térmico o mecánico.

El presente proyecto abarca el diseño del circuito de acondicionamiento de señal, el montaje mecánico a escala, la programación del algoritmo de control y la evaluación experimental en laboratorio. A partir de las pruebas ejecutadas, se efectúa un diagnóstico técnico de los factores que limitaron la respuesta del prototipo frente a los parámetros teóricos exigidos en entornos biomédicos.

---

## II. MARCO TEÓRICO

### A. Termorregulación Fisiológica y Entorno Controlado
El recién nacido prematuro exhibe un balance térmico frágil. La pérdida de calor hacia el entorno ocurre mediante cuatro mecanismos físicos:

$$\text{Q}_{\text{total}} = \text{Q}_{\text{conducción}} + \text{Q}_{\text{convección}} + \text{Q}_{\text{radiación}} + \text{Q}_{\text{evaporación}}$$

En la incubadora, la variable de mayor impacto para contrarrestar esta pérdida es la transferencia por convección forzada. Se hace circular un flujo de aire templado generado por la interacción entre un ventilador de baja velocidad y una resistencia eléctrica. La estabilidad de este proceso exige sostener la temperatura interna dentro del rango de neutralidad térmica.

---

### B. Caracterización del Sensor Térmico NTC
Los termistores de Coeficiente de Temperatura Negativo (NTC) varían su resistencia eléctrica de forma inversamente proporcional a la temperatura absoluta. Esta variación responde a la ecuación de Steinhart-Hart, la cual se simplifica mediante el modelo del parámetro Beta ($\beta$):

$$R_{\text{NTC}}(T) = R_0 \cdot \exp \left( \beta \left( \frac{1}{T} - \frac{1}{T_0} \right) \right)$$

Donde $R_0 = 10\text{ k}\Omega$ es la resistencia medida a la temperatura de referencia $T_0 = 298.15\text{ K}$ ($25\ ^\circ\text{C}$), y $\beta = 3950\text{ K}$ define la constante del material. Para obtener un voltaje proporcional en el microcontrolador, el sensor se ubica dentro de un divisor de tensión alimentado a $V_{cc} = 3.3\text{V}$ junto a una resistencia fija $R_{\text{ref}} = 10\text{ k}\Omega$:

$$V_{\text{out}} = V_{cc} \cdot \left( \frac{R_{\text{ref}}}{R_{\text{NTC}} + R_{\text{ref}}} \right)$$

---

### C. Algoritmo de Control Térmico en Lazo Cerrado (PID)
El algoritmo PID evalúa de forma constante la diferencia entre la temperatura de referencia ($T_{\text{setpoint}} = 36.5\ ^\circ\text{C}$) y el valor medido por el sensor ($T(t)$). El error instantáneo se define como:

$$e(t) = T_{\text{setpoint}} - T(t)$$

La señal de control analítica se calcula mediante la combinación de las tres acciones:

$$u(t) = K_p \cdot e(t) + K_i \int_{0}^{t} e(\tau) \, d\tau + K_d \cdot \frac{de(t)}{dt}$$

1. **Término Proporcional ($K_p$):** Proporciona una respuesta inmediata en función de la magnitud del error instantáneo.
2. **Término Integral ($K_i$):** Acumula los errores pasados a lo largo del tiempo para eliminar la desviación en estado estable (*offset*).
3. **Término Derivativo ($K_d$):** Evalúa la velocidad de cambio del error para prever sobrepicos y amortiguar la respuesta.

La implementación en código requiere la discretización del algoritmo para un tiempo de muestreo $\Delta t \approx 0.1\text{ s}$:

$$I(k) = I(k-1) + e(k) \cdot \Delta t$$

$$D(k) = \frac{e(k) - e(k-1)}{\Delta t}$$

$$u(k) = K_p \cdot e(k) + K_i \cdot I(k) + K_d \cdot D(k)$$

---

### D. Principio de Pesaje Digital mediante Celdas de Carga y Módulo HX711
El pesaje del neonato se basa en una celda de carga de tipo barra dispuesta con galgas extensométricas en puente de Wheatstone. Ante una fuerza ejercida sobre la barra, las galgas experimentan una deformación mecánica que altera su resistencia interna, generando una diferencia de potencial en el rango de los milivoltios.

El módulo **HX711** realiza la amplificación del diferencial de voltaje y su posterior conversión analógica-digital mediante un convertidor Σ-Δ (*Sigma-Delta*) de 24 bits. La masa resultante $M$ en kilogramos se obtiene a partir de la resta de la lectura en vacío ($L_{\text{tara}}$) y un factor de escala $k$:

$$M = \frac{L_{\text{actual}} - L_{\text{tara}}}{k} \cdot \frac{1}{1000}$$

---

## III. METODOLOGÍA Y DESARROLLO EXPERIMENTAL

El diseño se estructuró en tres fases de desarrollo: acondicionamiento y simulación (Parte A), integración de la estructura física y firmware (Parte B), y validación operativa en laboratorio (Parte C).

```text
+-------------------------------------------------------------------+
|                            FASE A                                 |
|  - Cálculo y simulación del divisor de tensión para NTC (10k)     |
|  - Etapa de potencia PWM invertida (Transistor 2N2222 / MOSFET)   |
|  - Diseño de la interfaz I2C para pantalla LCD y módulo HX711    |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                            FASE B                                 |
|  - Ensamble de la cabina a escala (material traslúcido)           |
|  - Calibración de tara y factor de escala (5.405) en HX711        |
|  - Integración del código unificado PID y muestreo en ESP32       |
+-------------------------------------------------------------------+
                                  |
                                  v
+-------------------------------------------------------------------+
|                            FASE C                                 |
|  - Pruebas de estabilización térmica a Setpoint = 36.5 °C         |
|  - Pruebas de medición de masa con carga patrón de 2.00 kg        |
|  - Diagnóstico técnico de fallas y análisis de desacoplamiento    |
+-------------------------------------------------------------------+
