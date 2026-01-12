# Sender of laboral offers — Documentación

**ID del flujo:** `zalh8R1yo1fgZ_VfOcLes`

**Resumen:**
Este workflow scrapea ofertas de trabajo (fullstack) desde `https://www.getonbrd.com/empleos-fullstack`, filtra las publicadas en los últimos 20 días y envía un mensaje por Discord con los detalles de cada oferta (título, link y fecha). Puede ejecutarse manualmente o mediante un trigger de programación.

---

## Tabla de contenidos

1. Objetivo
2. Requisitos previos
3. Diagrama lógico del flujo
4. Descripción nodo a nodo
5. Explicación del código JavaScript (nodo `Code in JavaScript`)
6. Ejemplo de salida esperada
7. Configuración de credenciales y permisos (Discord)
8. Uso, pruebas y despliegue
9. Manejo de errores y logging
10. Mejoras recomendadas
11. Registro de cambios / notas

---

## 1) Objetivo

Automatizar la recolección diaria (o manual) de ofertas de empleo desde GetOnBrd y notificar a un usuario vía Discord con las ofertas nuevas (últimos 20 días), limitando la cantidad enviada a 10 por ejecución.

## 2) Requisitos previos

* Instancia de n8n funcionando (puede ser Docker, n8n.cloud o autoalojada).
* Credenciales del bot de Discord creadas y configuradas en n8n (OAuth token / Bot token) con permiso de enviar mensajes directos al usuario objetivo.
* Acceso a la URL objetivo (`https://www.getonbrd.com/empleos-fullstack`) desde el servidor donde corre n8n.
* Recomendado: entorno con zona horaria correcta (el flujo no depende de variables de entorno, pero la interpretación de `new Date()` depende del timezone del servidor).

## 3) Diagrama lógico del flujo

* Trigger manual (`When clicking 'Execute workflow'`) o `Schedule Trigger` → `HTTP Request` → `HTML` (parser) → `Code in JavaScript` (filtrado y normalización de fechas) → `Limit` (max 10) → `Send a message` (Discord)

## 4) Descripción nodo a nodo

A continuación explico cada nodo con sus parámetros principales y su propósito.

### 4.1 When clicking ‘Execute workflow` (manualTrigger)

* **Tipo:** `manualTrigger`
* **Uso:** Permite ejecutar el flujo manualmente desde la UI de n8n.
* **Conexión:** salida → `HTTP Request`.

### 4.2 Schedule Trigger

* **Tipo:** `scheduleTrigger` (v1.3)
* **Parámetros relevantes:**

  * `rule.interval` está presente pero vacío en el JSON. Debes definir la programación deseada (por ejemplo `cron: 0 8 * * *` para ejecutar diariamente a las 8:00).
* **Uso:** Ejecuta automáticamente en la programación configurada. Conecta también a `HTTP Request`.

**Importante:** Si quieres que el flujo corra diariamente, edita `rule.interval` o utiliza la UI para configurar la periodicidad.

### 4.3 HTTP Request

* **Tipo:** `httpRequest` (v4.3)
* **Parámetros:**

  * `url`: `https://www.getonbrd.com/empleos-fullstack`
  * `options`: vacío (puedes añadir headers, user-agent, o configurar timeouts).
* **Uso:** Recupera el HTML de la página de resultados.

**Sugerencias:**

* Añadir header `User-Agent` para simular navegador si la web bloquea requests no-browser.
* Configurar `retry` o `timeout` si la página tarda.

### 4.4 HTML (parser)

* **Tipo:** `html` (extractHtmlContent)
* **Extracciones configuradas:**

  * `links` — selector `.gb-results-list__item`, `returnValue: attribute`, `attribute: href`, `returnArray: true`
  * `title` — selector `.gb-results-list__title`, `returnArray: true`
  * `Companies` — selector `.size0 strong`, `returnArray: true`
  * `dates` — selector `.opacity-half.size0`, `returnArray: true`

**Notas / advertencias:**

* El selector `.gb-results-list__item` puede apuntar a un contenedor en vez del `a` que contiene `href`. Si `links` devuelve `undefined`, cambia el selector a `.gb-results-list__item a` o al selector exacto del `<a>` que tiene el `href`.
* Verifica con la herramienta `Execute node` en n8n para ver la salida real y ajustar los selectores.

