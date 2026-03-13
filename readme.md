# 🚀 Memoria Técnica: Configuración de Infraestructura - Hito 3

**Autores:** Ruth y Adam.  
**Curso:** 2º DAW - IES Hermenegildo Lanz.

## 📑 I. Introducción y Propósito

Para este **Hito 3**, hemos diseñado una arquitectura basada en **microservicios** utilizando **Docker**. Nuestro objetivo es construir un entorno **agéntico** donde **n8n** actúe como orquestador, comunicándose con **Ollama** (IA), **Qdrant** (vectores) y **PostgreSQL** (datos relacionales).

Hemos priorizado la **persistencia de datos** y la **comunicación interna entre servicios** mediante una **red aislada**, con el fin de garantizar la estabilidad y correcto funcionamiento de los agentes.

## 🛠️ II. Descripción de la Configuración

### Fase 1: Despliegue de la Infraestructura (Docker)

#### 1. Definición del archivo `docker-compose.yml`

Hemos configurado un **stack con 5 servicios clave**. La decisión técnica más importante ha sido **unificarlos en una red llamada `hito3_net`** y definir **volúmenes locales** para no perder el progreso ni los datos generados por los contenedores.

Los servicios definidos en el `docker-compose.yml` son los siguientes:

- **n8n**  
  Motor de automatización donde se diseñan y ejecutan los workflows.  
  **Puerto:** `5678`

- **PostgreSQL 16**  
  Base de datos relacional utilizada para almacenar **metadatos y memoria del sistema**.  

- **Qdrant**  
  Base de datos **vectorial** utilizada en el **Sistema RAG** para almacenar los embeddings de los documentos.  
  **Puerto:** `6333`

- **Ollama**  
  Servidor de inferencia que ejecuta **modelos LLM de forma local** para generación de texto y embeddings.  
  **Puerto:** `11434`

- **pgAdmin**  
  Interfaz web para la **gestión y visualización de PostgreSQL**.  
  **Puerto:** `5050`

#### 2. Levantamiento de los servicios

Desde la terminal, situados en la **carpeta del proyecto**, ejecutamos el comando para **construir y levantar los contenedores en segundo plano**:

```bash
docker-compose up -d
```
Usamos **-d** para que los servicios sigan corriendo mientras seguimos trabajando en la terminal.

![Contenedores Docker levantados](/img/contenedores.png)

### Fase 2: Aprovisionamiento de la IA (Ollama)

Una vez levantado el servicio, el contenedor de **Ollama** está activo pero **carece de modelos**. Procedemos a descargar los modelos requeridos mediante comandos de ejecución remota en el contenedor.

- **Mistral**: Lo utilizaremos para el **análisis de intención** y la **generación de respuestas**.

```bash
docker exec -it ollama_hito3 ollama pull mistral
```

- **Nomic-Embed-Text**: Fundamental para el **Proyecto A**, ya que se encarga de **transformar el texto en vectores de 768 dimensiones**.

```bash
docker exec -it ollama_hito3 ollama pull nomic-embed-text
```

- **Verificación:** Comprobamos que ambos modelos están disponibles ejecutando:

```bash
docker exec -it ollama_hito3 ollama list
```

![Lista de Modelos Ollama](/img/lista_modelos.png)

### Fase 3: Administración de Datos (pgAdmin y PostgreSQL)

Para gestionar la persistencia estructurada que requiere el Hito 3, utilizamos **pgAdmin 4** como interfaz gráfica de administración. El objetivo en esta fase es vincular nuestro motor de base de datos relacional con el gestor y definir el esquema de tablas solicitado por el profesor.

#### 1. Acceso y Autenticación
Accedemos a la consola de administración a través del navegador en `http://localhost:5050`. Para el inicio de sesión, utilizamos las credenciales de administrador que definimos previamente en nuestro archivo de infraestructura de Docker:
* **Usuario:** `admin@admin.com`
* **Contraseña:** `admin`

![Pantalla de Login de pgAdmin](/img/inicio_sesion_pgAdmin.png)

#### 2. Registro y Conexión del Servidor
Para que pgAdmin pueda comunicarse con el contenedor de PostgreSQL dentro de la red virtual `hito3_net`, debemos registrar el servidor siguiendo estos pasos técnicos:

1.  Hacemos clic derecho en **Servers** > **Register** > **Server...**
2.  En la pestaña **General**, asignamos el nombre identificativo: `Servidor-Hito3`.
3.  En la pestaña **Connection**, introducimos los parámetros de red interna del stack:
    * **Host name/address:** `postgres` *(Nombre del servicio definido en el Docker Compose)*.
    * **Port:** `5432`.
    * **Maintenance database:** `hito3_db`.
    * **Username:** `admin`.
    * **Password:** `hito3_password`.

*Justificación Técnica:* Al usar el hostname `postgres`, pgAdmin utiliza el DNS interno de Docker para localizar el contenedor de la base de datos sin necesidad de conocer su IP privada variable.

![Registro del Servidor en pgAdmin](/img/pgAdmin_confi1.png)
![Configuración de Conexión en pgAdmin](/img/pgAdmin_confi2.png)

#### 3. Creación del Esquema Relacional (SQL)
Una vez establecida la conexión con la base de datos `hito3_db`, procedemos a crear la estructura necesaria para el **Proyecto A (Sistema RAG)**. Abrimos la herramienta **Query Tool** y ejecutamos el siguiente script DDL (Data Definition Language):

```sql
CREATE TABLE documentos ( 
    id SERIAL PRIMARY KEY, 
    nombre VARCHAR(255), 
    num_chunks INTEGER, 
    fecha TIMESTAMP DEFAULT NOW() 
);
```

*Explicación Técnica:* Esta tabla `documentos` se diseñó para almacenar los metadatos de cada documento procesado, incluyendo un identificador único (`id`), el nombre del documento, el número de fragmentos generados y la fecha de registro. La columna `id` es autoincremental, lo que facilita la gestión de registros sin necesidad de asignar manualmente un identificador.

![Herramienta Query Tool](/img/query_tool.png)
![Ejecutando el Script SQL en pgAdmin](/img/creacion_tabla.png)

#### 4. Verificación de la Creación de la Tabla
Para confirmar que la tabla `documentos` se ha creado correctamente, navegamos por el árbol de objetos en pgAdmin:
**Servers > Hito3_Server > Databases > hito3_db > Schemas > public > Tables > documentos**
Al hacer clic en la tabla `documentos`, podemos visualizar su estructura y confirmar que los campos se han definido según lo esperado.

![Verificación de la Tabla en pgAdmin](/img/tabla_documentos.png)

### Fase 4: Configuración y Orquestación en n8n

Una vez que la infraestructura de microservicios está operativa, procedemos a configurar **n8n** como el núcleo de nuestra automatización. Para que los agentes de IA puedan interactuar con el resto de servicios (Bases de datos y Modelos LLM), realizamos el siguiente proceso de configuración técnica:

#### 1. Registro inicial y Seguridad
Accedemos a la interfaz web a través de `http://localhost:5678`. Al ser el primer inicio, realizamos el registro del **Owner (Propietario)**. Este paso es fundamental, ya que n8n genera una clave de cifrado única basada en este usuario para proteger todas las credenciales que almacenaremos a continuación.

![Crear cuenta de n8n](/img/cuenta_n8n.png)

#### 2. Configuración de Credenciales de Conexión
Para que los nodos de n8n puedan comunicarse con los contenedores dentro de la red privada `hito3_net`, hemos configurado tres conexiones específicas. Un punto clave en este paso es que, al estar en un entorno Dockerizado, no utilizamos `localhost` como nombre de host, sino los **nombres de servicio** definidos en nuestro archivo YAML.

Hemos configurado las siguientes credenciales en la sección **Credentials > Add Credential**:

* **PostgreSQL:**
    * **Host:** `postgres` (Nombre del servicio definido en Docker).
    * **Database:** `hito3_db`.
    * **User:** `admin`.
    * **Password:** `hito3_password`.
    * *Propósito:* Permitir que el sistema registre los metadatos de los documentos y gestione la memoria del chat.

* **Ollama:**
    * **Base URL:** `http://ollama:11434`.
    * *Propósito:* Establecer el puente con los modelos `mistral` (razonamiento) y `nomic-embed-text` (vectores).

* **Qdrant API:**
    * **Base URL:** `http://qdrant:6333`.
    * *Propósito:* Definir el destino donde se almacenarán y consultarán los fragmentos vectorizados de los documentos.

