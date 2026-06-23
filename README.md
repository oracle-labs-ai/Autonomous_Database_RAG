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

# Búsqueda semántica (RAG) — pregunta → vector → distancia

```mermaid
flowchart LR
    A[">¿Qué le sucedió a la<br/>familia Gómez Ramirez?"] -->|embeddings 🕶️| B["[0.32, 0.12, 4.24, ..., 0.75]"]
 
    B --> C1
    B --> C2
    B --> C3
 
    subgraph KB["base de conocimiento"]
        direction TB
        C1["[0.12, 1.23, 2.24, ..., 1.26]"]
        C2["[0.32, 0.12, 4.24, ..., 0.75]"]
        C3["[0.98, 1.23, 1.24, ..., 0.46]"]
    end
 
    C1 -.->|distancia 0.34| D1["La familia Gómez Ramírez<br/>perdió su hogar en el incendio"]
    C2 -.->|distancia 0.64| D2["El tren partió rumbo al sur<br/>antes del amanecer"]
    C3 -.->|distancia 0.45| D3["Los vecinos evacuaron tras<br/>el aviso de tormenta"]
 
    style A fill:#C8DFE0,stroke:#2B5D5A,color:#1A3A38
    style B fill:#DDD8EC,stroke:#2B5D5A,color:#1A3A38
    style C1 fill:#DDD8EC,stroke:#6A4A7A,color:#3A2A4A
    style C2 fill:#DDD8EC,stroke:#6A4A7A,color:#3A2A4A
    style C3 fill:#DDD8EC,stroke:#6A4A7A,color:#3A2A4A
    style D1 fill:#C8DFE0,stroke:#1A3A38,color:#1A3A38
    style D2 fill:#C8DFE0,stroke:#4A2A5A,color:#1A3A38
    style D3 fill:#C8DFE0,stroke:#2A4A1A,color:#1A3A38
```

## Requisitos

Para poder ejecutar este laboratorio es necesario contar con los siguientes prerrequisitos:

1. **Políticas** para autorizar el consumo de servicios de IA desde la base de datos.

    > Si no existe, podemos crear las políticas necesarias siguiendo [este tutorial](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Creaci%C3%B3n%20de%20una%20pol%C3%ADtica.md)

2. **Un compartment** que permitirá agrupar los recursos creados para el laboratorio.

    > Si no tienes un compartment, puedes crearlo siguiendo [este tutorial](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Creaci%C3%B3n%20de%20un%20compartment.md).

3. **Base de datos Oracle con capacidades de IA**
    
    Es necesario contar con una base de datos **Oracle Database 23ai o 26ai** que funcionará como base de conocimiento, puede ser de tipo:
    
    - Transaction Processing
    - Data Warehouse

    > Si no tienes una base de datos, puedes crear una base de datos autónoma 26ai siguiendo [este tutorial](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Crear%20base%20de%20datos%20aut%C3%B3noma.md).
    
