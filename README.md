# DISEÑO, CONSTRUCCIÓN Y EVALUACIÓN EXPERIMENTAL DE UN PROTOTIPO DE INCUBADORA NEONATAL A ESCALA CON CONTROL TÉRMICO PID Y SUBSISTEMA DE PESAJE DIGITAL

**Autores:** Estudiantes de Ingeniería Biomédica  
**Filiación:** Facultad de Ingeniería, Universidad Militar Nueva Granada, VII Semestre  
**Asignatura:** Instrumentación Biomédica y Biosensores  

---

## RESUMEN
Se presenta la implementación técnica y el análisis experimental de un prototipo de incubadora neonatal a escala supervisado por un microcontrolador ESP32. El desarrollo integra un lazo de control térmico en lazo cerrado tipo Proporcional-Integral-Derivativo (PID) orientado a mantener una temperatura objetivo de 36.5 °C, empleando un termistor NTC de 10 kΩ como elemento censor y una etapa de potencia activada por Modulación por Ancho de Pulso (PWM) invertida para la conmutación de un elemento calefactor resistivo de 10 Ω / 20 W. En paralelo, el prototipo incorpora un sistema de monitoreo de peso continuo mediante una celda de carga de 5 kg acondicionada con el amplificador analógico-digital de 24 bits HX711, proyectando las lecturas en una pantalla LCD de 16x2 líneas sobre bus I2C. Durante la fase de pruebas de laboratorio, el sistema exhibió desviaciones en el seguimiento de la variable térmica y variaciones en la estabilidad de la masa medida. Se determinó que estas condiciones se derivaron de la respuesta no lineal del convertidor analógico-digital (ADC) interno del microprocesador, perturbaciones por desacoplamiento de masa común y ruido de conmutación en la etapa de potencia. En este documento se examina el comportamiento físico del sistema, la formulación matemática implementada, la causa de las desviaciones encontradas y la reestructuración requerida para la optimización del circuito.

**Palabras clave:** Incubadora neonatal, ESP32, Control PID, Termistor NTC, HX711, Instrumentación Biomédica.

---

# I. INTRODUCCIÓN

Las incubadoras neonatales son equipos médicos diseñados para crear un microclima cerrado que simule las condiciones del vientre materno [1, 2]. En prematuros o recién nacidos de bajo peso, la piel es delgada, el tejido graso es mínimo y el sistema termorregulador no ha madurado, por lo que pierden calor rápidamente a través de la piel y la respiración [3, 4]. Si la temperatura del entorno sale del rango de 36.0 °C a 37.5 °C, el neonato gasta energía crítica para intentar compensarlo, lo que incrementa el riesgo de complicaciones graves y morbimortalidad [3, 5]. Por este motivo, el control preciso de la temperatura y el seguimiento del peso son dos funciones básicas en cualquier entorno de cuidado intensivo neonatal [1, 3].

El calentamiento dentro de una incubadora se logra haciendo circular aire caliente mediante convección forzada [1]. Para medir esta variable se utilizan sensores NTC [1, 2], cuya resistencia eléctrica cambia en función de la temperatura según el modelo Beta:

$$
R_{\text{NTC}}(T) = R_0 \cdot \exp \left( \beta \left( \frac{1}{T} - \frac{1}{T_0} \right) \right)
$$

A la par del control térmico, la medición de peso permite evaluar el desarrollo del paciente sin sacarlo de la cubierta [1, 2]. Esta tarea se realiza con celdas de carga basadas en el puente de Wheatstone, cuya señal analógica en milivoltios es amplificada por convertidores de alta resolución como el módulo HX711 de 24 bits [1].

El objetivo de este laboratorio fue diseñar, construir y probar un prototipo a escala de incubadora neonatal utilizando un microcontrolador ESP32, un algoritmo de control térmico PID y un sistema de pesaje digital con pantalla LCD I2C [1, 2]. Específicamente, se buscó acondicionar la señal del NTC con un divisor de tensión [1], programar la respuesta PID mediante PWM invertida para una resistencia calefactora de 10 Ω / 20 W [1], y calibrar la celda de carga para medir pesos de hasta 2.00 kg aplicando un factor de corrección de 5.405 [1]. Se planteó como hipótesis que el sistema mantendría la temperatura en 36.5 °C con variaciones menores a ± 0.5 °C [1, 3] y registraría la masa de prueba de forma estable.

La importancia de este proyecto radicó en enfrentar de forma práctica los problemas reales de la instrumentación electrónica [1, 2]. Durante las pruebas se identificaron las limitaciones del convertidor analógico-digital (ADC) de la ESP32, la inercia térmica dentro de la cabina y el ruido eléctrico que la etapa de potencia inyectó en la lectura del peso [1]. 

---

# II. MARCO TEÓRICO