### 4.5 Code in JavaScript

* **Tipo:** `code` (js)
* **Resumen:** Normaliza fechas en formato español (meses abreviados) a formato que `Date` pueda parsear, calcula un rango de `today` y `ago = today - 20 días`, y filtra las ofertas cuyo `jobDate` esté dentro del rango. Devuelve `[{title, link, date}, ...]`.
* **Salida:** Arreglo `data` con objetos JSON.

(La explicación detallada del código está en la sección 5.)

### 4.6 Limit

* **Tipo:** `limit`
* **Parámetros:** `maxItems: 10`
* **Uso:** Limita la cantidad de nodos enviados al siguiente paso. Útil para no saturar de mensajes al usuario.

### 4.7 Send a message (Discord)

* **Tipo:** `discord` (resource: message)
* **Parámetros clave:**

  * `resource`: `message`
  * `sendTo`: `user`
  * `userId`: `809015116175376414` (ID del destinatario en Discord)
  * `guildId`: `1206784516442820628` (ID del guild/server)
  * `content`: plantilla `=Nombre de oferta:{{ $json.title }}\nLink: {{ $json.link }}\nFeach: {{ $json.date }}`
  * `credentials`: referencia a `discordBotApi`

**Nota de seguridad:** la plantilla de contenido tiene un signo `=` al inicio; en n8n las expresiones deben respetar la sintaxis — verifica que la plantilla se renderice correctamente en la salida.

## 5) Explicación del código JavaScript (nodo `Code in JavaScript`)

Código original (resumen):

```js
const today = new Date();
const ago = new Date(today);
ago.setDate(today.getDate() - 20);

const months = {
  "ene": "jan",
  "feb": "feb",
  "mar": "mar",
  "abr": "apr",
  "may": "may",
  "jun": "jun",
  "jul": "jul",
  "ago": "aug",
  "sep": "sep",
  "oct": "oct",
  "nov": "nov",
  "dic": "dec"
};

const { links,title,Companies,dates } =$input.first().json;

const data = [];

for (const idx of links.keys()) {
  let clean = dates[idx].trim().toLowerCase();
  let splitted = clean.split(" ");
  if (Object.keys(months).includes(splitted[0])) {
    clean = months[splitted[0]] + " " + splitted[1];
  }

  const jobDate = new Date(clean);
  jobDate.setFullYear(ago.getFullYear());
  if (jobDate >= ago && jobDate <= today) {
    data.push({
      title: title[idx],
      link: links[idx],
      date: clean
    })
  }
}

return data;
```

### Paso a paso

1. `today` y `ago` — se calcula la fecha actual y `ago` que es `today - 20 días`.
2. `months` — diccionario que mapea abreviaciones de meses en español (`ene`, `feb`, ...) a abreviaciones en inglés (`jan`, `feb`, ...). Esto facilita que `new Date("jan 12")` pueda parsearse.
3. Obtiene `links`, `title`, `Companies`, `dates` del primer item del input HTML (`$input.first().json`).
4. Recorre `links.keys()` y por cada índice:

   * Limpia y normaliza `dates[idx]` (trim + lowercase).
   * Separa en palabras: `splitted = clean.split(" ")` — asume que el mes está en `splitted[0]` y el día en `splitted[1]`.
   * Si el primer token es una clave de `months`, la sustituye por la abreviación en inglés y reconstruye `clean` como `'{mesIngles} {día}'`.
   * Crea `jobDate = new Date(clean)` y fuerza el año con `jobDate.setFullYear(ago.getFullYear())`.
   * Si `jobDate` está entre `ago` y `today`, empuja el objeto al array `data`.
5. `return data;` devuelve el array filtrado.

### Observaciones y posibles fallos