4. **Credenciales para acceso a servicios de IA**
    
    Es importante contar con una API key creada en la cuenta de OCI. Esta credencial permite realizar solicitudes desde la base de datos a los servicios de inteligencia artificial.
    
    La configuración del API Key se ve de la siguiente manera.

    ```docker
    [DEFAULT]
    user=ocid1.user.oc1..
    fingerprint=24:
    tenancy=ocid1.tenancy.
    region=us-ashburn-1
    ```

    > Si no tienes credenciales, puedes crearlas y descargarlas siguiendo [este tutorial](https://github.com/oracle-labs-ai/oci-console-tutorials/blob/main/Creaci%C3%B3n%20de%20credenciales.md).

5. **URLs del PDF y del modelo ONNX**

    El script actualizado usa URLs preautenticadas de Object Storage en la región `us-ashburn-1`. Estos enlaces permiten descargar el PDF y el modelo ONNX necesarios para el laboratorio.
    

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

- `p_region`: este laboratorio usa Ashburn por defecto, por eso el script trae `'us-ashburn-1'`. Si tu base de datos está en otra región, reemplaza ese valor por el identificador correcto. Para encontrarlo, entra a la guía oficial [Regions and Availability Domains](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm) y usa el valor de la columna **Region Identifier**. Por ejemplo, Bogotá aparece como `sa-bogota-1`.
- `p_compartment_ocid`: reemplaza el texto `'<COMPARTMENT_OCID>'` completo por el OCID real de tu compartment. El valor final debe quedar entre comillas simples, por ejemplo `'ocid1.compartment.oc1..aaaaaaaa...'`.

En otras palabras, los textos con formato `<...>` son marcadores de posición. Debes borrarlos y escribir tu valor real.

```sql
SET SERVEROUTPUT ON

DECLARE
  p_region           VARCHAR2(200) := 'us-ashburn-1';
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

La URL ya está incluida en el script para que puedas descargar el archivo del laboratorio directamente desde Object Storage.

```sql
BEGIN
  DBMS_CLOUD.GET_OBJECT(
    credential_name => 'OCI_CRED',
    object_uri      => 'https://objectstorage.us-ashburn-1.oraclecloud.com/p/NxcJA0hgEKQV7ndrzs1HqdlgjcPLURYS2sdt53TyDkKaiaBwylzhGAmqwQoXjtP5/n/idkrkto5fyu8/b/relato/o/relatorelato-a.pdf',
    directory_name  => 'DATA_PUMP_DIR'
  );
END;
/
```

### Paso 3.5: Copiar el modelo ONNX

Oracle Database puede ejecutar modelos de machine learning dentro de la base de datos. En este laboratorio se usará un modelo de vectorización previamente entrenado para convertir texto en embeddings.

El flujo es el siguiente: primero se copia el archivo `.onnx` a un directorio de Oracle, luego se registra con `LOAD_ONNX_MODEL()` y finalmente se invoca desde SQL.

![Flujo para registrar e invocar un modelo ONNX en Oracle Database](images/onnx_model_flow.svg)

La URL ya está incluida en el script para que puedas descargar el modelo del laboratorio directamente desde Object Storage.

```sql
BEGIN
  DBMS_CLOUD.GET_OBJECT(
    credential_name => 'OCI_CRED',
    object_uri      => 'https://objectstorage.us-ashburn-1.oraclecloud.com/p/mfk5C5kdG5b787Y9-XYsCHhTOEt3oIEeHhK-8A7K5F1E9N2gq1vNehuMpIhj-WjF/n/idkrkto5fyu8/b/relato/o/modeall-MiniLM-L6.onnx',
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

- Si estás usando la región Ashburn, puedes dejar `p_region := 'us-ashburn-1';`.
- Si estás usando otra región, cambia `'us-ashburn-1'` por el **Region Identifier** correspondiente. Puedes consultarlo en [Regions and Availability Domains](https://docs.oracle.com/en-us/iaas/Content/General/Concepts/regions.htm).
- Cambia `'<COMPARTMENT_OCID>'` por el OCID real de tu compartment, manteniendo las comillas simples.

```sql
CREATE OR REPLACE EDITIONABLE FUNCTION GENERATE_TEXT_RESPONSE_GEN (
  p_user_question VARCHAR2
) RETURN CLOB IS

  messages    CLOB := EMPTY_CLOB();
  output_text CLOB := EMPTY_CLOB();

  p_region           VARCHAR2(200) := 'us-ashburn-1';
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
| No se descarga el PDF o el modelo | URL vencida, región distinta o acceso al objeto no disponible | Valida que la URL de Object Storage abra en el navegador y que corresponda a `us-ashburn-1` |
| La búsqueda semántica no devuelve resultados | No se insertaron vectores | Verifica que `TB_VECTOR_DATA` tenga registros |
| La función devuelve error JSON | El modelo no respondió con el formato esperado | Revisa la respuesta completa devuelta por `DBMS_CLOUD.GET_RESPONSE_TEXT` |

Si necesitas credenciales de prueba para una sesión guiada, solicítalas al instructor del laboratorio y úsalas solo en el entorno indicado.
