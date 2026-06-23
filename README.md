# Laboratorio: construir un RAG con Oracle Autonomous Database 26ai

En este laboratorio vas a construir, paso a paso, un sistema de **Retrieval-Augmented Generation (RAG)** usando **Oracle Autonomous Database 26ai** y **Oracle Generative AI**.

El objetivo es que aprendas a cargar documentos en la base de datos, convertir su contenido en vectores, hacer búsquedas semánticas y generar respuestas en lenguaje natural usando el contexto recuperado. Todo el flujo se ejecuta desde la base de datos, sin mover la información a una aplicación externa.

## Qué vas a construir

Al finalizar el laboratorio tendrás:

1. Una base de datos autónoma lista para ejecutar SQL.
2. Una credencial llamada `OCI_CRED` para conectarte con servicios de OCI.
3. Tablas para almacenar documentos y vectores.
4. Un modelo ONNX cargado en la base de datos para generar embeddings.
5. Una función SQL que combina búsqueda vectorial y Oracle Generative AI.
6. Una consulta final que responde preguntas sobre el 
contenido de un PDF.

```mermaid

flowchart LR

    S["sentence<br/><br/>How expensive it is going to be<br/>having a car?"] --> M(("embedding<br/>model"))

    M --> V["[0.12, 0.84, 0.12, 1.92, ..., 2.19]<br/><br/>vector"]

    classDef sentence fill:#FCE7F7,stroke:#F3C2E7,color:#111827,stroke-width:1px;

    classDef model fill:#2F5D50,stroke:#2F5D50,color:#FFFFFF,stroke-width:2px;

    classDef vector fill:#F3F6FF,stroke:#DDE5FF,color:#111827,stroke-width:1px;

    class S sentence;

    class M model;

    class V vector;

    linkStyle 0 stroke:#466A5D,stroke-width:2px;

    linkStyle 1 stroke:#466A5D,stroke-width:2px;

```

## Antes de comenzar

Este laboratorio inicia cuando ya tienes una **base de datos autónoma creada en OCI** y en estado **Available**.

Para evitar repetir pasos básicos, revisa primero las guías del repositorio de tutoriales de OCI:

1. [Creación de credenciales](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Creaci%C3%B3n%20de%20credenciales.md)
2. [Creación de un compartment](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Creaci%C3%B3n%20de%20un%20compartment.md)
3. [Creación de una política](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Creaci%C3%B3n%20de%20una%20pol%C3%ADtica.md)
4. [Crear base de datos autónoma](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Crear%20base%20de%20datos%20aut%C3%B3noma.md)

<u>Importante:</u> si es tu primera vez usando OCI, completa esas guías antes de ejecutar los scripts de este laboratorio. Aquí se asume que ya tienes a mano estos datos:

- `user_ocid`
- `tenancy_ocid`
- `compartment_ocid`
- `private_key`
- `fingerprint`
- Región de OCI, por ejemplo `us-chicago-1`

## Paso 1: Ingresar a la consola SQL

Cuando la base de datos esté en estado **Available**, abre la consola SQL para ejecutar los comandos del laboratorio.

1. Entra a tu base de datos autónoma desde la consola de OCI.
2. Abre **Database actions** o **SQL**, según el idioma de tu consola.
3. Inicia sesión con el usuario administrador de la base de datos.
4. Verifica que puedes abrir una hoja de trabajo SQL.

![Ingreso a la consola SQL](images/image%205.png)

**Resultado esperado:** ya puedes ejecutar sentencias SQL dentro de tu Autonomous Database.

## Paso 2: Configurar credenciales para llamar servicios de OCI

Para que la base de datos pueda conectarse con Oracle Generative AI y Object Storage, necesitas crear una credencial dentro de Autonomous Database.

La credencial se llamará `OCI_CRED` y usará los datos de la API key creada en los pasos previos.

<u>Antes de ejecutar:</u> reemplaza todos los valores entre `<...>` por los datos reales de tu cuenta OCI. No compartas ni subas tu llave privada a un repositorio público.