#### 3. Verificación de Vinculación
Para cada una de estas credenciales, hemos verificado la conectividad mediante el botón **"Save"**. El sistema confirma mediante un indicador visual en verde que n8n tiene acceso total a la infraestructura, quedando así el entorno totalmente preparado para el desarrollo de los flujos lógicos.

![Verificación de Credenciales en n8n](/img/lista_credenciales.png)

### Fase 5. Verificación de la Base Vectorial (Qdrant)
Como ultimo paso de la configuración previa al desarrollo de los proyectos, comprobamos el estado de **Qdrant** accediendo a su interfaz de administración a través de `http://localhost:6333/dashboard`. En esta consola, verificamos que el servicio está activo y listo para recibir las colecciones de vectores que generaremos a partir de los documentos.

![Dashboard de Qdrant](/img/qdrant_dashboard.png)

## 🚀 III. Desarrollo del Proyecto A: Sistema RAG Educativo (Fase 1 - Ingesta)

El objetivo de esta fase es procesar un archivo de texto, fragmentarlo y almacenarlo en una arquitectura de persistencia híbrida: vectorial (**Qdrant**) para la búsqueda semántica y relacional (**PostgreSQL**) para la auditoría. 

### Paso 1: Disparador y Lectura de Archivos
#### Nodo: Manual Trigger
Este nodo actúa como el punto de inicio del flujo. Al ser un disparador manual, nos permite controlar cuándo se ejecuta el proceso de ingesta, lo que es ideal para pruebas y desarrollo iterativo.

#### Nodo: Read File from Disk
Este nodo se encarga de leer el contenido del archivo de texto que queremos procesar. Para ello, hemos montado un volumen local en el contenedor de n8n que apunta a la carpeta `./documentos:/home/node/.n8n-files` en nuestro sistema host. Esto nos permite colocar cualquier archivo de texto en esa carpeta y que n8n pueda acceder a él sin problemas.

En file selector del nodo, seleccionamos la ruta del archivo dentro del contenedor, por ejemplo: `/home/node/.n8n-files/almacen.pdf`. Este nodo leerá el contenido completo del archivo y lo pasará al siguiente nodo para su procesamiento. 

![Configuración del Nodo Read File](/img/read_file.png)

### Paso 2: IA y Vectorización
Añadimos el nodo **Qdrant Vector Store** para procesar la información. En este paso, es fundamental seleccionar las **credenciales de Qdrant que creamos anteriormente** en la sección de configuración inicial.

#### Configuración del Nodo Qdrant Vector Store
- **Operation:** `Insert Documents` (Operación para insertar nuevos documentos en la base vectorial).
- **Collection Name:** `documentos_hito3` (Nombre de la colección donde se almacenarán los vectores).

![Configuración del Nodo Qdrant](/img/qdrant_nodo.png)

#### Configuración de SubNodos para Vectorización
- **Embeddings Ollama:** Conectado al puerto Embedding. Seleccionamos la credencial de Ollama que creamos antes y elegimos el modelo `nomic-embed-text` para generar los vectores a partir del texto del documento.

![Configuración del SubNodo de Embeddings](/img/embedding_nodo.png)

- **Default Document Loader:** Conectado al puerto Document. 
   - **Type of Data:** `Binary` (para procesar el contenido del archivo leído en formato binario).
   - **Mode:** `Load All Input Data` (para cargar todo el contenido del archivo de una sola vez, lo que es adecuado para documentos de tamaño moderado).
   - **Data Format:** `Automatically Detect by Mime Type` (para que el sistema determine el formato del documento de forma inteligente).
   - **Text Splitting:** `Custom` (para permitir la configuración personalizada del splitter que fragmentará el texto en partes manejables).

![Configuración del SubNodo Document Loader](/img/document_loader.png)

- **Recursive Character Text Splitter:** Conectado a la base del Loader. Configuramos el `chunk size` a 500 y el `chunk overlap` a 50 para fragmentar el texto en partes manejables, lo que mejora la calidad de los vectores generados y la posterior búsqueda semántica.

![Configuración del SubNodo Text Splitter](/img/text_splitter.png)

### Paso 3: Consolidación y Registro en PostgreSQL
Para evitar que se inserten múltiples filas por un solo un archivo, implementamos una lógica de consolidación de datos.

#### Nodo: Aggregate
Este nodo recibe los fragmentos generados por el splitter y los agrupa en un único paquete de salida. Esto asegura que el siguiente paso solo se ejecute una vez.
-**Aggregate:** Seleccionamos `All Item Data` para consolidar toda la información en un solo bloque.

