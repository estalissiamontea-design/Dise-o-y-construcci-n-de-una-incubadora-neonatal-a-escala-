# DISEÑO, CONSTRUCCIÓN Y EVALUACIÓN EXPERIMENTAL DE UN PROTOTIPO DE INCUBADORA NEONATAL A ESCALA CON CONTROL TÉRMICO PID Y SUBSISTEMA DE PESAJE DIGITAL

Alissia Montealegre Quintero. 
Raul Alexander Peñuela Jimenez.
Silvia Lorena Vargas Rueda.

Instrumentación Biomédica y Biosensores  
Facultad de Ingeniería, Universidad Militar Nueva Granada, VII Semestre  


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
El programa utiliza la librería `math.h` para el cálculo del logaritmo natural (`log()`) necesario en la conversión del termistor NTC mediante la ecuación de Steinhart-Hart [4]. Para la pantalla LCD se emplean `Wire.h` y `LiquidCrystal_I2C.h`, permitiendo la comunicación por bus I2C utilizando solo dos líneas de datos. Por su parte, la comunicación con el módulo de pesaje de 24 bits se realiza a través de la librería `DFRobot_HX711.h`.

La distribución fija de pines en la ESP32 queda configurada de la siguiente manera:
* `NTC_PIN (GPIO 34)`: Canal analógico de entrada (ADC1_CH6).
* `HEATER_PIN (GPIO 25)`: Salida de Modulación por Ancho de Pulso (PWM) hacia la etapa de potencia.
* `HX711_DOUT (GPIO 19)` y `HX711_SCK (GPIO 18)`: Pines de datos serie y reloj para la celda de carga.

#### b. Configuración e Inicialización (`setup`)
En el bloque `setup()`, se inicializa el monitor serie a 115200 baudios para la depuración de datos. El pin `HEATER_PIN` se define como salida digital y se fuerza a un estado ALTO (`HIGH`). Esta acción es crítica por seguridad: dado que la etapa de potencia con el transistor BJT 2N2222 invierte la señal lógica, un nivel ALTO inicial garantiza que la compuerta del MOSFET IRFZ44N permanezca a tierra, evitando que la resistencia calefactora de $10\ \Omega / 20\text{W}$ se encienda de manera accidental durante el arranque del microcontrolador. Posteriormente, se inicializa el bus I2C en los pines GPIO 21 (SDA) y GPIO 22 (SCL) y se muestra la pantalla de bienvenida.

#### c. Adquisición Analógica y Modelo Térmico
Para mitigar el ruido electromagnético y la no linealidad del convertidor ADC de 12 bits de la ESP32, se toma un promedio móvil de 20 muestras consecutivas con un retardo de $2\text{ ms}$ entre lecturas [2]. El valor digital promediado ($N_{\text{ADC}}$) se convierte a resistencia instantánea del termistor ($R_{\text{NTC}}$) aplicando la fórmula del divisor de tensión:

$$R_{\text{NTC}} = 10000 \cdot \left( \frac{N_{\text{ADC}}}{4095 - N_{\text{ADC}}} \right)$$

Posteriormente, la resistencia se transforma a grados Celsius (`tempC`) mediante el modelo Beta de Steinhart-Hart, utilizando los parámetros $T_0 = 298.15\text{ K}$, $R_0 = 10000\ \Omega$ y $\beta = 3950\text{ K}$ [4].

#### d. Algoritmo de Control PID Discreto e Inversión de Salida
El control térmico calcula el error restando la temperatura actual del valor de consigna (`setpoint = 36.5` °C) [3]. El intervalo de tiempo $\Delta t$ transcurrido entre iteraciones se calcula dinámicamente con la función `millis()`.

La componente integral acumula el error en el tiempo. Para evitar el fenómeno de *windup* (saturación de la integral por desviaciones prolongadas), se delimita su acumulación entre $-100$ y $100$. La componente derivativa evalúa la velocidad de cambio del error respecto a $\Delta t$. La suma de las tres acciones ($K_p = 15.0$, $K_i = 0.5$, $K_d = 2.0$) genera la variable `outputPID`, la cual se satura en el rango de $0$ a $255$. Debido a la inversión de fase que introduce el driver transistorizado, la potencia calculada se aplica de forma invertida mediante `255 - outputPID`.

