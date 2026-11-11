# Apuntes semanas 3 y 4 Sistemas digitales

---

## Lo de los timers

Un timer es básicamente un **contador que cuenta pulsos de reloj solo**, sin que la CPU haga nada. Y es hardware de verdad, un circuito dentro del chip, no un programa. La CPU solo lo configura, lo prende y lo apaga. En medio, cuenta él solo.

Por eso es mejor que `delay()` o `millis()`:

```
delay(1000)  -> la CPU se queda parada esperando
millis()     -> la CPU pregunta "¿ya pasó?" en cada vuelta del loop
timer        -> otro circuito cuenta y AVISA cuando toca
```

Con `millis()` si el loop está ocupado 40 ms, el aviso llega hasta con 40 ms de retraso. Con el timer no.

### Cómo pasa de "contar" a "tiempo"

El reloj que le llega al timer es de 80 MHz = 80 millones de pulsos por segundo. Si el contador sumara con cada pulso, para medir 1 segundo tendría que llegar a 80 000 000, mucho.

Entonces se pone un **prescaler**, que es un divisor: deja pasar solo 1 de cada N pulsos.

```
reloj 80 000 000 pulsos/s  --> [ ÷ 80 ] --> 1 000 000 ticks/s --> [ contador ]
                                  prescaler      (1 tick = 1 µs)
```

Con N = 80 cada tick dura 1 µs, o sea el contador "cuenta en microsegundos". Eso es lo que usamos en clase.

Lo que había que entender:
- **resolución** = lo que dura un tick = 1 / f (lo más chiquito que el timer distingue)
- **alarma** = el número al que tiene que llegar el contador para avisar

La imagen de la escalera: cada peldaño mide lo que la resolución, y la alarma es cuántos peldaños subes.

```
tiempo = alarma × resolución
alarma = tiempo × f
```

**Ejemplo:** quiero un aviso cada 0,5 s con ticks de 1 µs.
alarma = 0,5 × 1 000 000 = **500 000**. Comprobación: 500 000 × 1 µs = 0,5 s ✔️

Otro: f = 10 000 y alarma = 2500 → resolución 100 µs, tiempo = 2500 × 100 µs = 0,25 s.

Los de la actividad: A) 2000 µs = 2 ms · B) 10 ms → alarma 10 000 · C) 40 000 ticks a 40 kHz = 1 s, y el prescaler sería 80 000 000 / 40 000 = 2000.

### Autorrecarga vs desbordamiento (se confunden fácil)

```
autorrecarga:   0 1 2 3 ... [ALARMA] -> aviso -> vuelve a 0 -> repite
                              (yo elijo este número)

desbordamiento: 0 1 2 3 ... [MÁXIMO DE BITS] -> da la vuelta a 0
                              (límite físico, casi nunca se quiere)
```

La analogía de la pista de atletismo: la alarma es una línea que pinto yo en el metro 400. El desbordamiento es el final de la pista.

Dato curioso: el contador del ESP32 es de 64 bits, así que se llenaría en unos 584 000 años (con ticks de 1 µs). O sea, nunca. En cambio el de 8 bits del AVR se llena en 0,26 ms. Por eso en el AVR el prescaler es vital y en el ESP32 solo sirve para elegir una unidad cómoda.

El ESP32 tiene 4 timers (2 grupos de 2), cada uno de 64 bits, y trabajan en paralelo.

### API, core, versión 3 (cosas de software)

- **API** = el menú de un restaurante. Yo pido una función (`pinMode`, `delay`...) sin saber cómo se cocina por dentro. La API del timer calcula prescaler y alarma y escribe los registros por mí.
- **Core** = el paquete que le enseña al Arduino IDE a programar una placa. Por defecto solo sabe de Arduino Uno/Mega; para el ESP32 hay que instalar "esp32 by Espressif Systems".
- ⚠️ En la **versión 3.x** del core cambiaron las funciones de los timers. Casi todos los tutoriales de internet son de la versión 2 y **no compilan**. No copiar a ciegas.

### Cómo se entera la CPU: interrupción

```
loop() ejecutando línea 12
        |
   ¡suena la alarma!  --> la CPU PAUSA, corre la ISR
        |
loop() sigue en la línea 13 como si nada
```

- **Interrupción** = señal que dice "detente, pasó algo".
- **ISR** = la función que atiende ese aviso.
- **Callback** = una función que yo escribo pero no llamo yo; se la doy al timer y él la llama. (Como dejar tu número en el taller: "llámeme cuando esté listo".)

