# Informes por laboratorio

Informes entregados en cada laboratorio (Labs 1–5), tal como se entregaron.

**El [DDR](../DDR.md) es la versión consolidada.** Si un informe y el DDR no coinciden, vale el
DDR. Para consolidar se aplicó esta regla: manda el informe de cada laboratorio y, dentro de él,
la tabla de medición respaldada por captura de consola, por encima de resúmenes, conclusiones o
lo que repita el informe de otro laboratorio.

## Fe de erratas

| # | Informe | Dice | Valor consolidado en el DDR |
|---|---|---|---|
| 1 | Lab 2: resúmenes, ADR-002 y conclusiones | 3 saltos a 828–1145 ms; healing de 35–73 s; 189 ms post-healing a 60 m | Tablas 2–5: 1 salto 15,85 ms, 2 saltos 30,15 ms, corte de 8 s (sec. 71–78), 16,08 ms post-healing |
| 2 | Lab 2: tabla 6.5 | 2 saltos = 36,5 ms | 30,15 ms (Tabla 3) |
| 3 | Lab 5: tabla 7.4 (líneas base) | Lab 1 "20 m, 0 % PER"; Lab 2 "94 / 1145 ms, 73 s"; meta del OTBR "< 250 ms" | Lab 1: 10 m (FAIL); Lab 2: según el punto 1; OTBR: +109 ms frente a la meta de < 100 ms (FAIL) |
| 4 | Lab 3: sección 1 | Menciona ESP-IDF, FreeRTOS, `coap_demo.c` y libcoap | El firmware del equipo es Zephyr ([firmware/lab3_coap](../../firmware/lab3_coap)) |
| 5 | Lab 3: relación Observe | 187:1 | 300 GET / 7 notificaciones = 43:1 (−97,7 % de tráfico) |
| 6 | Lab 3: ADR-004 | ~35 µA, más de 3 años, latencia promedio de 2,5 s | Informe 4: ~155 µA, 238 días, 4,8 s en el peor caso |
| 7 | Labs 3 y 4: watchdog de la válvula | 15 min (Lab 3) / 60 s "incorporado" (Lab 4) | Diseñado, no implementado en el firmware de los Labs 3–4 |
| 8 | Lab 4: tabla 6.1 | Caracteriza DS18B20, DHT22, DHT11 y BMP280 | Solo se usó el DHT22 (el que entregó el profesor) |