![Configuración del Nodo Aggregate](/img/aggregate_nodo.png)

#### Nodo: PostgreSQL (Insert Rows)
Seleccionamos la credencial de PostgreSQL que creamos en la configuración inicial y configuramos el nodo para insertar una nueva fila en la tabla `documentos` por cada archivo procesado. En este nodo, mapeamos los campos de la tabla con los datos correspondientes:
- **nombre:** Extraemos el nombre del archivo utilizando la función `{{ $('Read/Write Files from Disk').item.binary.data.fileName }}`.
- **num_chunks:** Recuperamos el conteo de los fragmentos originales antes de ser agrupados con esta expresión `{{ $json.data.length }}`.

![Configuración del Nodo PostgreSQL](/img/postgresql_nodo.png)

### Paso 4: Ejecución y Verificación
Al ejecutar el flujo, el sistema procesa el archivo de texto, genera los vectores correspondientes y los almacena en Qdrant, mientras que en PostgreSQL se registra una nueva fila con los metadatos del documento. Para verificar que todo ha funcionado correctamente, podemos consultar la tabla `documentos` en pgAdmin y revisar la colección `documentos_hito3` en el dashboard de Qdrant.

![Verificación en pgAdmin](/img/pgadmin_verificacion.png)
![Verificación en Qdrant](/img/qdrant_verificacion.png)

### Vista General del Flujo de Ingesta
A continuación se muestra una vista general de todos los nodos descritos anteriormente.

![Vista General del Flujo de Ingesta](/img/flujo_general.png)

## 🚀 IV. Desarrollo del Proyecto A: Sistema RAG Educativo (Fase 2 - Consulta)
El objetivo de esta segunda fase es construir el agente que permita al usuario interactuar y hacer preguntas en lenguaje natural sobre los documentos que hemos vectorizado e indexado previamente en la base de datos en la Fase 1. 

### Paso 1: Interfaz de Chat (Entrada de la Pregunta)
**Nodo: When chat message received**
Al igual que en otros flujos interactivos, este nodo sirve como la puerta de entrada. 
- **Tipo de nodo:** Chat Trigger.
- **Funcionamiento:** Se queda a la escucha esperando el mensaje escrito por el usuario en n8n. La pregunta entrará por este nodo y desencadenará el inicio del sistema RAG.

### Paso 2: Motor Central de Recuperación y Respuesta (QA Chain)
**Nodo: Question and Answer Chain**
Este el núcleo inteligente o "cerebro" de toda la arquitectura RAG. Su función es bidireccional: primero ordena buscar en la base de datos de vectores fragmentos que tengan semejanza con la pregunta, y luego le dicta a la inteligencia artificial cómo debe utilizar exclusivamente esos recortes de texto para hilar la respuesta.

Al ser un super-nodo de tipo "Chain", delega la carga en 4 subnodos conectados:

1. **Subnodo de Inferencia IA (Ollama Chat Model):**
   - Conectado al puerto de modelo de lenguaje (`ai_languageModel`).
   - Seleccionamos la credencial local `Ollama account` y usamos de igual forma el modelo `mistral:latest`.
   - **Atención - Configuración Especial:** Hemos bajado expresamente el parámetro **Temperature** (Temperatura) a `0.1`. Esta decisión técnica es fundamental porque obliga al Mistral a ser matemático, determinista y extremadamente preciso, impidiendo que divague, invente o sea creativo.

![Configuración del SubNodo de Inferencia IA](/img/inferencia_ia.png)

2. **Subnodo de Búsqueda (Vector Store Retriever):**
   - Conectado al puerto de recuperación (`ai_retriever`). Su trabajo es recolectar y transportar los documentos de origen, actuando como un simple nexo hacia la base vectorial.
   - Le configuramos un limite de recuperación de `4` documentos para que solo traiga los fragmentos más relevantes y no sature al modelo de lenguaje con información innecesaria.

![Configuración del SubNodo de Búsqueda](/img/busqueda_nodo.png)

3. **Subnodo de Almacén Vectorial (Qdrant Vector Store):**
   - Se alimenta del Retriever (`ai_vectorStore`).
   - Apuntamos a nuestra credencial `QdrantApi account`.
   - Seleccionamos exactamente la colección que poblamos en el paso anterior: `documentos_hito3`.