* **Dependencia del formato de `dates`:** El código asume que `dates[idx]` tiene formato como `"ene 12"` o `"feb 3"`. Si la web usa formatos diferentes (por ejemplo `12 ene` o `hace 5 días`), el parseo fallará. Verifica con la salida real del nodo `HTML`.
* **Año forzado:** Se fuerza el `jobDate` al año de `ago` con `setFullYear(ago.getFullYear())`. Si `ago` pertenece a un año distinto (por ejemplo cuando hoy es enero y `ago` cae en diciembre del año anterior), puede que la comparación no sea correcta. Mejor estrategia: construir una fecha con día/mes y usar el año actual, o parsear fechas completas si la web las provee.
* **Parsing robusto:** En lugar de depender de `new Date()` con strings, puedes normalizar y construir `new Date(year, monthIndex, day)` o usar una librería (si está disponible en el entorno) para manejar locales.
* **Índices:** El for hace `for (const idx of links.keys())` que asume `links`, `title` y `dates` tienen la misma longitud y orden. Si alguna extracción falla, habrá desalineación.

## 6) Ejemplo de salida esperada

Salida (ejemplo) desde `Code in JavaScript`:

```json
[
  { "title": "Fullstack Developer - Empresa X", "link": "/empleos/fullstack/empresa-x", "date": "ene 05" },
  { "title": "Backend/Fullstack - Empresa Y", "link": "/empleos/fullstack/empresa-y", "date": "dic 30" }
]
```

Después del nodo `Limit`, cada item será enviado al nodo Discord con el template:

```
Nombre de oferta: Fullstack Developer - Empresa X
Link: /empleos/fullstack/empresa-x
Feach: ene 05
```

## 7) Configuración de credenciales y permisos (Discord)

Pasos rápidos para usar `discord` node:

1. Crear una aplicación en el [Portal de Desarrolladores de Discord].
2. Añadir un Bot a la aplicación y copiar el Bot Token.
3. Conceder permisos al bot: al menos `Send Messages` y `Send Messages in DMs` si vas a enviar mensajes directos.
4. Invitar el bot al servidor con el scope `bot` y los permisos adecuados.
5. En n8n, crear una nueva credencial `Discord Bot API` y pegar el token. Asociarla en el nodo `Send a message`.
6. Obtener `userId` y `guildId` (puedes obtener userId haciendo que el bot responda en un canal y ver la ejecución o usar herramientas de Discord).

**Nota:** Si quieres enviar a un canal en lugar de DM, cambia `sendTo` a `channel` y configura `channelId`.

## 8) Uso, pruebas y despliegue

### Pruebas locales

* Ejecuta el nodo `HTTP Request` con "Execute Node" y revisa la salida.
* Ejecuta `HTML` y valida las matrices `links`, `title`, `dates` en la salida.
* Ejecuta `Code in JavaScript` y revisa el array resultante.
* Finalmente, ejecuta el flujo completo con `When clicking 'Execute workflow'`.

### En producción

* Configura el `Schedule Trigger` con la periodicidad deseada (ej. diaria).
* Habilita `active: true` en la UI si corresponde.
* Monitoriza ejecuciones en la sección de 'Executions' de n8n.

## 9) Manejo de errores y logging

* Añadir un nodo `IF` o `Function` para detectar si `links` está vacío y mandar un log o alerta.
* Añadir `Try / Catch` logic: usar el nodo `Error Trigger` de n8n para capturar fallos y notificar por Discord o correo.
* Registrar respuesta del `HTTP Request` (status code, body) para facilitar debugging.

## 10) Mejoras recomendadas

* **Deduplicación:** Mantener un storage (Google Sheets, Airtable, base de datos) con IDs/URLs ya enviadas para evitar reenvíos.
* **Parsing más robusto:** En vez de `new Date` con strings, parsear explícitamente día y mes y construir `Date(year, monthIndex, day)`; soportar formatos como "hace X días".
* **Soporte de páginas paginadas:** Si GetOnBrd usa paginación, iterar sobre todas las páginas.
* **Enriquecimiento de payload:** Incluir Company, ubicación, y un extracto de la descripción.
* **Retries y backoff:** Para el `HTTP Request` en caso de errores.
* **Templating mejorado:** Usar template más rico (embed de Discord) para mostrar campos ordenados.
* **Seguridad:** No exponer tokens ni IDs en el JSON exportado; usar variables/credentials de n8n.

## 11) Registro de cambios / notas

* `versionId`: `d8081d32-e798-407f-ab40-b201ff41984a`
* `instanceId`: `b4ce499dd54e0958a9a7273d6b016310d08a100c93eb53087c344e754aea6959`
* **Activo:** `true` (verificar entorno antes de habilitar en producción).

---