Y tres reglas que valen para todas las ISR:
- `IRAM_ATTR` delante del nombre → que la ISR viva en la memoria rápida
- `volatile` en las variables que comparten la ISR y el loop
- patrón bandera: la ISR levanta una banderita y se va, el loop hace el trabajo lento

(Estas tres las repetimos con más detalle en la semana 4.)

### Repaso rápido de la semana 2 que salió ahí

- GPIO18 es el *nombre de una señal*, no "el pin 18 de la placa". Hay que mirar el rótulo impreso.
- GPIO 34–39 solo son entrada. Un LED ahí nunca prende aunque compile bien.
- variable local = vaso desechable; `static` = taza en tu escritorio.
- `uint8_t` 200 + 100 da 44 (se calcula como int = 300 y se recorta a 8 bits: 300 − 256).

---

## Lo de las interrupciones externas

### El problema que resuelven

Sondeo = mirar por la ventana cada 5 min a ver si llegó el paquete.

```
loop():  lee pin ... [ocupado con delay(2000)] ... lee pin
botón:        ___/‾‾\___   <- el toque ocurrió aquí en medio
```

Cuando el loop vuelve a leer, el botón ya está suelto: **el evento se perdió**. En un juego de luces da igual, en una alarma de puerta no.

La interrupción es instalar un timbre: suena solo, vas, abres, vuelves a lo tuyo.

Y lo importante: quien vigila el pin no es mi código, es el **periférico GPIO** (un circuito). Por eso el aviso llega aunque el loop esté trabado en un delay.

```
botón -> pin GPIO21 baja de 1 a 0 -> periférico GPIO lo detecta -> CPU pausa el loop -> corre ISR
```

Es el mismo mecanismo del timer. Lo único que cambia es quién toca el timbre: antes el timer (por dentro, cada cierto tiempo), ahora algo de afuera (cuando pasa algo, sin avisar).

### Nivel vs flanco

- **Nivel** = "la puerta está abierta" (dura)
- **Flanco** = "la puerta se acaba de abrir" (un instante)

Con `INPUT_PULLUP` todo se invierte: suelto = 1, presionado = 0.

```
1  ‾‾‾‾‾‾‾\          /‾‾‾‾‾‾‾
0          \________/
  suelto   presionado   suelto
        FALLING     RISING
```

- FALLING (1→0) = al **presionar**
- RISING (0→1) = al **soltar**
- CHANGE = los dos, la ISR corre dos veces por cada toque

Para la alarma con pull-up: **FALLING**. (Truco: con pull-up, presionar hace *caer* la señal.)

### Qué pines sirven

La mayoría del ESP32 sirve (en el Uno solo D2 y D3). Para no equivocarme:

- los buenos: 16, 17, 18, 19, 21, 22, 23, 25, 26, 27, 32, 33
- 34, 35, 36, 39: sirven, pero **no tienen pull-up interna**, hay que ponerle una resistencia de 10 kΩ por fuera a 3,3 V. Si no, el pin queda flotando y capta ruido → alarmas fantasma.
- 0, 2, 5, 12, 15 (strapping): evitar. Si están presionados al encender, el chip arranca mal.
- 6 a 11: ni tocarlos, los usa la flash.

### attachInterrupt

```cpp
pinMode(21, INPUT_PULLUP);                                       // 1.º
attachInterrupt(digitalPinToInterrupt(21), isrBoton, FALLING);   // 2.º
```

Se lee: "cuando el pin 21 caiga de 1 a 0, llama a isrBoton". Va una sola vez en `setup()` (el timbre se instala una vez, no cada vez que llega alguien). Y `isrBoton` va **sin paréntesis**: le doy el nombre, no la estoy llamando.

### La ISR: tres reglas y por qué

**1) Corta.** Como el extintor: apagas el fuego y te vas, no limpias la cocina.

```
SÍ:  subir bandera, guardar millis(), sumar 1, digitalWrite
NO:  Serial.print, delay, cálculos largos, esperar algo
```

Si es larga, el **watchdog** (perro guardián) cree que el chip se trabó y lo reinicia → "Guru Meditation Error". Síntoma típico: "el ESP32 se reinicia de vez en cuando".

**2) `IRAM_ATTR`.** Esto me costó entenderlo. El programa vive en la flash (grande pero lenta), y la CPU la lee con ayuda de una **caché** (una copia rápida de lo que leyó hace poco; como tener en tu mesa los libros que usaste hace rato). El problema:

```
el ESP32 escribe en la flash -> la caché se apaga un momento
-> llega una interrupción -> la ISR estaba en la flash -> la CPU no puede leerla -> se cae
```

Es raro y casi imposible de reproducir ("a veces falla y no sé por qué"). Solución: `IRAM_ATTR` le pide al compilador que copie la ISR a la **IRAM**, la zona de la SRAM para instrucciones, siempre disponible. La bodega a veces está cerrada; la pared de la cocina no.

(La SRAM del chip se divide en IRAM = instrucciones y DRAM = datos. Ojo: aquí DRAM es "Data RAM", no la DRAM de un computador.)

**3) `volatile`.** La más traicionera. El compilador es ahorrativo: si ve

```cpp
while (alerta == false) { }
```

piensa "nadie cambia `alerta` aquí, la copio una vez a mi cuaderno (registro de la CPU) y no vuelvo al pizarrón (RAM)". Y como **no sabe que existe la ISR** (nadie la llama desde el código, la dispara el hardware), cree que nunca cambia.

La analogía de la fecha del examen: copias "viernes" del pizarrón, el profe lo cambia a "lunes" (la ISR), tú miras tu cuaderno y sigue diciendo "viernes".

```
momento                 RAM (pizarrón)   copia en CPU (cuaderno)
arranca el loop              false              false
la ISR escribe true          true               false
siguen las vueltas           true               false  <- para siempre
```

El LED nunca prende y el código "se ve bien", ni un error de compilación. `volatile` dice: "no confíes en tu copia, ve a la RAM cada vez". Solo eso; no la hace más rápida ni más segura.

**¿Cuándo va?** Cuando la tocan **la ISR y el loop** los dos. Si solo la toca uno, no.
- `pulsos` (ISR suma, loop lee) → volatile
- `tUltimo` (solo la ISR) → no
- `total` (solo el loop) → no

Ojo, es traicionero porque a veces funciona sin él (por ejemplo con un `Serial.println` en el loop el compilador vuelve a leer la RAM) y falla cuando cambias algo. Si cumple la regla, va `volatile`, aunque "ya me funcionó".

### Patrón bandera

El buzón rural: el cartero (ISR) deja la carta, levanta la banderita y se va en 5 s. Yo (loop) veo la banderita, la bajo y leo la carta con calma. El cartero no entra a leérmela en voz alta.

```cpp
volatile bool alerta = false;

void IRAM_ATTR isrBoton() { alerta = true; }   // rápida

void loop() {
  if (alerta) {
    alerta = false;                  // primero bajar
    Serial.println("¡Alerta!");      // después el trabajo pesado
  }
}
```

El aviso puede atenderse un poquito tarde, pero **no se pierde**.

Limitación: la bandera **no cuenta**. 3 toques con el loop ocupado = 1 aviso. Si quiero saber cuántas veces, uso un contador (`volatile uint32_t pulsos`, la ISR hace `pulsos++`).

### Rebote

Un botón mecánico no hace un solo flanco limpio, hace varios en pocos ms:

```
‾‾‾\_/‾\_/‾\______   <- 3 flancos de bajada = la ISR corre 3 veces por 1 toque
```

Con sondeo (semana 2) el loop lo filtraba. Con interrupción cada rebote dispara la ISR, así que un toque cuenta como 3 o 5.

Solución: dentro de la ISR, ignorar flancos a menos de 50 ms del último que acepté. Es el mismo patrón `ahora - tAnterior` de la semana 2:

```cpp
uint32_t ahora = millis();
if (ahora - tUltimo >= 50) {   // ¿toque nuevo?
  alerta = true;
  tUltimo = ahora;
}                              // si no, es rebote
```

### Todo junto

```
Botón (FALLING) -> periférico GPIO detecta -> ISR [IRAM_ATTR + antirrebote]
                -> bandera [volatile] -> loop() hace el trabajo pesado
```

Preguntas:
¿Qué falla si olvido `volatile`? 
→ el loop nunca ve la bandera. ¿Y si olvido `IRAM_ATTR`? → funciona casi siempre y falla de vez en cuando, sin razón aparente.

Y el ejemplo final con `delay(2000)` en el loop está hecho a propósito: el loop está ocupado y aun así el aviso no se pierde. Eso es lo que no lograba el sondeo.

---