## A. Termorregulación Fisiológica y Vulnerabilidad Neonatal
La termorregulación es el proceso fisiológico mediante el cual el organismo mantiene un equilibrio dinámico entre la producción metabólica de calor y las pérdidas hacia el entorno[cite: 5]. En el recién nacido, y con mayor severidad en el neonato prematuro, esta capacidad se encuentra gravemente comprometida por factores anatómicos, metabólicos y neurológicos[cite: 5, 10]:

* **Relación superficie-masa elevada:** La amplia superficie corporal en proporción con el volumen o peso genera una tasa de pérdida calórica por unidad de masa significativamente mayor[cite: 5].
* **Inmadurez de la barrera epidérmica:** La piel del prematuro cuenta con una capa córnea delgada e hipervascularizada, lo que facilita la pérdida rápida de temperatura y la evaporación transepidérmica de fluidos[cite: 5, 10].
* **Deficiencia de tejido adiposo pardo:** La termogénesis química o sin tiritar (*non-shivering thermogenesis*) depende de la oxidación del tejido adiposo pardo, cuyos depósitos son mínimos o inexistentes en infantes nacidos antes de las 32 semanas de gestación[cite: 5, 10].
* **Centro vasomotor inmaduro:** La respuesta autónoma de vasoconstricción periférica ante estímulos térmicos de frío no se ejecuta con la velocidad ni la intensidad requeridas para retener el calor corporal[cite: 5, 10].

El concepto central en el cuidado neonatal es el Ambiente Térmico Neutro (ATN)[cite: 5]. Se define como el rango de temperatura ambiental dentro del cual el recién nacido sostiene su temperatura corporal central dentro de los límites fisiológicos normotérmicos (36.5 °C a 37.5 °C)[cite: 8, 10], minimizando el consumo de oxígeno y el gasto metabólico basal[cite: 5]. La pérdida calórica se rige por cuatro mecanismos físicos fundamentales:

$$\dot{Q}_{\text{pérdida}} = \dot{Q}_{\text{conducción}} + \dot{Q}_{\text{convección}} + \dot{Q}_{\text{radiación}} + \dot{Q}_{\text{evaporación}}$$

* **Conducción:** Transferencia de calor por contacto directo del cuerpo del neonato con superficies rígidas o frías, como la colchoneta o las sábanas[cite: 5].
* **Convección:** Transferencia calórica producida por el flujo de aire que circula directamente sobre la superficie cutánea del prematuro[cite: 5].
* **Radiación:** Emisión de energía en forma de ondas infrarrojas hacia las paredes internas de la cabina cuando estas se encuentran a menor temperatura que la piel[cite: 5].
* **Evaporación:** Pérdida de calor latente asociada al cambio de fase del agua a través de la piel y del tracto respiratorio[cite: 5].

---

## B. Identificación Detallada de las Partes y Subsistemas de una Incubadora Neonatal
Una incubadora neonatal es un dispositivo biomédico complejo diseñado para replicar de forma artificial el microclima uterino[cite: 5, 10]. Se compone de múltiples subsistemas mecánicos, mecátronicos, de acondicionamiento y de interfaz[cite: 5, 8]:

* **Capota o Cabina de Aislamiento:** Corresponde a la cubierta transparente fabricada habitualmente en acrílico de alta resistencia mecánica[cite: 8]. En prototipos a escala, mantiene dimensiones estandarizadas de 90 cm x 40 cm x 45 cm[cite: 8]. Su función principal es delimitar el volumen del microclima, aislar al neonato de corrientes de aire externas, reducir pérdidas radiativas/convectivas y garantizar visibilidad constante[cite: 5, 8]. Incorpora un sistema de apertura/cierre y portezuelas de acceso (*irises*) para permitir la manipulación médica sin generar caídas drásticas de temperatura[cite: 8].
* **Modulo Calefactor (Elemento Resistivo):** Es la fuente primaria de generación térmica, constituida por una resistencia eléctrica de potencia (10 Ω / 20 W en prototipos a escala)[cite: 5, 8]. Funciona bajo el principio del efecto Joule ($\dot{Q} = I^2 R$), convirtiendo la corriente suministrada en energía calórica para calentar el volumen de aire circulante[cite: 5, 8].
* **Turbina o Ventilador de Convección Forzada:** Elemento electromecánico encargado de aspirar el aire del interior o exterior, forzándolo a pasar por el elemento calefactor[cite: 5, 8]. Su función es generar una corriente laminar uniforme que distribuya el calor homogéneamente en toda la cabina, eliminando puntos fríos o gradientes de temperatura peligrosos[cite: 5, 8].
* **Cámara de Humidificación:** Depósito de agua con un sistema de evaporación controlado que incrementa la humedad relativa dentro de la cabina a valores entre 70% y 85%[cite: 5]. Esto evita la deshidratación epidérmica y atenúa la pérdida calórica evaporativa[cite: 5].
* **Báscula y Sistema de Pesaje Interno:** Estructura mecánica dispuesta bajo la colchoneta que integra celdas de carga[cite: 8]. Permite estimar la masa corporal del recién nacido sin necesidad de extraerlo de la incubadora, evitando exponerlo a choques térmicos[cite: 5, 8].
* **Subsistema de Sensado Térmico:** Sondas de temperatura basadas en termistores NTC colocados en el ambiente interno y en la piel del paciente[cite: 5, 7]. Proveen la señal de realimentación para que el procesador evalúe si la temperatura se mantiene dentro del rango objetivo[cite: 5, 8].
* **Unidad de Procesamiento y Control:** Tarjeta electrónica (microcontrolador ESP32) que ejecuta en tiempo real el algoritmo de control de lazo cerrado (PID)[cite: 5, 8]. Interpreta las lecturas de los sensores y regula la potencia del calefactor mediante señales moduladas[cite: 5, 8].
* **Interfaz de Usuario (HMI) y Panel de Alertas:** Pantalla LCD (I2C) y panel indicador visual mediante LEDs[cite: 8]. Su función es mostrar la temperatura actual, el setpoint y el peso en kilogramos, además de activar códigos de color (LED verde en rango normal 36.0 °C - 37.5 °C; LED rojo para temperaturas fuera de rango) que alerten al personal[cite: 5, 8].

