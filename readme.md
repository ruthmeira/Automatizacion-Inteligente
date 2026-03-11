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

- **Llama3**: Lo utilizaremos para el **análisis de intención** y la **generación de respuestas**.

```bash
docker exec -it ollama_hito3 ollama pull llama3
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

### 🐘 Fase 3: Administración de Datos (pgAdmin y PostgreSQL)

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
    * *Propósito:* Establecer el puente con los modelos `llama3` (razonamiento) y `nomic-embed-text` (vectores).

* **Qdrant API:**
    * **Base URL:** `http://qdrant:6333`.
    * *Propósito:* Definir el destino donde se almacenarán y consultarán los fragmentos vectorizados de los documentos.

#### 3. Verificación de Vinculación
Para cada una de estas credenciales, hemos verificado la conectividad mediante el botón **"Save"**. El sistema confirma mediante un indicador visual en verde que n8n tiene acceso total a la infraestructura, quedando así el entorno totalmente preparado para el desarrollo de los flujos lógicos.

![Verificación de Credenciales en n8n](/img/lista_credenciales.png)

### Fase 5. Verificación de la Base Vectorial (Qdrant)
Como ultimo paso de la configuración previa al desarrollo de los proyectos, comprobamos el estado de **Qdrant** accediendo a su interfaz de administración a través de `http://localhost:6333/dashboard`. En esta consola, verificamos que el servicio está activo y listo para recibir las colecciones de vectores que generaremos a partir de los documentos.

![Dashboard de Qdrant](/img/qdrant_dashboard.png)

## 🚀 III. Desarrollo del Proyecto A: Sistema RAG Educativo