```sql
BEGIN
  DBMS_CLOUD.DROP_CREDENTIAL(credential_name => 'OCI_CRED');
EXCEPTION
  WHEN OTHERS THEN NULL;
END;
/

DECLARE
  jo JSON_OBJECT_T;
BEGIN
  jo := JSON_OBJECT_T();
  jo.put('user_ocid', '<USER_OCID>');
  jo.put('tenancy_ocid', '<TENANCY_OCID>');
  jo.put('compartment_ocid', '<COMPARTMENT_OCID>');
  jo.put('private_key', q'[<PRIVATE_KEY_PEM>]');
  jo.put('fingerprint', '<FINGERPRINT>');

  DBMS_VECTOR.CREATE_CREDENTIAL(
    credential_name => 'OCI_CRED',
    params          => JSON(jo.to_string)
  );
END;
/
```

<details>
<summary>Ver ejemplo dummy de cómo debe quedar la credencial</summary>

Este ejemplo no funciona para conectarse a OCI. Solo muestra el formato correcto de los valores, especialmente de `private_key`.

Observa que la llave privada se pega completa dentro de `q'[...]'`, incluyendo:

- La línea `-----BEGIN PRIVATE KEY-----`.
- Todas las líneas intermedias de la llave.
- La línea `-----END PRIVATE KEY-----`.
- Los saltos de línea originales.

```sql
BEGIN
  DBMS_CLOUD.DROP_CREDENTIAL(credential_name => 'OCI_CRED');
EXCEPTION
  WHEN OTHERS THEN NULL;
END;
/

DECLARE
  jo JSON_OBJECT_T;
BEGIN
  jo := JSON_OBJECT_T();
  jo.put('user_ocid', 'ocid1.user.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa');
  jo.put('tenancy_ocid', 'ocid1.tenancy.oc1..bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb');
  jo.put('compartment_ocid', 'ocid1.compartment.oc1..cccccccccccccccccccccccccccccccccccccccccccccccccccc');
  jo.put('private_key', q'[-----BEGIN PRIVATE KEY-----
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
-----END PRIVATE KEY-----]');
  jo.put('fingerprint', 'aa:bb:cc:dd:ee:ff:11:22:33:44:55:66:77:88:99:00');

  DBMS_VECTOR.CREATE_CREDENTIAL(
    credential_name => 'OCI_CRED',
    params          => JSON(jo.to_string)
  );
END;
/
```

<u>Validación rápida:</u> si tu llave quedó en una sola línea o si faltan las líneas `BEGIN PRIVATE KEY` y `END PRIVATE KEY`, vuelve a copiarla desde el archivo `.pem`.

</details>

Si necesitas recrear la credencial, puedes eliminarla con este comando y volver a ejecutar el bloque anterior:

```sql
BEGIN
  DBMS_CLOUD.DROP_CREDENTIAL('OCI_CRED');
END;
/
```

### Paso 2.1: Autorizar llamadas de red desde la base de datos

La base de datos necesita permiso para conectarse a servicios externos, como Oracle Generative AI.

```sql
BEGIN
  DBMS_NETWORK_ACL_ADMIN.APPEND_HOST_ACE(
    host => '*',
    ace  => XS$ACE_TYPE(
      privilege_list => XS$NAME_LIST('connect'),
      principal_name => 'ADMIN',
      principal_type => XS_ACL.PTYPE_DB
    )
  );
END;
/
```

**Resultado esperado:** la credencial `OCI_CRED` queda creada y el usuario `ADMIN` puede hacer llamadas de red desde la base de datos.

## Paso 3: Preparar el ambiente RAG

En este paso vas a validar la conexión con Oracle Generative AI, crear las tablas del laboratorio, cargar un PDF, cargar un modelo ONNX y generar vectores a partir del documento.

### Paso 3.1: Validar conexión con Oracle Generative AI

Ejecuta el siguiente bloque para confirmar que la base de datos puede llamar al endpoint de Oracle Generative AI.

<u>Antes de ejecutar:</u> revisa y reemplaza estos dos valores dentro del bloque SQL:

- `p_region`: si tu base de datos no está en Chicago, reemplaza `'us-chicago-1'` por el identificador de tu región. Para encontrarlo, entra a la guía oficial [Regions and Availability Domains](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) y usa el valor de la columna **Region Identifier**. Por ejemplo, Bogotá aparece como `sa-bogota-1`.
- `p_compartment_ocid`: reemplaza el texto `'<COMPARTMENT_OCID>'` completo por el OCID real de tu compartment. El valor final debe quedar entre comillas simples, por ejemplo `'ocid1.compartment.oc1..aaaaaaaa...'`.

En otras palabras, los textos con formato `<...>` son marcadores de posición. Debes borrarlos y escribir tu valor real.

```sql
SET SERVEROUTPUT ON

DECLARE
  p_region           VARCHAR2(200) := 'us-chicago-1';
  p_endpoint         VARCHAR2(500) := 'https://inference.generativeai.' || p_region || '.oci.oraclecloud.com';
  p_compartment_ocid VARCHAR2(200) := '<COMPARTMENT_OCID>';

  resp          DBMS_CLOUD_TYPES.RESP;
  json_response CLOB;
BEGIN
  resp := DBMS_CLOUD.SEND_REQUEST(
    credential_name => 'OCI_CRED',
    uri             => p_endpoint || '/20231130/actions/chat',
    method          => 'POST',
    body            => UTL_RAW.CAST_TO_RAW(
      JSON_OBJECT(
        'compartmentId' VALUE p_compartment_ocid,
        'servingMode' VALUE JSON_OBJECT(
          'modelId' VALUE 'google.gemini-2.5-flash',
          'servingType' VALUE 'ON_DEMAND'
        ),
        'chatRequest' VALUE JSON_OBJECT(
          'messages' VALUE JSON_ARRAY(
            JSON_OBJECT(
              'role' VALUE 'USER',
              'content' VALUE JSON_ARRAY(
                JSON_OBJECT(
                  'type' VALUE 'TEXT',
                  'text' VALUE 'Qué le sucedió a la familia Gómez Ramírez'
                )
              )
            )
          ),
          'apiFormat' VALUE 'GENERIC',
          'maxTokens' VALUE 4000,
          'isStream' VALUE FALSE,
          'numGenerations' VALUE 1,
          'frequencyPenalty' VALUE 0,
          'presencePenalty' VALUE 0,
          'temperature' VALUE 1,
          'topP' VALUE 1.0,
          'topK' VALUE 1
        )
      )
    )
  );

  json_response := DBMS_CLOUD.GET_RESPONSE_TEXT(resp);
  DBMS_OUTPUT.PUT_LINE(json_response);
END;
/
```

**Resultado esperado:** la base de datos devuelve una respuesta JSON del servicio de IA generativa.

### Paso 3.2: Crear la tabla de documentos

Esta tabla guarda el archivo original. En este laboratorio se usará un PDF.

```sql
CREATE TABLE IF NOT EXISTS TB_DATA (
  id           INTEGER GENERATED BY DEFAULT ON NULL AS IDENTITY
               (START WITH 1 CACHE 20) PRIMARY KEY,
  file_name    VARCHAR2(900),
  file_size    INTEGER,
  file_type    VARCHAR2(100),
  file_content BLOB
);
```

### Paso 3.3: Crear la tabla de vectores

Esta tabla guarda los fragmentos de texto y sus embeddings. Cada vector queda asociado al documento original.

```sql
CREATE TABLE IF NOT EXISTS TB_VECTOR_DATA (
  vec_id       NUMBER(*,0) NOT NULL ENABLE,
  embed_id     NUMBER,
  embed_data   VARCHAR2(4000 BYTE),
  embed_vector VECTOR,
  FOREIGN KEY (vec_id) REFERENCES TB_DATA(id)
);
```

**Resultado esperado:** existen dos tablas: `TB_DATA` para documentos y `TB_VECTOR_DATA` para fragmentos vectorizados.

### Paso 3.4: Copiar el PDF desde Object Storage