---

## C. Caracterización del Sensor NTC y Acondicionamiento por Divisor de Tensión
Los termistores NTC (*Negative Temperature Coefficient*) son componentes semiconductores cuya resistencia decrece de manera exponencial a medida que incrementa la temperatura absoluta[cite: 5]. La respuesta del sensor se modela mediante la aproximación del parámetro Beta ($\beta$) de la ecuación de Steinhart-Hart[cite: 5]:

$$R_{\text{NTC}}(T) = R_0 \cdot \exp \left( \beta \left( \frac{1}{T} - \frac{1}{T_0} \right) \right)$$

En esta expresión, $R_0 = 10\text{ k}\Omega$ es la resistencia nominal a la temperatura de referencia $T_0 = 298.15\text{ K}$ ($25\ ^\circ\text{C}$), y $\beta = 3950\text{ K}$ es la constante característica del material semiconductor[cite: 5]. Despejando la temperatura $T$ en Kelvin en función de la resistencia $R_{\text{NTC}}$ calculada:

$$\frac{1}{T} = \frac{1}{T_0} + \frac{1}{\beta} \ln\left(\frac{R_{\text{NTC}}}{R_0}\right) \quad \implies \quad T_{(^\circ\text{C})} = T - 273.15$$

Para adaptar la variación de resistencia a un voltaje legible por la ESP32 (convertidor ADC de 12 bits con rango de 0 a 4095), se utiliza un divisor de tensión alimentado a $V_{cc} = 3.3\text{V}$ junto a una resistencia de referencia $R_{\text{ref}} = 10\text{ k}\Omega$[cite: 7]:

$$V_{\text{ADC}} = V_{cc} \cdot \left( \frac{R_{\text{ref}}}{R_{\text{NTC}} + R_{\text{ref}}} \right) \quad \implies \quad R_{\text{NTC}} = R_{\text{ref}} \cdot \left( \frac{V_{cc} - V_{\text{ADC}}}{V_{\text{ADC}}} \right)$$

Expresado directamente en cuentas digitales del ADC ($N_{\text{ADC}} = \frac{V_{\text{ADC}}}{V_{cc}} \cdot 4095$):

$$R_{\text{NTC}} = 10000 \cdot \left( \frac{N_{\text{ADC}}}{4095 - N_{\text{ADC}}} \right)$$

---

## D. Control de Temperatura PID y Modulación PWM Invertida
La regulación de la temperatura en lazo cerrado se ejecuta comparando continuamente la temperatura deseada ($T_{\text{setpoint}} = 36.5\ ^\circ\text{C}$) con el valor medido ($T(t)$)[cite: 5, 8], generando una variable de error:

$$e(t) = T_{\text{setpoint}} - T(t)$$

El algoritmo Proporcional-Integral-Derivativo (PID) combina tres acciones matemáticas para obtener la señal de control $u(t)$:

$$u(t) = K_p \, e(t) + K_i \int_{0}^{t} e(\tau) \, d\tau + K_d \, \frac{de(t)}{dt}$$

* **Acción Proporcional ($K_p e(t)$):** Aplica una fuerza de corrección proporcional a la magnitud del error instantáneo. Un valor elevado acelera la respuesta pero causa oscilaciones.
* **Acción Integral ($K_i \int e(\tau)d\tau$):** Suma los errores pasados a lo largo del tiempo para eliminar el error en estado estable (*offset*). Se aplica una restricción de acumulación (*anti-windup*: $-100 \le I_{\text{acum}} \le 100$) para evitar la saturación del integrador.
* **Acción Derivativa ($K_d \frac{de}{dt}$):** Mide la velocidad de cambio del error para prever sobrepicos (*overshoots*) y amortiguar la respuesta térmica ante cambios repentinos.