![Configuración del SubNodo de Almacén Vectorial](/img/almacen_vectorial.png)

4. **Subnodo del Traductor Semántico (Embeddings Ollama):**
   - Conectado directamente a las tripas de Qdrant (`ai_embedding`).
   - Selecionamos nuestra credencial y apuntamos al modelo `nomic-embed-text:latest`. Esto hace que cada una de las frases que nos escribe un usuario humano en el chat sea transformada instantáneamente al idioma matemático de los vectores de 768 dimensiones para ser comparada con el almacén.

![Configuración del SubNodo de Embeddings](/img/embedding_qa.png)

**Prompt Estricto Anti-Alucinaciones:**
Dentro de las opciones de nuestro nodo emisor, el **Question and Answer Chain**, configuramos el apartado **System Prompt Template** inyectando dinámicamente nuestra variable nativa del lenguaje langchain `{context}`. El grandioso prompt utilizado es el siguiente:

```text
PROHIBICIÓN ESTRICTA: NO UTILICES TU CONOCIMIENTO GENERAL.

Responde ÚNICAMENTE basándote en los fragmentos de texto del {context}.

REGLAS DE ORO:
1. Si la respuesta no está en el contexto, responde EXACTAMENTE: "Lo siento, este tema no está tratado en los documentos del curso."
2. Tienes terminantemente PROHIBIDO decir "sin embargo", "aunque" o proporcionar respuestas basadas en tu propia memoria (como capitales, cultura o hechos generales).
3. Si el contexto no contiene la información, NO respondas a la pregunta bajo ninguna circunstancia.

----------------
CONTEXTO DEL CURSO:
{context}
```

![Configuración del Prompt en el Nodo QA Chain](/img/prompt_qa.png)

### Paso 3: Retorno de la Respuesta al Usuario
**Nodo: Edit Fields (Set)**
Al finalizar de reescribir la información, el formato devuelto por el nodo de respuesta (Q&A Chain) almacena los datos alojados en un campo predeterminado llamado `response`. Ya que el sub-sistema de chat de la ventana de n8n requiere que la respuesta final esté encapsulada puntualmente en una llave o variable llamada `output`, necesitamos realizar esta asignación:
- Añadimos un Assignment de tipo `String`.
- **Name:** Introducimos de forma rígida `output`.
- **Value:** Aplicamos la fórmula directa `={{ $json.response }}`.

![Configuración del Nodo Edit Fields (RAG)](/img/rag_edit_fields.png)

### Vista General del Flujo de Consulta (RAG)
Con un despliegue muy minimalista visualmente pero enorme por su capacidad algorítmica y de cadenas bajo el capó (LangChain), disponemos de una potentísima estructura anti-engaños, que solamente devuelve información local y privada.

A continuación se muestra una vista general del flujo completo tras configurar todo el motor de consulta en n8n:

![Vista General del Flujo RAG](/img/flujo_rag.png)

## 🤖 IV. Desarrollo del Proyecto B: Chatbot Multiherramienta

El objetivo de este proyecto es construir un asistente conversacional inteligente capaz de identificar la intención del usuario y consultar diferentes herramientas externas (APIs de Wikipedia, Clima y Países) para proporcionar respuestas precisas y naturales. Además, dejaremos constancia de cada interacción registrando todo el historial en **PostgreSQL**.

### Paso 0: Creación de la Tabla de Historial (pgAdmin)
Tal y como hicimos en la fase de configuración del Proyecto A, el primer paso antes de orquestar el flujo en n8n es preparar la base de datos donde persisteremos los datos.

Accedemos a nuestra interfaz de **pgAdmin** (`http://localhost:5050`), abrimos la herramienta **Query Tool** conectada a nuestra base de datos `hito3_db` y ejecutamos el siguiente script DDL:

```sql
CREATE TABLE historial_conversaciones (
    id SERIAL PRIMARY KEY,
    pregunta TEXT,
    respuesta TEXT,
    herramienta_usada VARCHAR(255),
    fecha TIMESTAMP DEFAULT NOW()
);
```

*Explicación Técnica:* La tabla `historial_conversaciones` sirve como registro de auditoría. La columna `id` garantiza una identificación única y autoincremental de cada chat, `pregunta` y `respuesta` capturarán los intercambios textuales de manera literal, y `herramienta_usada` nos servirá para analizar posteriormente qué intenciones (`WIKI`, `CLIMA`, `PAIS`, etc.) ha derivado con éxito nuestro modelo clasificador. La `fecha` se auto-registra con el valor predeterminado del sistema.

