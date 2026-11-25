# bda-ut1-pentaho-hotel · Cierre de cobros por hotel y canal

**Autor:** Asier Gutiérrez Ortiz  
**Código personal asignado:** A013  
**Conjunto de datos:** `cierre_aula_01`  
**Herramienta:** Pentaho Data Integration (PDI), ejecutado con Kitchen

---

## 1. Objetivo del encargo y recorrido de los datos

Dirección pide conocer **el importe cobrado y el número de reservas cobradas por hotel y canal**, recibir un CSV claro del cierre y saber **qué reservas todavía no tienen un cobro asociado**. El procedimiento debe poder repetirse y avisar si no puede generar el informe.

Se entregan dos CSV:

| Archivo | Para quién | Qué contiene |
|---|---|---|
| `salida/informe_hotel_canal.csv` | Dirección | Una fila por hotel y canal: `hotel;canal;n_reservas;cobrado_total` (euros) |
| `salida/reservas_sin_cobro.csv` | Control interno | Reservas sin cobro asociado: `id_reserva`, `hotel`, `canal`, `importe`, `motivo` |

### Párrafo para gerencia

El cierre `cierre_aula_01` es un conjunto cerrado de 20 reservas, 15 cobros y 5 hoteles del grupo, ya seleccionado para esta práctica. De las 20 reservas, **15 tienen un cobro asociado y suman 3965,50 €**, repartidos en 13 combinaciones de hotel y canal. Las otras **5 reservas (identificadores 4, 9, 11, 15 y 18) no tienen ningún cobro asociado en los archivos recibidos**: conviene revisarlas, pero no demuestran por sí solas un impago, porque pueden tener un pago pendiente o un plazo de pago que estos archivos no describen. En el listado de revisión, el campo `importe` es el precio de la reserva, no dinero cobrado, y no se suma al informe principal. Queda fuera del informe cualquier cálculo de ocupación.

### Límites de los datos

- Los archivos **no tienen fechas**, así que no se puede filtrar «ayer» ni analizar por periodo. Se asume que los tres pertenecen a un mismo cierre ya seleccionado.
- La **fecha y hora de generación** que figura más abajo es la de la ejecución del proceso; **no es la fecha de cobro**.
- **No se calcula la ocupación**. Para responder a esa pregunta pediría, por hotel y periodo: habitaciones disponibles, habitaciones ocupadas (o noches-habitación vendidas), fechas de entrada y salida de cada reserva y estado de la reserva (confirmada, cancelada, no presentado).
- Se asume como máximo un cobro por reserva (así ocurre en estos archivos). Los importes están en **euros**.

### Recorrido de los datos

```text
reservas_pdi.csv + cobros_pdi.csv + hoteles_pdi.csv        (entrada/, originales sin modificar)
                         ↓
        Lectura y comprobación de tipos y claves
                         ↓
   informe_cobros.ktr          reservas_sin_cobro.ktr       → temporal/ (provisionales, no se suben)
                         ↓
   validar_informe.ktr         validar_sin_cobro.ktr        (si fallan: Abort, no se publica nada)
                         ↓
   Copia de los dos CSV a salida/ y registro del resultado  (éxito solo tras las dos copias)
```

---

## 2. Entorno y ejecución

### Estructura del repositorio

```text
bda-ut1-pentaho-hotel/
├── README.md
├── .gitignore            # contiene /temporal/
├── asignacion.json       # código del paquete (A013) y huellas SHA-256 de los originales
├── entrada/              # los tres CSV originales
├── procesos/             # informe_cobros.ktr, reservas_sin_cobro.ktr,
│                         # validar_informe.ktr, validar_sin_cobro.ktr, cierre_hotel.kjb
├── temporal/             # resultados provisionales (no se suben a Git)
├── salida/               # los dos CSV finales comprobados
└── evidencias/           # comprobaciones y pruebas comentadas
```

### Pasos para ejecutarlo desde una copia recién descargada