Para su ejecución en microcontroladores, el algoritmo se discretiza para un periodo de muestreo $\Delta t \approx 0.1\text{ s}$:

$$u[k] = K_p e[k] + K_i \sum_{j=0}^{k} e[j]\Delta t + K_d \left(\frac{e[k] - e[k-1]}{\Delta t}\right)$$

La salida $u[k]$ se limita al rango de 8 bits (0 a 255). Debido a que la etapa de potencia incorpora un transistor BJT 2N2222 que invierte la fase lógica antes de excitar la compuerta del MOSFET IRFZ44N, la señal de Modulación por Ancho de Pulso (PWM) se programa con **lógica invertida**:

$$
\text{PWM}_{\text{salida}} = 255 - \text{sat}(u[k], 0, 255)
$$

---

## E. Principio de Pesaje Digital: Puente de Wheatstone y Módulo HX711
El sistema de estimación de peso se fundamenta en una celda de carga de aluminio con cuatro galgas extensométricas (*strain gauges*) configuradas en un **Puente de Wheatstone completo**[cite: 8]. Cuando una masa $M$ se ubica sobre la plataforma, la deformación mecánica altera la resistencia eléctrica de las galgas ($R \pm \Delta R$), generando un diferencial de voltaje en el orden de los milivoltios ($\text{mV}$)[cite: 8]:

$$V_{\text{diferencial}} = V_{\text{excitación}} \cdot \left( \frac{\Delta R}{R} \right)$$

Debido a la baja amplitud de este voltaje, se acopla el circuito integrado **HX711**, el cual incorpora un preamplificador de bajo ruido con ganancia de 128 y un convertidor analógico-digital (ADC) $\Sigma-\Delta$ de 24 bits, entregando una resolución de $2^{24} = 16,777,216$ niveles[cite: 8].

La masa $M$ en kilogramos ($\text{kg}$) se obtiene descontando el peso estático inicial ($L_{\text{tara}}$) y aplicando el factor de calibración de escala $k_{\text{escala}} = 5.405$:

$$M_{(\text{kg})} = \frac{(L_{\text{bruta}} - L_{\text{tara}}) \cdot k_{\text{escala}}}{1000}$$

Para reducir el ruido de conmutación e interferencias electromagnéticas, se aplica un filtro de promedio móvil sobre 10 muestras:

$$\bar{L} = \frac{1}{10} \sum_{i=1}^{10} L_i$$

---

## F. Análisis Comparativo de Costos y Soluciones Comerciales
El desarrollo de tecnología biomédica implica evaluar la relación entre el costo de producción y las prestaciones operativas de seguridad exigidas por normas internacionales (e.g., IEC 60601-2-19 para incubadoras)[cite: 8, 9, 10]:

* **Sistemas Comerciales (e.g., Dräger Medical, LEEX Medical, Instrumentalia S.A.S.):** Tienen un costo de mercado que oscila entre **$15,000 USD y $35,000 USD** ($\approx 60.000.000$ a $140.000.000\text{ COP}$)[cite: 8, 9]. Ofrecen cabinas de doble pared acrílica para reducir pérdidas por radiación, servocontrol de humedad mediante evaporadores independientes, filtrado de aire de alta eficiencia (HEPA), básculas integradas con autorrango de alta precisión y circuitos redundantes de corte térmico por hardware independientes del procesador principal[cite: 5, 8, 9].
* **Prototipo a Escala Desarrollado:** El costo total de materiales del prototipo ensamblado asciende a **$177,500 COP** (incluyendo tarjeta ESP32, celda de carga de 2 kg con HX711, NTC, elementos de potencia, estructura acrílica y LCD I2C)[cite: 8, 9]. Aunque este desarrollo demuestra la viabilidad de implementar control PID y pesaje digital a un costo sumamente bajo ($<0.3\%$ del valor de un equipo comercial)[cite: 8, 9], carece de la redundancia, el aislamiento galvánico y la precisión necesarios para una aplicación clínica real sin un proceso extenso de rediseño biomédico.

---

# III. METODOLOGÍA Y DESARROLLO EXPERIMENTAL

El desarrollo del prototipo de incubadora neonatal a escala se dividió en tres etapas interconectadas: el diseño de las etapas analógicas de acondicionamiento y lógica de control (Parte A), la integración física de la cabina y la programación del microcontrolador ESP32 (Parte B), y la realización de pruebas de estabilización e instrumentación en laboratorio (Parte C).

---