![Creación de la Tabla Historial en pgAdmin](/img/creacion_tabla_historial.png)

### Paso 1: Disparo y Clasificación de Intención
Para comenzar, necesitamos recibir el mensaje del usuario y analizar qué es lo que realmente nos está pidiendo.

#### Nodo: When chat message received
Este nodo actúa como el punto de inicio del flujo de conversación.
- **Tipo de nodo:** Chat Trigger.
- **Funcionamiento:** Queda a la escucha de las interacciones del usuario en el panel de chat integrado de n8n. No requiere configuración compleja, simplemente captura el texto introducido por el usuario, que después usaremos como punto de partida.

#### Nodo: Basic LLM Chain (Clasificador IA)
Conectado al chat, este nodo de IA actúa como un enrutador estructurado en lugar de un conversador libre. Su misión es clasificar el mensaje.
- **Inteligencia Artificial (Subnodo):** Añadimos un subnodo **Ollama Chat Model**, conectando nuestra credencial `Ollama account`, y seleccionamos el modelo `mistral:latest`.

![Configuración del SubNodo de Clasificación IA](/img/subnodo_clasificador.png)

- **Configuración del Nodo:** Activamos la opción **Require Specific Output Format** para asegurarnos de que la IA solo nos devuelva respuestas estructuradas y fáciles de interpretar.
En el campo **Message**, introducimos el siguiente *prompt* estricto para evitar alucinaciones y darle reglas claras:

```text
Eres un micro-servicio de clasificación de texto. Tu respuesta es la entrada de un algoritmo que fallará si añades una sola palabra extra.

Formato OBLIGATORIO de salida: CATEGORIA|BUSQUEDA
CATEGORIAS permitidas: WIKI, CLIMA, PAIS, NADA.
En BUSQUEDA: Pon la entidad principal SIN paréntesis, SIN explicaciones y SIN descripción de lo que haces.

PRIORIDAD DE CATEGORIZACIÓN:
Si preguntan por fechas, descubrimientos, personas, historia o 'qué es/quién fue' -> WIKI|Nombre (siempre en español)
Ejemplo: '¿Cuándo se descubrió América?' -> WIKI|Descubrimiento de America
'¿Quién pintó la Mona Lisa?' -> WIKI|Leonardo Da Vinci
Si preguntan por 'qué tiempo hace', 'clima' o 'temperatura' -> CLIMA|Ciudad
Ejemplo: 'Clima en Madrid' -> CLIMA|Madrid
Si preguntan por 'capital', 'población' o 'país' -> PAIS|Pais en inglés
Ejemplo: 'Capital de España' -> PAIS|Spain
Si saludan o hacen charlas triviales -> NADA/Nada
Ejemplo: 'Que como hoy' -> NADA|Nada

REGLA ANTI-ALUCINACIÓN y PROHIBICIONES:
NO clasifiques como CLIMA nada que no pida explícitamente el tiempo atmosférico.
NO clasifiques como PAIS nada que no pida datos demográficos/políticos.
PROHIBIDO poner explicaciones como '(population)' o '(capital)', osea nada entre ().
PROHIBIDO responder con frases. Solo CATEGORIA|BUSQUEDA.
```

*Justificación Técnica:* Al establecer estas restricciones, este nodo generará salidas perfectas como `CLIMA|Madrid` o `WIKI|Albert Einstein`, que luego podremos diseccionar automáticamente.

![Configuración del Nodo Clasificador IA](/img/clasificador_ia.png)

### Paso 2: Enrutamiento de la Petición
#### Nodo: Switch
Ahora necesitamos desviar el flujo lógico del sistema basándonos en la categoría devuelta por nuestra IA. Utilizamos el nodo **Switch**.
- Añadimos **4 reglas de enrutamiento (Values)**.
- En la evaluación usaremos una pequeña porción de JavaScript puro para partir (`split`) el texto devuelto por la IA usando la barra vertical (`|`).
- Para las cuatro reglas, el **Value 1** es siempre: `={{ $json.text.split('|')[0] }}` (esto nos devuelve la categoría de interés al lado izquierdo del split, por ejemplo la cadena 'CLIMA').
- La **Operation** la configuramos en **Contains** tipo **String**.
- A continuación, configuramos el **Value 2** en cada una de las cuatro reglas correspondientemente para cada ruta:
  - Ruta 0 (Value 2): `WIKI`
  - Ruta 1 (Value 2): `CLIMA`
  - Ruta 2 (Value 2): `PAIS`
  - Ruta 3 (Value 2): `NADA`