1. Clonar o descargar el repositorio y entrar en su carpeta.
2. **Crear las carpetas `temporal` y `salida`** (Git no conserva carpetas vacías y `temporal` está en `.gitignore`).
3. Comprobar que los tres CSV están en `entrada/` y que `asignacion.json` indica `A013`.
4. Ejecutar el job con Kitchen (adaptar la ruta a cada equipo):

   Windows:
   ```bat
   kitchen.bat /file="C:/ruta/bda-ut1-pentaho-hotel/procesos/cierre_hotel.kjb" /level:Basic
   ```
   Linux:
   ```bash
   ./kitchen.sh -file="/ruta/bda-ut1-pentaho-hotel/procesos/cierre_hotel.kjb" -level=Basic
   ```
5. Revisar el final del log: debe aparecer el registro de éxito y el job debe terminar sin errores.

Para probar una transformación por separado se usa Pan, por ejemplo:

```bat
pan.bat /file="C:/ruta/bda-ut1-pentaho-hotel/procesos/informe_cobros.ktr" /level:Basic
```

**Dónde aparecen los resultados**

- Provisionales: `temporal/informe_hotel_canal.csv` y `temporal/reservas_sin_cobro.csv`.
- Finales, tras validar y copiar: `salida/informe_hotel_canal.csv` y `salida/reservas_sin_cobro.csv`.

**Programación.** Para la entrega, Kitchen se ejecuta manualmente. En un entorno real, el programador de tareas del sistema (Programador de tareas de Windows o `cron`) lanzaría ese mismo comando de madrugada. Ejecutar el comando no deja nada programado por sí mismo.

---

## 3. Datos de entrada y generación

**Código personal:** A013. **Conjunto:** `cierre_aula_01`.

**Procedencia.** Los tres CSV proceden del paquete personal A013 entregado por el profesor. Se conservan sin modificar en `entrada/`, y `asignacion.json` guarda sus huellas SHA-256 para comprobarlo. Las pruebas de fallo se hacen sobre copias de trabajo, nunca sobre los originales.

| Archivo | Separador | Filas de datos | Integer | String | BigNumber |
|---|---|---|---|---|---|
| `reservas_pdi.csv` | coma (`,`) | 20 | `id_reserva`, `id_hotel`, `noches` | `canal` | `importe` |
| `cobros_pdi.csv` | punto y coma (`;`) | 15 | `id_reserva` | `medio` | `cobrado` |
| `hoteles_pdi.csv` | punto y coma (`;`) | 5 | `id_hotel`, `id_cadena` | `hotel`, `comarca` | — |

Todos se leen con cabecera y codificación UTF-8. En los importes el símbolo decimal es el punto y el separador de miles queda vacío; se usa BigNumber para no perder precisión en dinero. `comarca`, `id_cadena` y `medio` no se usan en los informes y se conservan en los originales.

**Formato de salida.** Ambos CSV llevan cabecera, UTF-8, importes con punto decimal y dos decimales sin separador de miles, y no se añaden filas a ejecuciones anteriores (el archivo se sobrescribe).

**Fecha y hora de generación de los resultados:** 25 de noviembre de 2026, 23:22 (ejecución final de Kitchen). Es la fecha de generación del cierre en el aula, **no** la fecha de cobro.

### Resultados esperados (cierre_aula_01)

| Comprobación | Esperado |
|---|---|
| Reservas / cobros / hoteles leídos | 20 / 15 / 5 |
| Filas tras unir reservas y cobros | 15 |
| Filas tras añadir hoteles | 15 |
| Grupos del informe (sin cabecera) | 13 |
| Suma de `n_reservas` | 15 |
| Suma de `cobrado_total` | 3965.50 € |
| Laredo, canal web | 2 reservas; 496.50 € |
| Reservas sin cobro | 5: 4, 9, 11, 15 y 18 |
| Reconciliación | 15 cobradas + 5 sin cobro = 20 |

Los 13 grupos son: Comillas (ota, recepcion, web), Laredo (ota, web), Noja (ota, web), Potes (ota, recepcion, web) y Santander (ota, recepcion, web).

---

## 4. Decisiones de diseño