## A. Arquitectura del Sistema
Se utilizó un microcontrolador ESP32 como unidad principal para la adquisición de datos, ejecución del algoritmo de control y despliegue de información. La distribución de los pines analógicos, digitales y buses de comunicación para los módulos del prototipo se consolida en la Tabla I.

### Tabla I: Asignación de Pines y Periféricos en la ESP32

| Componente | Pin ESP32 | Protocolo | Función Técnica |
| :--- | :--- | :--- | :--- |
| **Sensor NTC (10 kΩ)** | GPIO 34 | ADC1_CH6 (Entrada Analógica) | Captura del voltaje variable en el divisor resistivo |
| **Driver de Potencia** | GPIO 25 | Salida Digital / PWM | Conmutación de la etapa de potencia del calefactor |
| **HX711 DOUT** | GPIO 19 | Entrada Digital | Recepción de datos serie de 24 bits de la celda de carga |
| **HX711 SCK** | GPIO 18 | Salida Digital | Reloj de sincronización para la lectura del HX711  |
| **LCD I2C (SDA)** | GPIO 21 | Bus I2C (SDA) | Transmisión de datos para la pantalla LCD 16x2 |
| **LCD I2C (SCL)** | GPIO 22 | Bus I2C (SCL) | Reloj del bus I2C para la pantalla LCD  |

---

## B. Acondicionamiento Analógico del Sensor de Temperatura
Para monitorear la variable térmica dentro del recipiente, se diseñó un circuito divisor de tensión alimentado a 3.3 V compuesto por un termistor NTC de $10\text{ k}\Omega$ a $25\ ^\circ\text{C}$ ($\beta = 3950\text{ K}$) y una resistencia fija de precisión $R_{\text{ref}} = 10\text{ k}\Omega$ [2]. 

La tensión leída por el puerto analógico ADC1_CH6 de la ESP32 se transforma en la resistencia instantánea del sensor ($R_{\text{NTC}}$) mediante la relación del divisor:

$$
R_{\text{NTC}} = R_{\text{ref}} \cdot \left( \frac{V_{cc} - V_{\text{ADC}}}{V_{\text{ADC}}} \right) = 10000 \cdot \left( \frac{N_{\text{ADC}}}{4095 - N_{\text{ADC}}} \right)
$$

donde $N_{\text{ADC}}$ representa la lectura digital de 12 bits ($0$ a $4095$). La conversión de resistencia a temperatura absoluta $T$ en Kelvin se realiza evaluando la ecuación del modelo Beta de Steinhart-Hart [4]:

$$
T = \frac{1}{\frac{1}{T_0} + \frac{1}{\beta} \ln\left(\frac{R_{\text{NTC}}}{R_0}\right)}
$$

con $T_0 = 298.15\text{ K}$ ($25\ ^\circ\text{C}$) y $R_0 = 10000\ \Omega$. Finalmente, la temperatura en grados Celsius se obtiene al restar $273.15$.

---

## C. Etapa de Potencia y Modulación PWM
El elemento de calefacción consiste en una resistencia de $10\ \Omega / 20\text{W}$ acoplada a un ventilador de $12\text{V}$ DC para impulsar el flujo de aire por convección [1, 2]. Puesto que las salidas digitales de la ESP32 entregan un máximo de $12\text{ mA}$ a $3.3\text{V}$, se requirió un circuito de acoplamiento de potencia [2].

La señal PWM enviada por el GPIO 25 se conectó a la base de un transistor BJT 2N2222 con una resistencia de imitación de $1\text{ k}\Omega$. Este transistor opera como etapa preamplificadora para activar la compuerta de un MOSFET IRFZ44N conectado a la fuente principal de $12\text{V}$. Este arreglo produce una inversión de la señal de control:
* Un nivel lógico ALTO en la ESP32 satura el transistor BJT, llevando la compuerta del MOSFET a tierra (desconectando el calefactor).
* Un nivel lógico BAJO en la ESP32 corta el transistor BJT, permitiendo la activación de la compuerta del MOSFET (conectando el calefactor).

Debido a esta configuración física, la salida del algoritmo PID se calcula con lógica invertida en el firmware:

$$
\text{PWM}_{\text{aplicado}} = 255 - \text{sat}(u[k], 0, 255)
$$

---

## D. Algoritmo de Control PID Discreto
La regulación automática de la temperatura ambiente se estructuró mediante un algoritmo PID en tiempo discreto con un período de muestreo de $\Delta t \approx 0.1\text{ s}$. El error de temperatura se calcula respecto al valor de consigna dentro del rango de confort neutro ($T_{\text{setpoint}} = 36.5\ ^\circ\text{C}$) [3]:

$$
e[k] = T_{\text{setpoint}} - T[k]
$$

La acción de control $u[k]$ combina de forma matemática las tres componentes del controlador:

$$
u[k] = K_p \cdot e[k] + K_i \sum_{j=0}^{k} e[j]\Delta t + K_d \left( \frac{e[k] - e[k-1]}{\Delta t} \right)
$$

Los valores de ganancia sintonizados experimentalmente para el prototipo fueron $K_p = 15.0$, $K_i = 0.5$ y $K_d = 2.0$. Para evitar la saturación de la suma acumulada (*anti-windup*), se impuso un límite estricto a la componente integral:

$$
-100 \le \sum e[j]\Delta t \le 100
$$

---

## E. Subsistema de Pesaje y Calibración del Módulo HX711
El pesaje continuo del paciente dentro del entorno controlado se resolvió mediante una celda de carga de aluminio de $2\text{ kg}$ instalada bajo la base del colchón [1]. Las galgas extensométricas dispuestas en puente de Wheatstone entregan variaciones de milivoltios al integrado HX711 [1].

Para atenuar las oscilaciones causadas por vibraciones de la estructura o por el viento del ventilador, se aplicó un filtro digital de promedio sobre 20 lecturas consecutivas. La conversión de la señal cruda a peso en gramos se calcula restando la masa estática en vacío ($L_{\text{tara}}$) y aplicando el factor de calibración obtenido en laboratorio ($5.405$) [2]:

$$
\text{Masa}_{(\text{g})} = (L_{\text{lectura}} - L_{\text{tara}}) \cdot 5.405
$$

El valor final se procesa dividiendo entre $1000$ para proyectar la masa en kilogramos ($\text{kg}$) sobre la pantalla LCD.

---

## F. Estructura Mecánica de la Cabina
La cabina del prototipo se construyó utilizando paneles de acrílico transparente con dimensiones a escala de $90\text{ cm} \times 40\text{ cm} \times 45\text{ cm}$ [1, 2]. Esta envolvente reduce las pérdidas térmicas causadas por corrientes de aire del recinto y cuenta con una portezuela abatible que permite simular el acceso médico sin exponer la totalidad de la cámara al exterior [1].

---

## G. Estructura, Análisis del Firmware e Implementación en C++ (ESP32)

El programa desarrollado para la tarjeta ESP32 se estructuró en C++ utilizando un modelo de ejecución secuencial en bucle (`loop`). Este firmware ejecuta de forma integrada cuatro tareas esenciales: la adquisición y conversión analógica de la temperatura, la evaluación del algoritmo de control PID discreto, la lectura de la masa corporal mediante la celda de carga y la actualización de la interfaz visual en la pantalla LCD [1, 2].

---

### 1. Desglose y Análisis del Firmware

#### a. Inclusión de Librerías y Asignación de Pines
El programa utiliza la librería `math.h` para el cálculo del logaritmo natural (`log()`) necesario en la conversión del termistor NTC mediante la ecuación de Steinhart-Hart [4]. Para la pantalla LCD se emplean `Wire.h` y `LiquidCrystal_I2C.h`, permitiendo la comunicación por bus I2C utilizando solo dos líneas de datos. Por su parte, la comunicación con el módulo de pesaje de 24 bits se realiza a través de la librería `DFRobot_HX711.h` [1, 2].

La distribución fija de pines en la ESP32 queda configurada de la siguiente manera:
* `NTC_PIN (GPIO 34)`: Canal analógico de entrada (ADC1_CH6).
* `HEATER_PIN (GPIO 25)`: Salida de Modulación por Ancho de Pulso (PWM) hacia la etapa de potencia [1].
* `HX711_DOUT (GPIO 19)` y `HX711_SCK (GPIO 18)`: Pines de datos serie y reloj para la celda de carga [1].

#### b. Configuración e Inicialización (`setup`)
En el bloque `setup()`, se inicializa el monitor serie a 115200 baudios para la depuración de datos. El pin `HEATER_PIN` se define como salida digital y se fuerza a un estado ALTO (`HIGH`). Esta acción es crítica por seguridad: dado que la etapa de potencia con el transistor BJT 2N2222 invierte la señal lógica, un nivel ALTO inicial garantiza que la compuerta del MOSFET IRFZ44N permanezca a tierra, evitando que la resistencia calefactora de $10\ \Omega / 20\text{W}$ se encienda de manera accidental durante el arranque del microcontrolador [1, 2]. Posteriormente, se inicializa el bus I2C en los pines GPIO 21 (SDA) y GPIO 22 (SCL) y se muestra la pantalla de bienvenida.

#### c. Adquisición Analógica y Modelo Térmico
Para mitigar el ruido electromagnético y la no linealidad del convertidor ADC de 12 bits de la ESP32, se toma un promedio móvil de 20 muestras consecutivas con un retardo de $2\text{ ms}$ entre lecturas [2]. El valor digital promediado ($N_{\text{ADC}}$) se convierte a resistencia instantánea del termistor ($R_{\text{NTC}}$) aplicando la fórmula del divisor de tensión:

$$R_{\text{NTC}} = 10000 \cdot \left( \frac{N_{\text{ADC}}}{4095 - N_{\text{ADC}}} \right)$$

Posteriormente, la resistencia se transforma a grados Celsius (`tempC`) mediante el modelo Beta de Steinhart-Hart, utilizando los parámetros $T_0 = 298.15\text{ K}$, $R_0 = 10000\ \Omega$ y $\beta = 3950\text{ K}$ [4].

#### d. Algoritmo de Control PID Discreto e Inversión de Salida
El control térmico calcula el error restando la temperatura actual del valor de consigna (`setpoint = 36.5` °C) [3]. El intervalo de tiempo $\Delta t$ transcurrido entre iteraciones se calcula dinámicamente con la función `millis()`.

La componente integral acumula el error en el tiempo. Para evitar el fenómeno de *windup* (saturación de la integral por desviaciones prolongadas), se delimita su acumulación entre $-100$ y $100$. La componente derivativa evalúa la velocidad de cambio del error respecto a $\Delta t$. La suma de las tres acciones ($K_p = 15.0$, $K_i = 0.5$, $K_d = 2.0$) genera la variable `outputPID`, la cual se satura en el rango de $0$ a $255$. Debido a la inversión de fase que introduce el driver transistorizado, la potencia calculada se aplica de forma invertida mediante `255 - outputPID` [1].

#### e. Lectura de Pesaje e Interfaz LCD
La lectura bruta transmitida por el circuito integrado HX711 se escala multiplicándola por el factor de calibración ajustado en laboratorio (`factor_correccion = 5.405`) [1, 2]. Si el resultado es menor a cero debido a la deriva de tara, la variable se fuerza a cero gramos y luego se divide entre 1000 para obtener la masa en kilogramos (`peso_kg`).

Finalmente, la pantalla LCD de 16x2 actualiza en la primera línea la temperatura medida y el punto de consigna, mientras que en la segunda línea despliega la masa corporal estimada y el porcentaje de trabajo del calefactor ($0\%$ a $100\%$) [2].

---

### 2. Código Fuente Completo en C++ (ESP32)