![Configuración del Nodo Switch](/img/switch_nodo.png)

### Paso 3: Consulta a Herramientas Externas (APIs)
Dependiendo del cable por donde salga el flujo desde el Switch, ejecutamos una petición a su herramienta correspondiente utilizando la otra mitad de la respuesta de la IA: el término de búsqueda, representado por la expresión `{{ $json.text.split('|')[1] }}`.

#### Nodo HTTP Request: wiki
Consulta información histórica y enciclopédica a Wikipedia.
- **Method:** `GET`
- **URL:** `=https://es.wikipedia.org/api/rest_v1/page/summary/{{ $json.text.split('|')[1] }}`
- **Send Headers:** Es **obligatorio** habilitar esta opción (`true`), ya que Wikipedia deniega y bloquea las peticiones no identificadas.
  - En **Name** establecemos `User-Agent`.
  - En **Value** indicamos el correo: `ruthfaouzimeira@gmail,com`.

![Configuración del Nodo HTTP Request para Wikipedia](/img/http_wiki.png)

#### Nodo HTTP Request: paises
Consulta datos demográficos, políticos y capitales a RestCountries al buscar el país.
- **Method:** `GET`
- **URL:** `=https://restcountries.com/v3.1/name/{{ $json.text.split('|')[1] }}?fullText=true`

![Configuración del Nodo HTTP Request para Países](/img/http_paises.png)

#### Nodos HTTP Request: clima y HTTP Request2 (Doble Petición)
Para dar la información meteorológica, la API de Open-Meteo requiere que le proporcionemos coordenadas exactas (latitud y longitud). Sin embargo, el usuario nos ha pedido el tiempo en una ciudad. Por tanto, tenemos que hacerlo de manera secuencial en dos pasos:
1. **Primer nodo HTTP (clima):** Consulta a la API de *Geocoding* para traducir la ciudad textual en coordenadas numéricas.
   - **URL:** `=https://geocoding-api.open-meteo.com/v1/search?name={{ encodeURIComponent($json.text.split('|')[1].trim()) }}&count=1&language=es`

![Configuración del Nodo HTTP Request para Geocoding](/img/http_geocoding.png)

2. **Segundo nodo HTTP (HTTP Request2):** Conectado al nodo de clima anterior. Pasa estas coordenadas devueltas para hacer ya la consulta real de temperatura actual.
   - **URL:** `=https://api.open-meteo.com/v1/forecast?latitude={{ $json.results[0].latitude }}&longitude={{ $json.results[0].longitude }}&current_weather=true`

![Configuración del Nodo HTTP Request para Clima](/img/http_clima.png)

#### Nodo Set: nada
Si el usuario dice "Hola", "Nada" o cualquier charla trivial (NADA), lo derivamos aquí a un nodo **Set** pre-configurado para asignar nuestra respuesta de disculpa manualmente y no gastar inútilmente peticiones o procesamiento hacia ninguna API.
- Añadimos un Assignment de tipo `String`.
- **Name:** `output`
- **Value:** `Lo siento, no he podido identificar tu intención o no tengo información sobre este tema en mis APIs configuradas.`

![Configuración del Nodo Set para NADA](/img/set_nada.png)

### Paso 4: Interpretación de Datos y Respuesta Natural
#### Nodo: Basic LLM Chain1 (Intérprete)
Los datos que nos devuelven las APIs de Wikipedia, RestCountries y el de Open-Meteo, son en formato JSON muy rudimentario. Para crear una experiencia humana fluida, mandamos esta última información a un segundo eslabón de IA encargado de comprender en qué consisten estos datos técnicos y relatar una contestación natural que sea útil.

- Le conectamos nuevamente a su base un subnodo **Ollama Chat Model** seleccionado con nuestro mismo modelo `mistral:latest`.
- En su configuración, establecemos en el campo **Prompt (User Message)** la pregunta inicial original del usuario para que sepa de qué estábamos hablando desde que comenzó el flujo: `={{ $('When chat message received').item.json.chatInput }}`
- En **Messages**, le insertamos un grandioso prompt para que elija cómo darnos detalles con estilo educativo pero muy riguroso:

