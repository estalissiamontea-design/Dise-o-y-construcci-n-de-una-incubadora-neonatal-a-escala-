# DISEÑO, CONSTRUCCIÓN Y EVALUACIÓN EXPERIMENTAL DE UN PROTOTIPO DE INCUBADORA NEONATAL A ESCALA CON CONTROL TÉRMICO PID Y SUBSISTEMA DE PESAJE DIGITAL

**Autores:** Estudiantes de Ingeniería Biomédica  
**Filiación:** Facultad de Ingeniería, Universidad Militar Nueva Granada, VII Semestre  
**Asignatura:** Instrumentación Biomédica y Biosensores  

---

## RESUMEN
Se presenta la implementación técnica y el análisis experimental de un prototipo de incubadora neonatal a escala supervisado por un microcontrolador ESP32. El desarrollo integra un lazo de control térmico en lazo cerrado tipo Proporcional-Integral-Derivativo (PID) orientado a mantener una temperatura objetivo de 36.5 °C, empleando un termistor NTC de 10 kΩ como elemento censor y una etapa de potencia activada por Modulación por Ancho de Pulso (PWM) invertida para la conmutación de un elemento calefactor resistivo de 10 Ω / 20 W. En paralelo, el prototipo incorpora un sistema de monitoreo de peso continuo mediante una celda de carga de 2 kg acondicionada con el amplificador analógico-digital de 24 bits HX711, proyectando las lecturas en una pantalla LCD de 16x2 líneas sobre bus I2C. Durante la fase de pruebas de laboratorio, el sistema exhibió desviaciones en el seguimiento de la variable térmica y variaciones en la estabilidad de la masa medida. Se determinó que estas condiciones se derivaron de la respuesta no lineal del convertidor analógico-digital (ADC) interno del microprocesador, perturbaciones por desacoplamiento de masa común y ruido de conmutación en la etapa de potencia. En este documento se examina el comportamiento físico del sistema, la formulación matemática implementada, la causa de las desviaciones encontradas y la reestructuración requerida para la optimización del circuito.

**Palabras clave:** Incubadora neonatal, ESP32, Control PID, Termistor NTC, HX711, Instrumentación Biomédica.

---

# I. INTRODUCCIÓN

En la disciplina de la ingeniería biomédica, la incubadora neonatal constituye un equipo de soporte vital diseñado para crear un microambiente aislado que replica las condiciones termodinámicas del entorno uterino [5, 8]. Los recién nacidos prematuros o de bajo peso al nacer presentan una marcada inmadurez en su barrera epidérmica, escasez de tejido adiposo pardo y una respuesta vasomotora ineficiente ante estímulos térmicos, lo que incrementa su susceptibilidad a pérdidas aceleradas de calor por conducción, convección, radiación y evaporación [5, 10]. Esta fragilidad fisiológica hace imperativo mantener al neonato dentro de un Ambiente Térmico Neutro, situado de manera estricta entre $36.0\ ^\circ\text{C}$ y $37.5\ ^\circ\text{C}$ [8, 10], con el fin de minimizar el gasto metabólico basal y el consumo de oxígeno, previniendo episodios de hipotermia o hipertermia que incrementan la morbimortalidad en unidades de cuidados intensivos neonatales[cite: 5, 10]. En este contexto, el estudio y desarrollo de prototipos de monitoreo térmico y pesaje en tiempo real resulta esencial para comprender los principios de diseño y acondicionamiento en instrumentación médica de soporte vital [5, 8].

La regulación de la temperatura en este tipo de dispositivos se fundamenta en la transferencia de calor por convección forzada en lazo cerrado, utilizando elementos calefactores accionado por modulación de potencia y sensores de coeficiente de temperatura negativo (NTC)[5, 7], cuya respuesta no lineal se describe mediante el modelo Beta:

$$
R_{\text{NTC}}(T) = R_0 \cdot \exp \left( \beta \left( \frac{1}{T} - \frac{1}{T_0} \right) \right)
$$

A la par del control térmico, la evaluación continua de la masa corporal representa una variable clínica crítica para el seguimiento del crecimiento y el balance hídrico del infante sin someterlo al estrés mecánico de retirarlo de la cabina[cite: 5, 8]. Esto se logra mediante celdas de carga basadas en el principio de deformación en puente de Wheatstone, acondicionadas con etapas de amplificación y digitalización de alta resolución como el circuito integrado HX711 de 24 bits[cite: 8].