```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DFRobot_HX711.h>
#include <math.h>

// ============================================================================
// ASIGNACIÓN DE PINES Y PERIFÉRICOS (ESP32)
// ============================================================================
#define NTC_PIN 34        // Entrada analógica ADC1_CH6 (Divisor de tensión NTC)
#define HEATER_PIN 25     // Salida PWM hacia el driver de potencia transistorizado
#define HX711_DOUT 19     // Línea de datos serie del convertidor HX711 (24 bits)
#define HX711_SCK 18      // Línea de reloj de sincronización para el HX711

// ============================================================================
// INSTANCIAS DE OBJETOS Y PERIFÉRICOS
// ============================================================================
LiquidCrystal_I2C lcd(0x27, 16, 2);
DFRobot_HX711 MyScale(HX711_DOUT, HX711_SCK);

// ============================================================================
// PARÁMETROS FÍSICOS Y CONSTANTES DEL MODELO TÉRMICO
// ============================================================================
const float Beta = 3950.0;     // Constante Beta del termistor NTC
const float R0 = 10000.0;      // Resistencia nominal del NTC a 25 °C (10 kΩ)
const float Temp0 = 298.15;    // Temperatura de referencia en Kelvin (25 °C + 273.15)

// ============================================================================
// PARÁMETROS Y VARIABLES DEL CONTROLADOR PID
// ============================================================================
float setpoint = 36.5;         // Consigna de temperatura en el Ambiente Térmico Neutro (°C)
float Kp = 15.0;               // Ganancia Proporcional
float Ki = 0.5;                // Ganancia Integral
float Kd = 2.0;                // Ganancia Derivativa

float integral = 0.0;          // Acumulador de la componente integral
float last_error = 0.0;        // Registro del error anterior para el cálculo derivativo
unsigned long last_time = 0;   // Control del tiempo entre muestras (dt)

// ============================================================================
// PARÁMETROS DE CALIBRACIÓN DE PESAJE
// ============================================================================
float factor_correccion = 5.405; // Escala ajustada experimentalmente para la celda de carga

void setup() {
  Serial.begin(115200);

  // Configuración de la salida PWM (Seguridad: Nivel ALTO mantiene el calefactor APAGADO)
  pinMode(HEATER_PIN, OUTPUT);
  digitalWrite(HEATER_PIN, HIGH); 

  // Inicialización del bus I2C y la pantalla LCD 16x2
  Wire.begin(21, 22);
  lcd.init();
  lcd.backlight();

  // Pantalla de arranque inicial del sistema
  lcd.setCursor(0, 0);
  lcd.print("   INCUBADORA   ");
  lcd.setCursor(0, 1);
  lcd.print("SISTEMA ACTIVO ");
  delay(3000);
  lcd.clear();
}

void loop() {
  // --------------------------------------------------------------------------
  // 1. ADQUISICIÓN Y FILTRADO ANALÓGICO DE TEMPERATURA (NTC 10k)
  // --------------------------------------------------------------------------
  long sumaADC = 0;
  for (int i = 0; i < 20; i++) {
    sumaADC += analogRead(NTC_PIN);
    delay(2);
  }
  float rawADC = (float)sumaADC / 20.0;
  if (rawADC < 1.0) rawADC = 1.0; // Protección contra división por cero

  // Conversión de tensión a resistencia y aplicación del modelo Beta de Steinhart-Hart
  float R_ntc = 10000.0 * (rawADC / (4095.0 - rawADC));
  float tempK = 1.0 / ((1.0 / Temp0) + (log(R_ntc / R0) / Beta));
  float tempC = tempK - 273.15;

  // --------------------------------------------------------------------------
  // 2. ALGORITMO DE CONTROL PID DISCRETO
  // --------------------------------------------------------------------------
  unsigned long now = millis();
  float dt = (now - last_time) / 1000.0;
  if (dt <= 0) dt = 0.1;
  last_time = now;

  float error = setpoint - tempC;
  
  // Término Integral con restricción Anti-windup
  integral += error * dt;
  if (integral > 100.0) integral = 100.0;
  if (integral < -100.0) integral = -100.0;

  // Término Derivativo y suma de acciones
  float derivative = (error - last_error) / dt;
  last_error = error;

  float outputPID = (Kp * error) + (Ki * integral) + (Kd * derivative);

  // Saturación de la variable de control al rango PWM de 8 bits (0 - 255)
  if (outputPID > 255.0) outputPID = 255.0;
  if (outputPID < 0.0) outputPID = 0.0;

  // Aplicación de la señal PWM con lógica invertida por la etapa BJT/MOSFET
  int pwmInvertido = 255 - (int)outputPID;
  analogWrite(HEATER_PIN, pwmInvertido);

  // --------------------------------------------------------------------------
  // 3. ADQUISICIÓN Y ESCALADO DE PESAJE DIGITAL (HX711)
  // --------------------------------------------------------------------------
  float peso_g = MyScale.readWeight() * factor_correccion;
  if (peso_g < 0) peso_g = 0; 
  float peso_kg = peso_g / 1000.0;

  // --------------------------------------------------------------------------
  // 4. DESPLIEGUE EN PANTALLA LCD Y TELEMETRÍA POR PUERTO SERIE
  // --------------------------------------------------------------------------
  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(tempC, 1);
  lcd.print("C Set:");
  lcd.print(setpoint, 1);
  lcd.print("C");

  lcd.setCursor(0, 1);
  lcd.print("P:");
  lcd.print(peso_kg, 2);
  lcd.print("kg ");

  lcd.setCursor(10, 1);
  lcd.print("H:");
  lcd.print((outputPID / 255.0) * 100.0, 0);
  lcd.print("% ");

  Serial.print("Temp: "); Serial.print(tempC, 2);
  Serial.print(" C | PWM: "); Serial.print(outputPID);
  Serial.print(" | Peso: "); Serial.print(peso_kg, 2);
  Serial.println(" kg");

  delay(200);
}
---

# REFERENCIAS

* [1] C. G. K. Tran, A. Gibson, D. Wong, D. Tilahun, N. Selock, T. Good, and G. Rao, "Designing a low-cost multifunctional infant incubator," *J. Lab. Autom.*, vol. 19, no. 3, pp. 332–337, Jun. 2014. doi: 10.1177/2211068214530391[cite: 10].
* [2] L. Restrepo-Pérez, N. Durango-Londoño, N. E. Gómez-Suárez, F. González-Ramírez, y N. Rivera-Bonilla, "Prototipo de incubadora neonatal," *Revista Ingeniería Biomédica*, vol. 1, no. 1, pp. 55–59, 2007[cite: 10].
* [3] M. van Leeuwen *et al.*, *European Standards of Care for Newborn Health: Temperature management in newborn infants*, European Foundation for the Care of Newborn Infants (EFCNI), Nov. 2018[cite: 10].
* [4] R. B. Knobel-Dail, "Role of effective thermoregulation in premature neonates," *Research and Reports in Neonatology*, vol. 4, pp. 147–156, 2014. doi: 10.2147/RRN.S52377[cite: 10].
* [5] R. E. Black *et al.*, "Global, regional, and national causes of child mortality in 2008: a systematic analysis," *Lancet*, vol. 375, pp. 1969–1987, Jun. 2010. doi: 10.1016/S0140-6736(10)60549-1[cite: 10].