#### e. Interfaz LCD
La lectura bruta transmitida por el circuito integrado HX711 se escala multiplicándola por el factor de calibración ajustado en laboratorio (`factor_correccion = 5.405`). Si el resultado es menor a cero debido a la deriva de tara, la variable se fuerza a cero gramos y luego se divide entre 1000 para obtener la masa en kilogramos (`peso_kg`).

Finalmente, la pantalla LCD de 16x2 actualiza en la primera línea la temperatura medida y el punto de consigna, mientras que en la segunda línea despliega la masa corporal estimada y el porcentaje de trabajo del calefactor ($0\%$ a $100\%$).

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
```
---

# IV. ESTIMACIÓN DE PRECIOS Y COMPARATIVA CON SISTEMAS SIMILARES

El análisis económico permite evaluar la viabilidad financiera del prototipo frente a soluciones comerciales de grado médico y a otros proyectos de investigación. Para este proyecto, la etapa de alimentación y la estructura física se construyeron utilizando componentes discretos de electrónica básica e insumos de maquetación de bajo costo, reduciendo al mínimo la inversión inicial.

---

### A. Desglose Racional de Costos del Prototipo Desarrollado

La Tabla IV detalla el costo real de los componentes e insumos adquiridos en el comercio local para el ensamblaje del prototipo.

#### Tabla IV: Presupuesto Detallado de Materiales e Insumos del Prototipo

| Categoría Funcional | Componente / Insumo Específico | Cant. | Costo Unitario (COP) | Costo Subtotal (COP) |
| :--- | :--- | :---: | :---: | :---: |
| **Procesamiento e Interfaz** | Tarjeta de desarrollo ESP32 (30 pines) | 1 | 38,000 | 38,000 |
| | Pantalla LCD 16x2 con adaptador I2C integrado | 1 | 24,000 | 24,000 |
| **Etapa de Sensorización** | Celda de carga tipo barra de 5 kg de capacidad | 1 | 22,000 | 22,000 |
| | Módulo amplificador ADC de 24 bits HX711 | 1 | 12,000 | 12,000 |
| | Sensor NTC 10 kΩ (3950K) encapsulado | 1 | 3,500 | 3,500 |
| **Etapa de Potencia y Pasivos** | Resistencia de alambre cerámico 10 Ω / 20 W | 1 | 8,500 | 8,500 |
| | Transistor BJT 2N2222A (TO-92) | 2 | 800 | 1,600 |
| | Transistor MOSFET IRFZ44N de canal N | 1 | 4,500 | 4,500 |
| | Diodos de protección 1N4007 y kit de resistencias variadas | 1 kit | 5,000 | 5,000 |
| **Fuente de Alimentación Discreta** | Transformador de alimentación (salida AC) | 1 | 25,000 | 25,000 |
| | Puente rectificador de diodos | 1 | 3,000 | 3,000 |
| | Regulador de voltaje lineal  | 1 | 2,500 | 2,500 |
| | Condensadores electrolíticos y cerámicos de filtrado | 4 | 1,000 | 4,000 |
| **Conexión y Ensamble** | Protoboards de 830 puntos con bus de alimentación | 2 | 18,000 | 36,000 |
| | Paquete de cables Jumper multifilar (M-M, M-H, H-H x 40 und) | 2 | 14,000 | 28,000 |
| | Conectores tipo bornera de 2 pines | 4 | 1,200 | 4,800 |
| **Estructura y Chasis Económico** | Acrílico moldeable / retazos para ventana de observación | 1 | 18,000 | 18,000 |
| | Láminas de madera para base rígida de la galga | 1 | 12,000 | 12,000 |
| | Cartón paja/corrugado, láminas de foami y silicona de pegado | 1 kit | 15,000 | 15,000 |
| | Tornillería pequeña M3 para fijación de soporte | 1 kit | 3,000 | 3,000 |
| **TOTAL COSTO** | — | — | — | **270,400** |

---

### B. Comparativa de Precios con Sistemas Similares y Comerciales

Para poner en perspectiva el presupuesto final del prototipo ($270,400 COP), la Tabla V establece una comparación frente a las alternativas existentes en el mercado médico y en proyectos académicos de referencia.

#### Tabla V: Cuadro Comparativo de Incubadoras y Prototipos del Mercado

| Parámetro | Incubadora Comercial Gama Alta (ej. Dräger / YP2000) | Incubadora Clínica Estándar / Transporte (ej. YP920) | Prototipos de Investigación (Arduino / Raspberry Pi) | Prototipo Desarrollado (ESP32) |
| :--- | :--- | :--- | :--- | :--- |
| **Rango de Precio (COP)** | $50,000,000 - $120,000,000 COP | $18,000,000 - $28,000,000 COP | $600,000 - $1,200,000 COP | 270,400 COP |
| **Sistema de Alimentación** | Fuentes conmutadas médicas redundantes e inmunizadas | Fuente conmutada regulada de grado médico | Adaptador o fuente conmutada comercial | Fuente artesanal (Transformador + Puente + LM78XX) |
| **Control Térmico** | Calefactor cerámico/blindado (200W - 400W) con flujo controlado | Resistencia de alta potencia (100W - 200W) | Calefactor PTC o bombillo halógeno (50W - 100W) | Resistencia de alambre cerámico (20W) |
| **Monitoreo de Peso** | Balanza integrada de precisión clínica ($\pm 1\text{ g}$) | Balanza de alta resolución externa opcional | Celda de carga 1 kg - 5 kg con HX711 | Celda de carga 5 kg con HX711 en base de madera |
| **Estructura / Materiales** | Doble pared de acrílico grado médico termoformado | Acrílico de alta densidad reforzado | Acrílico mecanizado o impresión 3D completa | Estructura ligera (Cartón, foami, acrílico moldeable y madera) |
| **Grado de Homologación** | Certificación ISO 13485 / FDA / Registro Sanitario | Certificación IEC 60601 / Registro Sanitario | Sin certificación (Prototipo académico) | Sin certificación (Prototipo académico) |
---

# V. ANÁLISIS DE RESULTADOS

El desarrollo práctico en el laboratorio permitió comprobar de manera directa cómo responden los componentes electrónicos elegidos frente a las condiciones reales de prueba, alejándose de los comportamientos teóricos o simulados. Al integrar el sistema sobre la tarjeta ESP32, el proceso de evaluación entregó resultados contrastantes entre lo que se pudo medir en físico y lo que quedó limitado desde la selección del hardware.

En primera instancia, la prueba del sistema de pesaje mediante la celda de carga de 5 kg y el amplificador HX711 fue el único procedimiento que se ejecutó físicamente en la mesa de trabajo. Al colocar masas fijas sobre la plataforma para verificar la lectura en la pantalla LCD, el resultado obtenido fue una variación constante y errática en los datos mostrados. En lugar de registrar un valor estático, la cifra en pantalla oscilaba en cada iteración del programa. El análisis de esta respuesta muestra que las microvariaciones de voltaje que entrega la celda de carga se vieron severamente alteradas por la fragilidad de las conexiones en la protoboard y el uso de cables jumper. Estos elementos introdujeron variaciones en la resistencia de contacto y sirvieron como antenas para el ruido electromagnético de la red eléctrica, superando la capacidad del filtro por software y provocando la inestabilidad en la conversión digital.

Por otro lado, la evaluación del sistema de calentamiento no llegó a la fase de prueba experimental dentro de la cabina de acrílico. Al revisar la capacidad de la resistencia de alambre de 10 Ω y 20 W frente a las dimensiones reales del recinto (90 cm x 40 cm x 45 cm), se determinó que la potencia resultaba excesivamente baja para el volumen de aire que se pretendía calentar. La tasa de transferencia térmica de un elemento de apenas 20 W es incapaz de vencer la inercia térmica del aire y las pérdidas de calor que ocurren de forma natural a través del acrílico hacia el ambiente. Al constatar que la energía disipada no produciría un incremento medible en la temperatura interna, el proceso no se llevó a la ejecución física, quedando el microclima invariable en la temperatura ambiental del laboratorio y dejando el algoritmo de control térmico sin posibilidad de actuar sobre el proceso.

En conjunto, los resultados obtenidos en la práctica reflejan que la integración de un prototipo funcional depende directamente de la calidad de la instrumentación y del dimensionamiento físico de las etapas de potencia. Mientras que la etapa de lectura analógica se vio afectada por la falta de un cableado firme y blindado frente al ruido, la etapa térmica quedó condicionada por la restricción de potencia de la resistencia seleccionada, lo que impidió la validación completa del sistema en condiciones de operación real.

---

# VI. DISCUSIÓN DE PREGUNTAS

---

### 1. ¿Qué otras variables (y por qué) además de las aquí mencionadas son críticas en el monitoreo neonatal?

Además de la temperatura corporal y el peso corporal, un microclima y entorno de soporte neonatal de grado clínico exige la regulación y supervisión continua de las siguientes variables biológicas y ambientales:

* **Humedad Relativa del Aire (40% - 80%):** Es una variable ambiental crítica en neonatos prematuros extremo (menos de 32 semanas). Debido a la inmadurez de la capa córnea de la piel, los recién nacidos sufren pérdidas de agua transepidérmicas elevadas, lo que genera deshidratación rápida y pérdida de calor por evaporación. Mantener un nivel alto de humedad dentro de la cabina previene la hipotermia y las lesiones cutáneas.
* **Concentración de Oxígeno ($FiO_2$):** La oxigenoterapia en la cabina debe controlarse con precisión entre el 21% y el 40%. Niveles deficientes de oxígeno provocan hipoxia y daño cerebral, mientras que niveles excesivos generan toxicidad por radicales libres, desencadenando retinopatía de la prematuridad (ROP) o displasia broncopulmonar.
* **Frecuencia Cardíaca y Respiratoria (Pletismografía / ECG):** Permiten la detección inmediata de apneas de la prematuridad, bradicardias o paros cardiorrespiratorios. Dado que la regulación autonómica del recién nacido es inestable, estos parámetros son los primeros indicadores de distress respiratorio o sepsis.
* **Saturación de Oxígeno en Sangre ($SpO_2$):** Medida mediante pulsioximetría no invasiva, valida la efectividad del intercambio gaseoso y la perfusión periférica sin generar trauma en la piel.
* **Nivel de Ruido Ambiental (inferior a 45 dB en el interior):** El sistema auditivo y el desarrollo neurológico del neonato son altamente sensibles. Niveles elevados de ruido dentro de la cabina (causados por ventiladores, alarmas bruscas o fricción mecánica) generan estrés térmico, aumentan la frecuencia cardíaca y pueden alterar el desarrollo del SNC.

---

### 2. ¿Qué haría falta para convertir el sistema desarrollado en una incubadora neonatal real?

Para escalar el prototipo actual hacia un equipo biomédico utilizable en entornos clínicos reales, es necesario rediseñar el hardware, la estructura y el control bajo las normativas de dispositivos médicos (como la IEC 60601-2-21):

* **Rediseño de la Etapa de Potencia y Calentamiento:** Reemplazar la resistencia de 20 W por un elemento calefactor de tipo ciego o aletado de alta capacidad (150 W a 300 W) alimentado por un circuito conmutado sólido, sumado a un flujo de aire forzado controlado mediante ductos para garantizar una distribución uniforme del calor.
* **Inmunidad Electromagnética y Diseño de PCB:** Reemplazar el montaje en protoboard y los cables jumper por una tarjeta de circuito impreso (PCB) multicapa con planos de masa independientes para la etapa de potencia y la de señal. Se requiere aislamiento galvánico (optoacopladores e aisladores digitales) entre la ESP32 y las etapas de lectura analógica para eliminar el ruido de la red eléctrica en el amplificador HX711.
* **Sistemas de Seguridad Redundantes por Hardware:** Una incubadora médica no puede depender de un único microcontrolador. Se exige un circuito de corte térmico analógico independiente (termostato físico o bimetálico) que desconecte mecánicamente la alimentación del calefactor si la temperatura supera los 38 °C, previniendo hipertermia por fallas de firmware.
* **Materiales de Grado Médico y Chasis Aislante:** Sustituir la estructura de cartón, foami y madera por un chasis de policarbonato o acrílico de doble pared termoformado con sellado hermético, resistente a procesos de desinfección química agresiva y capaz de mantener el aislamiento térmico.
* **Batería de Respaldo y Alarmas Auditivas/Visuales:** Incorporación de un sistema de Alimentación Ininterrumpida (UPS) interno para mantener el control de temperatura y los monitores activos durante fallas en el suministro eléctrico hospitalario, junto con un protocolo de alarmas priorizadas según norma IEC 60601-1-8.

---

### 3. ¿Qué semejanzas hay entre una incubadora neonatal y una servo-cuna?

Aunque la incubadora neonatal (recinto cerrado) y la servo-cuna o cuna radiante (sistema abierto) difieren en su geometría y mecanismo primario de transferencia de calor, comparten principios fundamentales de diseño biomédico:

* **Principio de Servo-Control en Bucle Cerrado:** Ambos sistemas utilizan un algoritmo de control automático que ajusta la potencia del elemento calefactor en tiempo real, basándose en la retroalimentación continua de la temperatura medida por un sensor pegado a la piel del neonato (Modo Servocontrolado o Modo Piel).
* **Objetivo de Ambiente Térmico Neutro (ATN):** Ambos equipos buscan mantener al recién nacido en un rango de temperatura donde el gasto metabólico y el consumo de oxígeno sean mínimos, previniendo tanto el estrés por frío como el estrés por calor.
* **Monitoreo Integrado y Seguridad:** Comparten la integración de módulos de sensorización (temperatura, peso, constantes vitales) y el uso de sistemas de alarma redundantes frente a desviaciones de temperatura respecto a la consigna fijada por el personal médico.
* **Uso en Cuidado Intensivo Neonatal (UCIN):** Ambas son plataformas diseñadas para dar soporte vital al neonato, permitiendo el acceso del personal de enfermería y pediatría para procedimientos médicos, toma de muestras o administración de tratamientos.

---

# VII. CONCLUSIONES

El desarrollo de sistemas de soporte neonatal mediante plataformas de hardware de bajo costo responde a la necesidad de evaluar alternativas accesibles para el monitoreo de constantes biológicas en entornos de recursos limitados. A través de este proyecto se buscó integrar la arquitectura de control de la ESP32 con sensores térmicos, interfaz de usuario y una etapa de pesaje digital basada en la celda de carga de 5 kg y el amplificador HX711. La evaluación experimental demostró que la viabilidad del prototipo depende críticamente de la relación entre el dimensionamiento de potencia y la integridad de las señales analógicas. Por un lado, la etapa de calentamiento no logró alterar el microclima de la cámara debido a la insuficiencia térmica de la resistencia de 20 W frente al volumen del recinto, impidiendo la validación práctica del bucle de control. Por otro lado, la etapa de pesaje evidenció una dispersión constante en la lectura de masa, atribuible a la susceptibilidad del puente de Wheatstone ante falsos contactos en la protoboard y al ruido inducido por la red eléctrica en la fuente de alimentación discreta. Los hallazgos de este trabajo delimitan con claridad la frontera entre la lógica de control por software y los requerimientos físicos del hardware biomédico. La experiencia práctica demuestra que la programación del microcontrolador resulta ineficaz si no está respaldada por un acondicionamiento de señal acoplado, blindaje electromagnético adecuado y un cálculo riguroso de transferencia de calor en la etapa de potencia. El siguiente paso para la evolución de esta plataforma consiste en reemplazar el montaje en protoboard por un circuito impreso (PCB) diseñado a medida con planos de masa aislados, incorporar optoacopladores para la etapa de potencia y redimensionar el elemento calefactor con un sistema de ventilación forzada que garantice la circulación uniforme de temperatura dentro de la cabina.

---


# REFERENCIAS

* [1] C. G. K. Tran, A. Gibson, D. Wong, D. Tilahun, N. Selock, T. Good, and G. Rao, "Designing a low-cost multifunctional infant incubator," *J. Lab. Autom.*, vol. 19, no. 3, pp. 332–337, Jun. 2014. doi: 10.1177/2211068214530391[cite: 10].
* [2] L. Restrepo-Pérez, N. Durango-Londoño, N. E. Gómez-Suárez, F. González-Ramírez, y N. Rivera-Bonilla, "Prototipo de incubadora neonatal," *Revista Ingeniería Biomédica*, vol. 1, no. 1, pp. 55–59, 2007[cite: 10].
* [3] M. van Leeuwen *et al.*, *European Standards of Care for Newborn Health: Temperature management in newborn infants*, European Foundation for the Care of Newborn Infants (EFCNI), Nov. 2018[cite: 10].
* [4] R. B. Knobel-Dail, "Role of effective thermoregulation in premature neonates," *Research and Reports in Neonatology*, vol. 4, pp. 147–156, 2014. doi: 10.2147/RRN.S52377[cite: 10].
* [5] R. E. Black *et al.*, "Global, regional, and national causes of child mortality in 2008: a systematic analysis," *Lancet*, vol. 375, pp. 1969–1987, Jun. 2010. doi: 10.1016/S0140-6736(10)60549-1[cite: 10].