El presente proyecto tuvo como objetivo principal diseñar, ensamblar y evaluar operativamente un prototipo de incubadora neonatal a escala controlado por una tarjeta ESP32, integrando un algoritmo de control térmico PID y un subsistema de pesaje digital con salida a una pantalla LCD 16x2 I2C[cite: 5, 8]. De manera específica, el trabajo se enfocó en implementar el acondicionamiento analógico para el termistor NTC mediante un divisor de tensión[cite: 7], programar el bucle de control PID en tiempo discreto con salida PWM invertida para regular la potencia de una resistencia de $10\ \Omega / 20\text{W}$[cite: 5, 8], y calibrar la celda de carga para registrar mediciones de masa de hasta $2.00\text{ kg}$ mediante un factor de escala corregido ($5.405$)[cite: 8]. Se planteó como hipótesis que la arquitectura de hardware y software implementada en la ESP32 permitiría mantener la temperatura interna dentro de un margen de $\pm 0.5\ ^\circ\text{C}$ respecto al punto de consigna ($36.5\ ^\circ\text{C}$)[cite: 5, 8] y registrar la masa con una precisión aceptable.

La justificación técnica de esta práctica radica en el valor académico de confrontar la teoría matemática de control y la física de transferencia de calor con los condicionantes de una implementación electrónica real[cite: 5, 6]. El desarrollo experimental permitió identificar y diagnosticar limitantes físicas críticas, tales como la no linealidad del convertidor analógico-digital (ADC) interno de la ESP32, la inercia térmica del recinto y la inyección de ruido electromagnético a las líneas de lectura analógica por el desacoplamiento de masa común en la conmutación de potencia[cite: 8, 9]. 

El informe se organiza de la siguiente manera: la Sección II (Marco Teórico) profundiza en los mecanismos fisiológicos de termorregulación, la anatomía y subsistemas de la incubadora, la caracterización de los sensores, la formulación del PID y el principio del pesaje digital[cite: 5, 8]; la Sección III (Metodología y Desarrollo Experimental) detalla la asignación de hardware, la etapa de potencia y la estructura del firmware integrador[cite: 7, 8]; la Sección IV (Análisis de Resultados y Diagnóstico de Fallas) examina las causas raíz de la inestabilidad observada en laboratorio[cite: 9]; la Sección V (Análisis Comparativo de Costos) contrasta el presupuesto del prototipo frente a alternativas comerciales hospitalarias[cite: 8, 9]; y finalmente, las Secciones VI y VII presentan la respuesta a las preguntas de discusión y las conclusiones generales del estudio[cite: 9, 10].

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

# REFERENCIAS

* [1] C. G. K. Tran, A. Gibson, D. Wong, D. Tilahun, N. Selock, T. Good, and G. Rao, "Designing a low-cost multifunctional infant incubator," *J. Lab. Autom.*, vol. 19, no. 3, pp. 332–337, Jun. 2014. doi: 10.1177/2211068214530391[cite: 10].
* [2] L. Restrepo-Pérez, N. Durango-Londoño, N. E. Gómez-Suárez, F. González-Ramírez, y N. Rivera-Bonilla, "Prototipo de incubadora neonatal," *Revista Ingeniería Biomédica*, vol. 1, no. 1, pp. 55–59, 2007[cite: 10].
* [3] M. van Leeuwen *et al.*, *European Standards of Care for Newborn Health: Temperature management in newborn infants*, European Foundation for the Care of Newborn Infants (EFCNI), Nov. 2018[cite: 10].
* [4] R. B. Knobel-Dail, "Role of effective thermoregulation in premature neonates," *Research and Reports in Neonatology*, vol. 4, pp. 147–156, 2014. doi: 10.2147/RRN.S52377[cite: 10].
* [5] R. E. Black *et al.*, "Global, regional, and national causes of child mortality in 2008: a systematic analysis," *Lancet*, vol. 375, pp. 1969–1987, Jun. 2010. doi: 10.1016/S0140-6736(10)60549-1[cite: 10].
