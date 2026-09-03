# Proyecto Relevo Financiero — Escalamiento sistemático de crédito rechazado

## Objetivo

Hoy, cuando un cliente sale **rechazado en la primera opción de financiamiento**, la
derivación a la segunda opción se hace por WhatsApp entre ejecutivos: si el ejecutivo
receptor no está disponible o el mensaje se traspapela, la oportunidad se pierde sin
quedar registrada en ningún lado. **Relevo Financiero** sistematiza ese traspaso: un
formulario simple que, al enviarse, notifica automáticamente al responsable correcto
(por sucursal y financiera) y deja un registro histórico de cada caso.

Origen: reunión 2026-08-21 entre Rodrigo Ruiz Zavala y Jaime Osvaldo Seguel Echeverria.
Alcance confirmado: **solo** el flujo de escalamiento de crédito rechazado — no incluye
renovaciones (tema aparte, tratado en `Proyectos/Renovaciones/`).

## Producto

Formulario + notificación + registro, sobre stack Microsoft 365:

- **Front:** Microsoft Forms (formulario que llena el ejecutivo).
- **Back:** dos listas de SharePoint — una maestra (`Maestro Responsables`) y una
  transaccional (`Casos Relevo Financiero`).
- **Automatización:** Power Automate dispara la notificación (Teams/Outlook) al
  responsable apenas se crea un caso nuevo.