```text
Eres un asistente experto en interpretación de datos de APIs. Tu objetivo es convertir datos técnicos en una respuesta natural y educativa para el usuario.

CONFIGURACIÓN DE RESPUESTA:
Responde siempre en Español.
Sé breve y directo.
Utiliza ÚNICAMENTE los datos proporcionados. No inventes información ni des más informacion de la que se pide.

GUÍA DE INTERPRETACIÓN (Solo para Clima):
Si recibes un 'weathercode', tradúcelo según esta tabla:

0: Cielo despejado ☀️
1, 2, 3: Parcialmente nublado ⛅
45, 48: Niebla 🌫️
51, 53, 55: Llovizna 🌦️
61, 63, 65: Lluvia 🌧️
71, 73, 75: Nieve ❄️
95, 96, 99: Tormenta ⚡
CONTEXTO ACTUAL:
PREGUNTA DEL USUARIO: {{ $('When chat message received').item.json.chatInput }}
DATOS CRUDOS DE LA HERRAMIENTA: {{ JSON.stringify($json) }}
Instrucción final: Basándote en el código de clima anterior (si existe) y en los datos crudos, redacta la respuesta final para el usuario.
```

![Configuración del Nodo Intérprete IA](/img/interprete_ia.png)

### Paso 5: Persistencia del Historial en PostgreSQL
#### Nodo: Insert rows in a table (PostgreSQL)
Aprovechando la gran base de datos relacional PostgreSQL de la red virtual de Docker que ya teníamos inicializada, procederemos a salvaguardar para documentar y auditar cada ciclo del chat. Registraremos qué ha sido solicitado, qué fue respondido, y qué método procesamos. La tabla que creamos para esto fue `historial_conversaciones`, y el proceso de configuración es el siguiente:
- Seleccionamos la credencial creada previamente `Postgres account`.
- **Table:** `historial_conversaciones`.
- **Mapping Column Mode:** Seleccionamos **Map Each Column Manually** y de allí asignamos nuestras correspondientes variables generadas para llenar esos tres campos primordiales:
  - **pregunta:** Viajamos al pasado para extraer en remoto cuál fue la pregunta gracias a esta expresión donde solicitamos esa variable al chatTrigger: `={{ $('When chat message received').item.json.chatInput }}`
  - **respuesta:** Recuperamos la bonita contestación narrada y final (generada o por nuestro Intérprete IA o en su defecto por el nodo de Error estático): `={{ $json.text || $json.output }}`
  - **herramienta_usada:** Obtenemos qué ruta utilizamos al clasificar de IA extrayendo la izquierda del Split: `={{ $('Basic LLM Chain').item.json.text.split('|')[0] }}`

![Configuración del Nodo PostgreSQL Historial](/img/postgres_historial.png)

### Paso 6: Formateo de Salida
#### Nodo: Edit Fields
El último acto recae en un nodo posterior a nuestra subida en PostgreSQL. Ya que los datos devueltos provienen directamente como salida a raíz de la introducción y guardado en la base de PostgreSQL, en n8n utilizaremos una sustitución mediante la directiva **Edit Fields (Set)** devolviendo un único ítem presentable.
- Seleccionamos Add Assignment y elegimos el tipo `String`.
- **Name:** Requerimos indicar tajantemente la clave `output` (ya que es crucial usar esta bandera porque es el parámetro o variable obligatoria a la que queda a la escucha la ventana principal del n8n Chat para responder textualmente allí nuestro texto). 
- **Value:** `={{ $json.respuesta }}`.

![Configuración del Nodo Edit Fields](/img/edit_fields_nodo.png)

### Vista General del Flujo Multiherramienta
De esta forma, hemos construido con éxito integral un avanzado e iterativo flujo dinámico, asíncrono y condicionado, siendo inteligente a la hora de estructurar un texto a un prompt, extrayéndonos datos y evaluándose tras requerir de varias APIs integradas, documentándose todo el sistema mediante una auto-inserción en una DB local permanente.

A continuación se muestra una vista general de cómo interactúan todos los nodos del agente conversacional.

![Vista General del Flujo Chatbot](/img/flujo_chatbot.png)