El siguiente bloque copia un PDF desde un bucket de OCI al directorio `DATA_PUMP_DIR` de la base de datos.

```sql
BEGIN
  DBMS_CLOUD.GET_OBJECT(
    credential_name => 'OCI_CRED',
    object_uri      => 'https://objectstorage.us-chicago-1.oraclecloud.com/n/idi1o0a010nx/b/labs/o/relatorelato-a.pdf',
    directory_name  => 'DATA_PUMP_DIR'
  );
END;
/
```

### Paso 3.5: Copiar el modelo ONNX

Oracle Database puede ejecutar modelos de machine learning dentro de la base de datos. En este laboratorio se usará un modelo de vectorización previamente entrenado para convertir texto en embeddings.

El flujo es el siguiente: primero se copia el archivo `.onnx` a un directorio de Oracle, luego se registra con `LOAD_ONNX_MODEL()` y finalmente se invoca desde SQL.

![Flujo para registrar e invocar un modelo ONNX en Oracle Database](images/onnx_model_flow.svg)

```sql
BEGIN
  DBMS_CLOUD.GET_OBJECT(
    credential_name => 'OCI_CRED',
    object_uri      => 'https://objectstorage.us-chicago-1.oraclecloud.com/n/idi1o0a010nx/b/labs/o/modeall-MiniLM-L6.onnx',
    directory_name  => 'DATA_PUMP_DIR'
  );
END;
/
```

### Paso 3.6: Cargar el modelo ONNX en la base de datos

El modelo quedará registrado con el nombre `ALL_MINILM_L6`. Ese nombre se usará después para generar embeddings.