- **Unión reservas–cobros por `id_reserva` con INNER JOIN.** El informe principal solo debe incluir reservas con cobro asociado; el INNER descarta las que no tienen cobro y deja 15 filas. Se ordenan ambas entradas por la clave porque Merge join lo exige, y se revisan expresamente entrada izquierda (reservas), derecha (cobros) y claves.
- **Unión con hoteles por `id_hotel`, también INNER.** Añade el nombre del hotel. Deben mantenerse las 15 filas; si faltara una correspondencia, no se continúa sin investigar (en estos datos todos los hoteles existen).
- **Sumar `cobrado`, no `importe`.** `importe` es el precio de la reserva; `cobrado` es el dinero realmente cobrado. Dirección pregunta por lo cobrado, y una reserva podría estar cobrada por un valor distinto de su precio.
- **`n_reservas` como recuento de `id_reserva`.** Es correcto porque hay como máximo un cobro por reserva. Si una reserva tuviera dos pagos, el cruce generaría dos filas: habría que sumar todos los cobros pero contar las reservas **distintas** (por ejemplo, agrupar antes los cobros por `id_reserva` o usar un recuento de valores distintos).
- **Detección de reservas sin cobro: LEFT OUTER JOIN y `id_reserva_cobro` nulo.** Se renombra la clave de cobros a `id_reserva_cobro` para distinguirla de la de la reserva. Tras el LEFT OUTER se conservan las 20 reservas y se filtran las que tienen esa clave nula. **No se comprueba `cobrado = 0`**: un cobro de cero y un cobro ausente son situaciones distintas.
- **Listado sin cobro separado.** No se suma al informe ni se descarta: es un control interno. No se inventan importes; `importe` conserva su nombre y es el precio de la reserva. El campo `motivo` explica por qué aparece cada fila («Sin cobro asociado en el archivo recibido»). No se incluyen nombres de huéspedes ni otros datos personales.
- **Validación antes de publicar.** Dos transformaciones leen los CSV provisionales completos. Un resumen sin claves con *Always give back a result row* produce siempre una fila, de modo que un CSV con solo cabecera (recuento 0, sumas nulas) también falla. Abort se configura con umbral 0 y error. Los valores esperados (13 grupos, 15 reservas, 3965.50 €, identificadores 4, 9, 11, 15 y 18) son controles de este conjunto; en un cierre real serían controles de reconciliación con las entradas, no totales fijos.
- **Job con publicación ordenada.** Se comprueban las tres entradas (falta alguna → Abort job indicando cuál), se generan los dos CSV, se comprueba que existen los provisionales, se validan y solo entonces se copian a `salida/` sustituyendo versiones anteriores. No se borra el último informe publicado al empezar, así que un fallo de preparación o validación lo deja intacto.
- **Publicación no atómica.** Copiar dos archivos no es atómico: si la primera copia termina y la segunda falla, `salida/` puede contener un archivo nuevo y otro anterior o ausente. El job termina con error, avisa de ese posible estado parcial y no registra éxito. No se deshacen las copias. Como mejora se podría publicar por versiones y señalar la nueva solo cuando esté completa.
- **Repetición.** Las salidas se sobrescriben (sin *append*), así que repetir con las mismas entradas da los mismos grupos, filas y totales, sin duplicados.
- **Por qué CSV.** Es texto plano, legible por dirección con cualquier hoja de cálculo, ligero y portable, ideal para un resultado pequeño de entrega. Para un histórico grande consultado por columnas estudiaría un formato columnar como Parquet (compresión y lectura solo de las columnas necesarias).

---

## 5. Comprobaciones y pruebas

Las comprobaciones y las seis pruebas de éxito, repetición y fallo (primera ejecución, repetición, falta una entrada, salida incompleta, total incorrecto y recuperación) están explicadas y comentadas en el PDF [Pruebas de éxito, repetición y fallo](evidencias/Pruebas%20de%20%C3%A9xito%2C%20repetici%C3%B3n%20y%20fallo.pdf), dentro de la carpeta `evidencias/`.

| Prueba | Resultado esperado |
|---|---|
| Primera ejecución | Dos salidas correctas y estado final de éxito |
| Repetición | Mismos grupos, filas y totales; sin duplicados |
| Falta una entrada | Job con error; no se publica un informe nuevo |
| Salida incompleta | La validación falla aunque el CSV exista |
| Total incorrecto | La validación falla |
| Recuperación | Se generan, validan y publican de nuevo las salidas correctas |

Las pruebas de validación se ejecutan directamente sobre la transformación de control con una copia alterada del provisional (si se ejecutara antes el job completo, regeneraría el archivo y borraría el fallo). Los cambios se hacen solo en copias de trabajo y se restauran después.