**Estado actual:** mockup visual y funcional (validación de campos y lógica) publicado
como Artifact — [ver mockup](https://claude.ai/code/artifact/893455e8-0468-41be-829c-6a301ed84c4b).
Pendiente construir la versión productiva (ver [Estado y próximos pasos](#estado-y-próximos-pasos)).

## Datos de entrada (formulario)

| Campo | Tipo | Detalle |
|---|---|---|
| Sucursal | Selección | Catálogo `referencia/sucursales_usados_codapi.csv` (41 sucursales). Se muestra y registra como `Nombre (Código API)` para no perder el código. |
| Ejecutivo que reporta | Texto | — |
| Nombre del cliente | Texto | Separado del RUT (antes iban juntos en un solo campo). |
| RUT del cliente | Texto validado | Se muestra con puntos en pantalla (`12.345.678-5`) para lectura, pero **se registra sin puntos, con guión** (`12345678-5`). Dígito verificador validado en vivo con Módulo 11 (ver algoritmo abajo). |
| Marca del vehículo | Selección | Catálogo `referencia/marcas_usados_catalogo.csv`, solo nombre de marca visible (103 marcas únicas). |
| Código de marca | Campo oculto (no se ve en el front) | Se completa automáticamente al elegir la Marca (lookup marca→código) y viaja igual al registro del back — no se muestra ni se digita a mano. Si la marca tiene dos códigos en el catálogo fuente (ver nota de duplicados abajo), el campo guarda ambos separados por "/". |
| Modelo del vehículo | Texto | Separado de Marca (antes iban juntos). |
| Financiera 1ª opción (rechazó) | Selección | Finex, Amicar, Forum, Global, … |
| Motivo de rechazo | Selección | Score/DICOM, Renta insuficiente, Documentación, Otro. |
| Financiera 2ª opción sugerida | Selección (auto-sugerida) | Se sugiere según la marca; el ejecutivo puede cambiarla. Lógica de sugerencia real **pendiente de confirmar**. |
| Comentario adicional | Texto libre | — |

### Algoritmo de validación de RUT (Módulo 11)

1. Tomar el número base (sin puntos, sin guion, sin dígito verificador).
2. Multiplicar cada dígito, de derecha a izquierda, por la secuencia `2,3,4,5,6,7`
   (se repite si el número tiene más de 6 dígitos).
3. Sumar los productos.
4. `resto = suma % 11`.
5. `resultado = 11 - resto`.
6. DV: `11`→`0`, `10`→`K`, `0-9`→ el número mismo.

Implementado en JavaScript dentro del mockup (ver Artifact).

## Datos de salida

**Lista `Maestro Responsables`** (mantenida a mano, referencia):

| Sucursal | Financiera | Responsable | Correo |
|---|---|---|---|

**Lista `Casos Relevo Financiero`** (transaccional, un registro por envío del formulario):

| Campo | Origen |
|---|---|
| Fecha | Automático al crear el registro |
| Sucursal (+ Código API) | Formulario |
| Ejecutivo que reporta | Formulario |
| Nombre y RUT del cliente | Formulario (RUT sin puntos, con guion) |
| Marca, Código de marca (autocompletado) y Modelo | Formulario |
| Financiera 1ª opción y motivo de rechazo | Formulario |
| Financiera 2ª opción sugerida | Formulario (auto-sugerida, editable) |
| Responsable notificado | Power Automate, cruzando Sucursal + Financiera 2 contra `Maestro Responsables` |
| Estado | Pendiente → Notificado → Resuelto |
| Fecha de resolución | Actualizado manualmente por el responsable |

Esta lista, con el tiempo, es la base para medir **qué porcentaje de rechazos se
recuperan en la segunda opción** — una métrica que hoy no existe.

## Reglas de negocio

Ver `Proyectos/RepositorioUsados/reglas-negocio/relevo-financiero-financiamiento.md`
para las reglas de Finex/Amicar/Global confirmadas en la reunión de origen.

## Catálogos de referencia

- **`referencia/sucursales_usados_codapi.csv`** — 41 sucursales con su código API.
  Fuente: `Sucursales_Usados_CodAPI.xlsx` (descargado 2026-08-21).
- **`referencia/marcas_usados_catalogo.csv`** — 116 pares marca+código, deduplicados
  por (nombre normalizado, código exacto) a partir del export
  `verautmarcasautos_20_08_2026_17_20.xlsx` (126 filas originales, con nombres
  repetidos en distinta capitalización). Normalización de nombre aplicada:
  primera letra mayúscula, resto minúscula.
- **`referencia/marcas_codigos_duplicados_pendientes.csv`** — **14 marcas** quedan con
  dos códigos API distintos en el catálogo fuente (no es un problema de este
  proyecto, sino del catálogo `verautmarcasautos` en origen):

  | Marca | Código A | Código B |
  |---|---|---|
  | Dfsk | 287 | DFS |
  | Honda | 21 | 39 |
  | Hyundai | 10 | 40 |
  | Leapmotor | 324 | LPM |
  | Lifan | 52 | 65 |
  | Mazda | 25 | 57 |
  | Mercedes benz | 28 | 58 |
  | Mini | 53 | 61 |
  | Opel | 38 | 67 |
  | Porsche | 22 | 72 |
  | Ram | 304 | 59 |
  | Shineray | 101 | SHI |
  | Tesla | 323 | TE |
  | Uaz | 313 | 316 |

  Para 8 de ellas (Honda, Hyundai, Lifan, Mazda, Mercedes benz, Mini, Opel, Ram) el
  patrón es el mismo: una carga masiva original (2023-07-27, usuario "Supervisor")
  y una entrada nueva creada por Daniel Abarca Medina el 2024-02-27 con otro código —
  probablemente una re-creación sin desactivar la original. Las otras 6 (Dfsk,
  Leapmotor, Shineray, Tesla, Uaz) tienen ambos códigos creados por "Supervisor" en
  fechas cercanas entre sí — algunos con código alfabético corto (DFS, LPM, SHI, TE)
  que podría ser de un sistema de codificación distinto, no necesariamente un error.
  **No se depuró en este proyecto** porque el catálogo lo mantiene otro equipo — el
  dropdown de Marca del mockup deja ambas opciones visibles (ej. `Hyundai (10)` y
  `Hyundai (40)`) hasta que se resuelva con quien mantiene esa data cuál código
  vigente corresponde a cada marca.

## Estado y próximos pasos

- [x] Mockup funcional publicado como Artifact:
  https://claude.ai/code/artifact/893455e8-0468-41be-829c-6a301ed84c4b
- [ ] Validar campos del formulario y si el motivo de rechazo debe ser obligatorio.
- [ ] Confirmar la regla real de sugerencia de Financiera 2ª opción por marca
  (incluye caso especial: bono con 90% de pie exclusivo de Global).
- [ ] Completar `Maestro Responsables` por sucursal y financiera — sin este dato no
  se puede notificar a la persona correcta.
- [ ] Coordinar con quien mantiene el catálogo `verautmarcasautos` la depuración de
  las 14 marcas con código duplicado (ver tabla arriba).
- [ ] Construir la versión productiva en Microsoft Forms + SharePoint + Power
  Automate (o Power Apps + Dataverse si se necesita más robustez).