Más información: [DBMS_VECTOR en Oracle Database](https://docs.oracle.com/en/database/oracle/oracle-database/23/arpls/dbms_vector1.html).

```sql
BEGIN
  DBMS_VECTOR.LOAD_ONNX_MODEL(
    directory  => 'DATA_PUMP_DIR',
    file_name  => 'modeall-MiniLM-L6.onnx',
    model_name => 'all_MiniLM_L6',
    metadata   => JSON(
      '{"function":"embedding","embeddingOutput":"embedding","input":{"input":["DATA"]}}'
    )
  );
END;
/
```

### Paso 3.7: Guardar el PDF en la tabla de documentos

Ahora se inserta el PDF en `TB_DATA`.

```sql
INSERT INTO TB_DATA (file_name, file_size, file_type, file_content)
VALUES (
  'relatorelato-a.pdf',
  DBMS_LOB.GETLENGTH(TO_BLOB(BFILENAME('DATA_PUMP_DIR', 'relatorelato-a.pdf'))),
  'PDF',
  TO_BLOB(BFILENAME('DATA_PUMP_DIR', 'relatorelato-a.pdf'))
);
```

### Paso 3.8: Extraer texto, dividirlo y generar vectores

Este bloque hace tres acciones:

1. Extrae el texto del PDF.
2. Divide el texto en fragmentos de máximo 100 palabras.
3. Genera un vector para cada fragmento usando el modelo `ALL_MINILM_L6`.

```sql
INSERT INTO TB_VECTOR_DATA (vec_id, embed_id, embed_data, embed_vector)
SELECT id, embed_id, text_chunk, embed_vector
FROM TB_DATA dt
CROSS JOIN TABLE(
  DBMS_VECTOR_CHAIN.UTL_TO_EMBEDDINGS(
    DBMS_VECTOR_CHAIN.UTL_TO_CHUNKS(
      DBMS_VECTOR_CHAIN.UTL_TO_TEXT(dt.file_content),
      JSON('{"by":"words","max":"100","split":"sentence","normalize":"all"}')
    ),
    JSON('{"provider":"database","model":"ALL_MINILM_L6"}')
  )
) t
CROSS JOIN JSON_TABLE(
  t.column_value,
  '$[*]' COLUMNS (
    embed_id     NUMBER        PATH '$.embed_id',
    text_chunk   VARCHAR2(4000) PATH '$.embed_data',
    embed_vector CLOB          PATH '$.embed_vector'
  )
) et
WHERE dt.file_name = 'relatorelato-a.pdf';
```

**Resultado esperado:** `TB_VECTOR_DATA` contiene fragmentos del PDF y sus vectores.

### Paso 3.9: Probar búsqueda semántica

Ejecuta esta consulta para buscar los fragmentos más cercanos a una pregunta.

```sql
SELECT embed_data
FROM TB_VECTOR_DATA,
     (
       SELECT VECTOR_EMBEDDING(
                ALL_MINILM_L6 USING 'Qué le sucedió a la familia Gómez Ramírez' AS data
              ) AS embedding
     ) query_vector
ORDER BY VECTOR_DISTANCE(embed_vector, query_vector.embedding, COSINE)
FETCH APPROX FIRST 4 ROWS ONLY;
```

**Resultado esperado:** la consulta devuelve los fragmentos del PDF más relacionados con la pregunta.

## Paso 4: Crear una función RAG

La siguiente función automatiza el flujo RAG:

1. Recibe una pregunta del usuario.
2. Busca los fragmentos más relevantes en `TB_VECTOR_DATA`.
3. Construye un contexto con esos fragmentos.
4. Envía el contexto y la pregunta a Oracle Generative AI.
5. Devuelve una respuesta en lenguaje natural.

<u>Antes de ejecutar:</u> revisa de nuevo `p_region` y `p_compartment_ocid`.

- Si estás usando la región Chicago, puedes dejar `p_region := 'us-chicago-1';`.
- Si estás usando otra región, cambia `'us-chicago-1'` por el **Region Identifier** correspondiente. Puedes consultarlo en [Regions and Availability Domains](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm).
- Cambia `'<COMPARTMENT_OCID>'` por el OCID real de tu compartment, manteniendo las comillas simples.

```sql
CREATE OR REPLACE EDITIONABLE FUNCTION ADMIN.GENERATE_TEXT_RESPONSE_GEN (
  p_user_question VARCHAR2
) RETURN CLOB IS

  messages    CLOB := EMPTY_CLOB();
  output_text CLOB := EMPTY_CLOB();

  p_region           VARCHAR2(200) := 'us-chicago-1';
  p_endpoint         VARCHAR2(200) := 'https://inference.generativeai.' || p_region || '.oci.oraclecloud.com';
  p_compartment_ocid VARCHAR2(200) := '<COMPARTMENT_OCID>';

  resp          DBMS_CLOUD_TYPES.RESP;
  json_response CLOB;

  l_root    JSON_OBJECT_T;
  l_chat    JSON_OBJECT_T;
  l_choices JSON_ARRAY_T;
  l_choice0 JSON_OBJECT_T;
  l_message JSON_OBJECT_T;
  l_content JSON_ARRAY_T;
  l_part0   JSON_OBJECT_T;

  l_text VARCHAR2(32767);

BEGIN
  FOR message_cursor IN (
    SELECT embed_data
    FROM TB_VECTOR_DATA,
         (
           SELECT VECTOR_EMBEDDING(
                    ALL_MINILM_L6 USING p_user_question AS data
                  ) AS embedding
         ) query_vector
    ORDER BY VECTOR_DISTANCE(embed_vector, query_vector.embedding, COSINE)
    FETCH APPROX FIRST 4 ROWS ONLY
  ) LOOP
    messages := messages || '- ' || message_cursor.embed_data || CHR(10) || CHR(10);
  END LOOP;

  messages := messages
              || 'Pregunta del usuario:' || CHR(10)
              || p_user_question;

  resp := DBMS_CLOUD.SEND_REQUEST(
    credential_name => 'OCI_CRED',
    uri             => p_endpoint || '/20231130/actions/chat',
    method          => 'POST',
    body            => UTL_RAW.CAST_TO_RAW(
      JSON_OBJECT(
        'compartmentId' VALUE p_compartment_ocid,
        'servingMode' VALUE JSON_OBJECT(
          'modelId' VALUE 'meta.llama-4-maverick-17b-128e-instruct-fp8',
          'servingType' VALUE 'ON_DEMAND'
        ),
        'chatRequest' VALUE JSON_OBJECT(
          'messages' VALUE JSON_ARRAY(
            JSON_OBJECT(
              'role' VALUE 'USER',
              'content' VALUE JSON_ARRAY(
                JSON_OBJECT(
                  'type' VALUE 'TEXT',
                  'text' VALUE messages
                )
              )
            )
          ),
          'apiFormat' VALUE 'GENERIC',
          'maxTokens' VALUE 4000,
          'isStream' VALUE FALSE,
          'numGenerations' VALUE 1,
          'frequencyPenalty' VALUE 0,
          'presencePenalty' VALUE 0,
          'temperature' VALUE 1,
          'topP' VALUE 1.0,
          'topK' VALUE 1
        )
      )
    )
  );

  json_response := DBMS_CLOUD.GET_RESPONSE_TEXT(resp);

  l_root    := JSON_OBJECT_T.PARSE(json_response);
  l_chat    := l_root.GET_OBJECT('chatResponse');
  l_choices := l_chat.GET_ARRAY('choices');

  IF l_choices.GET_SIZE = 0 THEN
    RETURN TO_CLOB('ERROR: el modelo no devolvió opciones. Respuesta completa: ') || json_response;
  END IF;

  l_choice0 := TREAT(l_choices.GET(0) AS JSON_OBJECT_T);
  l_message := l_choice0.GET_OBJECT('message');
  l_content := l_message.GET_ARRAY('content');

  IF l_content.GET_SIZE = 0 THEN
    RETURN TO_CLOB('ERROR: el modelo no devolvió contenido. Respuesta completa: ') || json_response;
  END IF;

  l_part0     := TREAT(l_content.GET(0) AS JSON_OBJECT_T);
  l_text      := l_part0.GET_STRING('text');
  output_text := TO_CLOB(l_text);

  RETURN output_text;
END;
/
```

## Paso 5: Probar el RAG

Ejecuta la función con una pregunta sobre el PDF:

```sql
SELECT generate_text_response_gen('Qué le sucedió a la familia Gómez Ramírez')
FROM dual;
```

**Resultado esperado:** la base de datos devuelve una respuesta generada a partir de los fragmentos recuperados del documento.

## Lista de verificación final

Antes de cerrar el laboratorio, confirma que completaste estos puntos:

- La base de datos autónoma está en estado **Available**.
- La credencial `OCI_CRED` fue creada correctamente.
- El usuario `ADMIN` tiene permiso para hacer llamadas de red.
- Las tablas `TB_DATA` y `TB_VECTOR_DATA` existen.
- El PDF fue cargado en `TB_DATA`.
- El modelo `ALL_MINILM_L6` fue cargado en la base de datos.
- La tabla `TB_VECTOR_DATA` contiene fragmentos vectorizados.
- La búsqueda semántica devuelve resultados relevantes.
- La función `GENERATE_TEXT_RESPONSE_GEN` responde preguntas usando el contenido del documento.

## Errores comunes

Si algo falla, revisa primero estos puntos:

| Síntoma | Posible causa | Qué revisar |
| --- | --- | --- |
| Error al crear `OCI_CRED` | OCID, fingerprint o llave privada incorrectos | Revisa la API key y copia la llave PEM completa |
| Error al llamar a Generative AI | Falta permiso de red o política de OCI | Revisa el paso 2.1 y la política del compartment |
| No se descarga el PDF o el modelo | La credencial no tiene acceso a Object Storage | Valida `OCI_CRED` y el `object_uri` |
| La búsqueda semántica no devuelve resultados | No se insertaron vectores | Verifica que `TB_VECTOR_DATA` tenga registros |
| La función devuelve error JSON | El modelo no respondió con el formato esperado | Revisa la respuesta completa devuelta por `DBMS_CLOUD.GET_RESPONSE_TEXT` |

Si necesitas credenciales de prueba para una sesión guiada, solicítalas al instructor del laboratorio y úsalas solo en el entorno indicado.
