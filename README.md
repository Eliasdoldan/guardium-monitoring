# IBM Guardium - Monitoreo de Agentes S-TAP vía Syslog, Logstash, Elasticsearch y Grafana

> **Estado actual:** productivo / funcional.  
> **Arquitectura vigente:** IBM Guardium envía reportes por **syslog TCP** hacia `pl-collect` (`10.10.18.117:5514`), Logstash parsea los eventos, Elasticsearch guarda estado actual e histórico, y Grafana muestra el dashboard operativo/ejecutivo.  
> **Fuera de foco:** el prototipo anterior por correo/CSV/SQLite/Grafana viejo queda discontinuado para este monitoreo.

---

## 1. Objetivo del sistema

Este monitoreo busca responder dos preguntas simples, pero críticas:

1. **¿El agente Guardium S-TAP está vivo y responde?**  
   Se mide con el reporte `S-TAP Status` y se grafica como minutos desde la última respuesta.

2. **¿El agente realmente está verificando correctamente el monitoreo del motor de base de datos?**  
   Se mide con el reporte `S-TAP Verification` y se grafica como `PASS`, `VERIFY VIEJO` o `FAIL`.

La segunda pregunta es la clave. Un agente puede aparecer como `Active`, pero tener el `Inspection Engine Verification` en `Fail`. En ese caso el agente responde, pero puede no estar capturando o verificando correctamente el tráfico.

### 1.1 Contexto del incidente que motivó este monitoreo

Este monitoreo no se implementó solamente por estética o por tener un dashboard adicional. Se implementó por un caso real observado en producción.

En una ocasión, los agentes asociados a **Medivh** perdieron comunicación con el colector principal. Cuando un S-TAP pierde comunicación con su colector, puede seguir acumulando datos en buffer para volcarlos cuando la conexión vuelva. Esa lógica es útil para no perder auditoría ante cortes breves, pero se vuelve peligrosa si la conexión no se restablece.

El problema observado fue:

```text
Agente pierde comunicación con el collector
→ el agente sigue acumulando datos en buffer
→ la conexión no vuelve
→ el buffer se llena
→ el S-TAP empieza a consumir muchos recursos en el servidor de base de datos
→ la base/core puede quedar impactada operativamente
```

Por eso el primer panel del dashboard, **minutos desde última respuesta**, es importante: confirma si el agente sigue respondiendo al colector. Si ese valor empieza a subir, especialmente en agentes críticos como Medivh, no es solamente un tema de observabilidad; puede convertirse en un problema de recursos en el servidor de base de datos si el buffer empieza a crecer.

Sin embargo, que un agente esté vivo no alcanza. También se observó que un agente puede estar `Active` y responder, pero no estar verificando correctamente el tráfico de base de datos. Por eso existe el segundo panel, **Estado de Verificación S-TAP**, que valida si el Inspection Engine está en `Pass`, `Fail` o si el último verify quedó viejo.

En resumen, se separaron dos controles porque cubren riesgos distintos:

```text
Panel 1 - S-TAP Status:
¿El agente responde al collector?
Sirve para detectar pérdida de comunicación y riesgo de buffer/consumo de recursos.

Panel 2 - S-TAP Verification:
¿El agente realmente verifica/captura tráfico del motor de base de datos?
Sirve para detectar falsos verdes: agente vivo, pero monitoreo real comprometido.
```

---

## 2. Arquitectura general

```text
IBM Guardium Collectors
    |
    | Syslog TCP hacia 10.10.18.117:5514
    v
pl-collect.bfamiliar.com.py
    |
    | Logstash pipeline: guardium_syslog
    | Config: /etc/logstash/pipelines/guardium/guardium-syslog.conf
    v
Elasticsearch local
    |
    | Índices current + history
    v
Grafana
    |
    | Dashboard ejecutivo / NOC
    v
Monitoreo de agentes Guardium S-TAP
```

Servidor principal:

```text
Hostname: pl-collect.bfamiliar.com.py
IP principal: 10.10.18.117/24
Timezone local: America/Asuncion (-03)
@timestamp en Elasticsearch: UTC
```

Servicios:

```text
Logstash      → recibe syslog y parsea eventos Guardium
Elasticsearch → almacena índices current e históricos
Grafana       → visualiza los datos
```

Puertos observados:

```text
5514/tcp  → entrada syslog Guardium hacia Logstash
5514/udp  → también escuchando, aunque el foco actual es TCP
9200/tcp  → Elasticsearch
3000/tcp  → Grafana
5044/tcp  → pipeline main / Beats, no es el foco de S-TAP
9600/tcp  → API local de Logstash
```

---

## 3. Componentes principales

### 3.1 Guardium

Guardium ejecuta Audit Processes / reportes y los envía por syslog hacia `10.10.18.117:5514`.

Reportes principales usados:

```text
S-TAP Status
S-TAP Verification
```

Además pueden llegar otros eventos por syslog, como pruebas `guardium_test_daemon.alert`, `guardium_test_user.info` u otros reportes, pero el foco de este dashboard son los dos reportes anteriores.

### 3.2 Logstash

Pipeline principal:

```text
pipeline.id: guardium_syslog
path.config: /etc/logstash/pipelines/guardium/guardium-syslog.conf
```

Entrada:

```text
syslog 0.0.0.0:5514
```

Salida:

```text
Archivo debug JSON:
/var/log/logstash/guardium-syslog-debug.log

Elasticsearch:
guardium-stap-current
guardium-stap-history-YYYY.MM.dd
guardium-stap-verification-current
guardium-stap-verification-history-YYYY.MM.dd
```

### 3.3 Elasticsearch

Guarda dos tipos de índices:

```text
current → estado actual, se sobreescribe por document_id
history → histórico diario, guarda cada evento recibido
```

### 3.4 Grafana

Dashboard final:

```text
Panel izquierdo: minutos desde última respuesta de agentes Guardium
Panel derecho: Estado de Verificación de agentes
```

---

## 4. Reporte S-TAP Status

Este reporte indica si el S-TAP responde y cuándo fue su última respuesta.

Encabezado crudo:

```csv
"S-TAP/UC Host","S-TAP/UC Version","DB Server Type","Status","Last Response","Primary Host Name","KTAP Installed","TEE Installed","Shared Memory Driver Installed","DB2 Shared Memory Driver Installed","LHMON Driver Installed","Named Pipes Driver Installed","Hunter DBS","App Server Installed","Encrypted?","Guardium Hosts","DB Exec File"
```

Ejemplo:

```csv
"10.10.10.90","STAP-12.1.1.2_r119073_v12_1_1-20250221_2348","oracle","Active","2026-06-18 10:57:59","10.10.21.89","Yes","No","No","No","Yes","No","","No","Unencrypted","10.10.21.89","/oracle/app/product/19.3.0/db_1/bin/oracle"
```

Campos parseados importantes:

```text
guardium.stap.host
guardium.stap.label
guardium.stap.status
guardium.stap.last_response
guardium.stap.last_response_ts
guardium.stap.age_minutes
```

Uso en Grafana:

```text
Datasource: guardium-graf
Índice: guardium-stap-current
Visualización: gauges
Métrica: guardium.stap.age_minutes
Agrupación: guardium.stap.label.keyword
```

Interpretación:

```text
0 a 10 min   → normal
10 a 20 min  → atención
más de 20 min → crítico / posible agente sin respuesta
```

---

## 5. Reporte S-TAP Verification

Este reporte indica si el Inspection Engine está verificando correctamente.

Encabezado crudo:

```csv
"S-TAP Host","S-TAP Version","DB Server Type","Inspection Engine Identifier","Port Range","Last Response from S-TAP","Inspection Engine Status","Last Verification Time","Verification Scheduled","Next Scheduled Time","Datasource Name","Datasource Description","Verification Type","Instance Name","KTAP","TEE","MSS Shm","Win DB2 Shm","Win TCP","Pipes","Encrypted?","Firewall Installed","DB Install Dir","Load Balancing","Alternate IPs","TLS","DB Exec File"
```

Ejemplo:

```csv
"10.10.10.90","STAP-12.1.1.2_r119073_v12_1_1-20250221_2348","oracle","oracle_10.10.10.90(1521,1521,DB_0)","1521-1521","Active","Pass","2026-06-18 10:00:58","Yes","2026-06-18 11:00:00","Default DataSource","Default datasource created for Stap Verification","Verifcation","","Yes","No","No","N/A","N/A","No","Unencrypted","No","/home/oracle","0","","Off","/oracle/app/product/19.3.0/db_1/bin/oracle"
```

Campos parseados importantes:

```text
guardium.stap_verification.host
guardium.stap_verification.label
guardium.stap_verification.inspection_engine_identifier
guardium.stap_verification.port_range
guardium.stap_verification.last_response_from_stap
guardium.stap_verification.inspection_engine_status
guardium.stap_verification.last_verification_time
guardium.stap_verification.last_verification_ts
guardium.stap_verification.next_scheduled_time
guardium.stap_verification.next_scheduled_ts
guardium.stap_verification.verification_age_minutes
guardium.stap_verification.verify_text
guardium.stap_verification.verify_code
guardium.stap_verification.health_text
guardium.stap_verification.health_code
```

Uso en Grafana:

```text
Datasource: guardium-verify
Índice: guardium-stap-verification-current
Visualización: tabla
Campo de estado: guardium.stap_verification.health_code
```

---

## 6. Diferencia entre Status y Verification

No son lo mismo y ambos son necesarios.

```text
S-TAP Status:
Indica si el agente responde.
Sirve para ver disponibilidad, última respuesta y riesgo de pérdida de comunicación con el collector.

S-TAP Verification:
Indica si el inspection engine está en Pass/Fail.
Sirve para detectar si Guardium realmente está verificando/capturando correctamente.
```

### 6.1 Por qué no alcanza con saber que el agente está Active

Un S-TAP puede estar `Active` y responder al collector, pero eso no garantiza por sí solo que el tráfico de base de datos esté siendo monitoreado correctamente. Por ese motivo el dashboard no se limita al estado del agente.

Caso crítico:

```text
S-TAP Status = Active
S-TAP Verification = Fail
```

Esto significa que el agente está vivo, pero el monitoreo real puede estar comprometido. En el dashboard esto debe verse como rojo, aunque el panel de minutos desde última respuesta esté verde.

### 6.2 Por qué el panel de última respuesta sigue siendo importante

El panel de minutos desde última respuesta no fue reemplazado por Verification. Cumple otro objetivo: detectar si el agente dejó de responder al collector. Si el agente no puede entregar datos, el buffer puede empezar a crecer, y si la situación se mantiene, el servidor de base de datos puede sufrir consumo alto de recursos.

Por eso la lectura correcta es:

```text
Status verde + Verification verde:
Estado normal. El agente responde y verifica correctamente.

Status rojo/alto + Verification desconocido/viejo:
Riesgo de comunicación/buffer. Revisar urgente, especialmente si es un agente crítico.

Status verde + Verification rojo:
El agente responde, pero no hay confianza en el monitoreo real. Revisar Inspection Engine.
```

---

## 7. Alias de agentes

El parser de Logstash convierte IPs en nombres legibles.

```text
10.10.10.105 → Cromi 10.105
10.10.10.57  → orcl-dboc01 10.57
10.10.10.66  → DWH 10.66
10.10.10.70  → alleria 10.70
10.10.10.90  → Quimera 10.90
10.10.15.43  → DWH2 15.43
10.10.10.92  → Medivh 10.92
```

Estos alias se aplican a:

```text
guardium.stap.label
guardium.stap_verification.label
```

Si se agrega un agente nuevo, agregarlo en ambos bloques del archivo:

```text
/etc/logstash/pipelines/guardium/guardium-syslog.conf
```

---

## 8. Lógica de health_code

El estado `0 / 1 / 2` se calcula en **Logstash**. Grafana solo lo muestra y lo pinta.

Regla actual:

```text
Si Inspection Engine Status = Fail:
    health_text = FAIL
    health_code = 2

Si Inspection Engine Status = Pass y verification_age_minutes > 7200:
    health_text = VERIFY VIEJO
    health_code = 1

Si Inspection Engine Status = Pass y verification_age_minutes <= 7200:
    health_text = PASS
    health_code = 0

Cualquier otro valor:
    health_text = UNKNOWN
    health_code = 1
```

Tabla:

```text
0 → PASS
1 → VERIFY VIEJO / UNKNOWN
2 → FAIL
```

Umbral actual de verify viejo:

```text
7200 minutos = 5 días
```

En Logstash está en esta condición:

```logstash
if [guardium][stap_verification][verification_age_minutes] and [guardium][stap_verification][verification_age_minutes] > 7200 {
```

Para cambiar el umbral:

```bash
sudo cp /etc/logstash/pipelines/guardium/guardium-syslog.conf \
/etc/logstash/pipelines/guardium/guardium-syslog.conf.bak.$(date +%F_%H%M%S)

sudo sed -i 's/verification_age_minutes\] > 7200/verification_age_minutes] > 1440/' \
/etc/logstash/pipelines/guardium/guardium-syslog.conf

sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
sudo systemctl restart logstash
```

Referencias de minutos:

```text
12 horas = 720 minutos
1 día    = 1440 minutos
2 días   = 2880 minutos
5 días   = 7200 minutos
```

---

## 9. Índices de Elasticsearch

Current:

```text
guardium-stap-current
guardium-stap-verification-current
```

History:

```text
guardium-stap-history-YYYY.MM.dd
guardium-stap-verification-history-YYYY.MM.dd
```

Ver índices:

```bash
curl -k -u "elastic:${ES_PWD}" -s 'https://localhost:9200/_cat/indices/guardium-stap*?v'
```

Interpretación:

```text
guardium-stap-current → 7 documentos, uno por agente
guardium-stap-verification-current → 9 documentos, uno por inspection engine
```

Actualmente hay 7 agentes, pero 9 inspection engines porque `alleria 10.70` tiene 3 engines.

---

## 10. Comandos para ver qué llega crudo

Ver todo lo que entra por 5514:

```bash
sudo timeout 90 tcpdump -ni any -s0 -A '(tcp port 5514 or udp port 5514)' \
| egrep -i 'STAP|S-TAP|Verification|Pass|Fail|Guardium|auditprocess'
```

Filtrar por collectors conocidos:

```bash
sudo timeout 90 tcpdump -ni any -s0 -A '(tcp port 5514 or udp port 5514) and (host 10.10.21.89 or host 10.10.21.86)' \
| egrep -i 'STAP|S-TAP|Verification|Pass|Fail|Guardium|auditprocess'
```

Collectors observados:

```text
GuardCol01.bfamiliar.com.py
GuardCol02.bfamiliar.com.py
10.10.21.86
10.10.21.89
```

---

## 11. Comandos para ver qué parseó Logstash

Archivo debug:

```text
/var/log/logstash/guardium-syslog-debug.log
```

Ver últimos eventos JSON:

```bash
sudo tail -n 5 /var/log/logstash/guardium-syslog-debug.log | jq .
```

Ver Status parseado:

```bash
sudo jq -r '
select(.guardium.data_type? == "stap_status")
| "STATUS time=\(. ["@timestamp"]) collector=\(.host.hostname) host=\(.guardium.stap.host) label=\(.guardium.stap.label) status=\(.guardium.stap.status) last=\(.guardium.stap.last_response) age=\(.guardium.stap.age_minutes)"
' /var/log/logstash/guardium-syslog-debug.log | tail -n 20
```

Ver Verification parseado:

```bash
sudo jq -r '
select(.guardium.data_type? == "stap_verification")
| "VERIFY time=\(. ["@timestamp"]) collector=\(.host.hostname) host=\(.guardium.stap_verification.host) label=\(.guardium.stap_verification.label) engine=\(.guardium.stap_verification.inspection_engine_identifier) verify=\(.guardium.stap_verification.verify_text) health=\(.guardium.stap_verification.health_text) code=\(.guardium.stap_verification.health_code) age=\(.guardium.stap_verification.verification_age_minutes)"
' /var/log/logstash/guardium-syslog-debug.log | tail -n 30
```

Ver solo FAIL:

```bash
sudo jq -r '
select(.guardium.data_type? == "stap_verification")
| select(.guardium.stap_verification.health_code? == 2)
| "FAIL time=\(. ["@timestamp"]) collector=\(.host.hostname) host=\(.guardium.stap_verification.label) engine=\(.guardium.stap_verification.inspection_engine_identifier) last=\(.guardium.stap_verification.last_verification_time)"
' /var/log/logstash/guardium-syslog-debug.log | tail -n 30
```

> Nota: en algunos shells puede ser necesario quitar el espacio entre `. [` y `.[`. Se dejó separado acá solo para evitar problemas de render en Markdown.

---

## 12. Consultas útiles en Elasticsearch

El password de Elasticsearch se carga en `ES_PWD` desde `~/.es_pwd` y `~/.bashrc`.

Verificar variable:

```bash
echo ${#ES_PWD}
source ~/.bashrc
echo ${#ES_PWD}
```

No escribir el password real en documentación ni tickets.

Ver Status actual:

```bash
curl -k -u "elastic:${ES_PWD}" -s 'https://localhost:9200/guardium-stap-current/_search?pretty' \
-H 'Content-Type: application/json' \
-d '{
  "size": 20,
  "sort": [ { "@timestamp": { "order": "desc" } } ],
  "_source": [
    "@timestamp",
    "host.hostname",
    "guardium.data_type",
    "guardium.stap.label",
    "guardium.stap.host",
    "guardium.stap.status",
    "guardium.stap.last_response",
    "guardium.stap.age_minutes"
  ]
}'
```

Ver Verification actual:

```bash
curl -k -u "elastic:${ES_PWD}" -s 'https://localhost:9200/guardium-stap-verification-current/_search?pretty' \
-H 'Content-Type: application/json' \
-d '{
  "size": 20,
  "sort": [ { "@timestamp": { "order": "desc" } } ],
  "_source": [
    "@timestamp",
    "host.hostname",
    "guardium.data_type",
    "guardium.stap_verification.label",
    "guardium.stap_verification.host",
    "guardium.stap_verification.inspection_engine_identifier",
    "guardium.stap_verification.inspection_engine_status",
    "guardium.stap_verification.verify_text",
    "guardium.stap_verification.health_text",
    "guardium.stap_verification.health_code",
    "guardium.stap_verification.verification_age_minutes",
    "guardium.stap_verification.last_verification_time",
    "guardium.stap_verification.next_scheduled_time"
  ]
}'
```

Ver mapping de `health_code`:

```bash
curl -k -u "elastic:${ES_PWD}" -s \
'https://localhost:9200/guardium-stap-verification-current/_mapping/field/guardium.stap_verification.health_code?pretty'
```

Ver mapping de label:

```bash
curl -k -u "elastic:${ES_PWD}" -s \
'https://localhost:9200/guardium-stap-verification-current/_mapping/field/guardium.stap_verification.label*?pretty'
```

---

## 13. Dashboard Grafana

### 13.1 Datasources

```text
guardium-graf   → guardium-stap-current
guardium-verify → guardium-stap-verification-current
```

### 13.2 Panel: Minutos desde última respuesta

Tipo:

```text
Gauge
```

Datasource:

```text
guardium-graf
```

Métrica:

```text
Max guardium.stap.age_minutes
```

Agrupación:

```text
Terms guardium.stap.label.keyword
```

Objetivo:

```text
Ver rápidamente si un agente dejó de responder.
```

### 13.3 Panel: Estado de Verificación de agentes

Tipo:

```text
Table
```

Datasource:

```text
guardium-verify
```

Query:

```text
guardium.data_type:stap_verification
```

Modo recomendado:

```text
Raw Data
Size: 50
```

Transformations:

1. Filter fields by name:

```text
guardium.stap_verification.label
guardium.stap_verification.health_code
guardium.stap_verification.verification_age_minutes
```

2. Group by:

```text
guardium.stap_verification.label → Group by
guardium.stap_verification.health_code → Calculate Max
guardium.stap_verification.verification_age_minutes → Calculate Max
```

3. Organize fields:

```text
guardium.stap_verification.label → AGENTE
guardium.stap_verification.health_code (max) → ESTADO
guardium.stap_verification.verification_age_minutes (max) → Min sin Verify
```

4. Sort by:

```text
Field: ESTADO
Reverse: enabled
```

Value mappings:

```text
0 → PASS
1 → VERIFY VIEJO
2 → FAIL
```

Colores:

```text
PASS → verde
VERIFY VIEJO → amarillo
FAIL → rojo
```

Cell options:

```text
Cell type: Colored background
Apply to entire row: ON
```

Resultado esperado:

```text
Si todo está bien: filas verdes.
Si un verify está viejo: fila amarilla.
Si un inspection engine falla: fila roja.
```

---

## 14. Interpretación operativa

### Todo verde

```text
El agente responde y su verification está en PASS.
```

Interpretación:

```text
Comunicación con collector OK.
Inspection Engine en PASS.
No hay acción inmediata.
```

### Gauge verde, tabla roja

```text
El agente responde, pero el inspection engine está en FAIL.
Esto es crítico.
```

Interpretación:

```text
El agente está vivo, pero Guardium no tiene una verificación sana del monitoreo real.
Puede existir un falso verde: hay respuesta del agente, pero la captura/verificación está comprometida.
```

Acción recomendada:

```text
1. Entrar a Guardium.
2. Ir a S-TAP Verification / S-TAP Control del agente afectado.
3. Ejecutar Verify manual sobre el agente/inspection engine.
4. Esperar el próximo envío por syslog y confirmar si Grafana cambia a PASS.
5. Si sigue en FAIL, ejecutar restart del S-TAP desde Guardium siguiendo el procedimiento operativo aprobado.
6. Si después del restart sigue en FAIL, revisar S-TAP Events y logs del agente.
7. Si no se normaliza, contactar al proveedor/soporte IBM Guardium.
```

En la práctica observada, muchas veces un **Verify manual** corrige el estado si el problema era un verify viejo o desactualizado. Si eso no alcanza, el **restart del S-TAP** suele normalizar varios problemas de agente/inspection engine. Si ninguna de esas acciones corrige el estado, ya no conviene seguir improvisando: se debe escalar al proveedor.

### Gauge verde, tabla amarilla

```text
El agente responde, pero el último verify es viejo.
```

Interpretación:

```text
El agente está vivo, pero la verificación quedó vieja según el umbral configurado en Logstash.
Actualmente el umbral es 7200 minutos = 5 días.
```

Acción recomendada:

```text
1. Ejecutar Verify manual desde Guardium.
2. Confirmar que el dashboard cambie a PASS en Grafana.
3. Revisar si el agente tiene Verification Scheduled = Yes o No.
4. Si vuelve a quedar viejo, revisar schedule de Verification en Guardium.
5. Si no actualiza aunque el Verify se ejecuta, revisar que el reporte S-TAP Verification siga saliendo por syslog.
```

### Gauge rojo o con muchos minutos

```text
El agente no responde recientemente.
```

Interpretación:

```text
Puede haber pérdida de comunicación entre el S-TAP y el collector.
Si se mantiene, el agente puede acumular datos en buffer.
Si el buffer crece demasiado, puede impactar recursos del servidor de base de datos.
```

Acción recomendada:

```text
1. Revisar conectividad entre servidor del agente y collector.
2. Revisar S-TAP Control en Guardium.
3. Revisar S-TAP Events.
4. Validar el servicio/proceso del agente en el servidor monitoreado.
5. Si el agente no recupera comunicación, ejecutar restart del S-TAP siguiendo procedimiento aprobado.
6. Si sigue sin responder o vuelve a fallar, contactar al proveedor/soporte IBM Guardium.
```

### 14.1 Flujo rápido de respuesta ante problema

```text
Problema amarillo o rojo en Grafana
    ↓
Identificar agente afectado
    ↓
Entrar a Guardium
    ↓
Ejecutar Verify
    ↓
¿Se normaliza?
    ├── Sí → confirmar en Grafana y dejar registrado
    └── No
        ↓
        Ejecutar restart del S-TAP
        ↓
        ¿Se normaliza?
        ├── Sí → confirmar en Grafana y dejar registrado
        └── No → contactar proveedor / soporte IBM Guardium
```

---

## 15. Operación diaria

Ver servicios:

```bash
sudo systemctl status logstash --no-pager
sudo systemctl status elasticsearch --no-pager
sudo systemctl status grafana-server --no-pager
```

Ver puertos:

```bash
sudo ss -lntup | egrep '5514|9200|3000|logstash|grafana|java'
```

Ver syslog crudo:

```bash
sudo timeout 60 tcpdump -ni any -s0 -A '(tcp port 5514 or udp port 5514)' \
| egrep -i 'STAP|S-TAP|Verification|Pass|Fail|auditprocess'
```

Ver parseado Verification:

```bash
sudo jq -r '
select(.guardium.data_type? == "stap_verification")
| "VERIFY host=\(.guardium.stap_verification.label) verify=\(.guardium.stap_verification.verify_text) health=\(.guardium.stap_verification.health_text) code=\(.guardium.stap_verification.health_code) age=\(.guardium.stap_verification.verification_age_minutes)"
' /var/log/logstash/guardium-syslog-debug.log | tail -n 20
```

Ver índices:

```bash
curl -k -u "elastic:${ES_PWD}" -s 'https://localhost:9200/_cat/indices/guardium-stap*?v'
```

---

## 16. Reinicios

### Logstash

Validar antes:

```bash
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
```

Reiniciar:

```bash
sudo systemctl restart logstash
sudo journalctl -u logstash -n 150 --no-pager
```

Logs en vivo:

```bash
sudo journalctl -u logstash -f
```

### Grafana

```bash
sudo systemctl restart grafana-server
sudo journalctl -u grafana-server -n 150 --no-pager
```

### Elasticsearch

Usar con cuidado:

```bash
sudo systemctl restart elasticsearch
sudo journalctl -u elasticsearch -n 150 --no-pager
```

---

## 17. Backup y rollback de Logstash

Backup:

```bash
sudo cp /etc/logstash/pipelines/guardium/guardium-syslog.conf \
/etc/logstash/pipelines/guardium/guardium-syslog.conf.bak.$(date +%F_%H%M%S)
```

Ver backups:

```bash
sudo bash -c 'ls -1t /etc/logstash/pipelines/guardium/guardium-syslog.conf.bak.* 2>/dev/null | head -n 10'
```

Restaurar último backup:

```bash
BACKUP=$(sudo bash -c 'ls -1t /etc/logstash/pipelines/guardium/guardium-syslog.conf.bak.* 2>/dev/null | head -n 1')
echo "$BACKUP"
sudo cp "$BACKUP" /etc/logstash/pipelines/guardium/guardium-syslog.conf
sudo -u logstash /usr/share/logstash/bin/logstash --path.settings /etc/logstash -t
sudo systemctl restart logstash
```

---

## 18. Troubleshooting

### Grafana muestra No data

1. Probar `Raw Data`.
2. Confirmar datasource correcto.
3. Confirmar rango horario.
4. Confirmar que Elasticsearch tiene documentos.
5. Confirmar que el campo existe en mapping.

Comandos:

```bash
curl -k -u "elastic:${ES_PWD}" -s 'https://localhost:9200/_cat/indices/guardium-stap*?v'
```

```bash
curl -k -u "elastic:${ES_PWD}" -s 'https://localhost:9200/guardium-stap-verification-current/_search?pretty' \
-H 'Content-Type: application/json' \
-d '{ "size": 5, "sort": [ { "@timestamp": { "order": "desc" } } ] }'
```

### Logstash no parsea

Ver errores:

```bash
sudo journalctl -u logstash -n 200 --no-pager
```

Buscar failures en debug:

```bash
sudo grep -iE '_grok_failure|_csv|failure|stap_verification' /var/log/logstash/guardium-syslog-debug.log | tail -n 100
```

### No llega syslog

```bash
sudo ss -lntup | egrep '5514|logstash|java'
```

```bash
sudo timeout 90 tcpdump -ni any -s0 -A '(tcp port 5514 or udp port 5514)'
```

En Guardium revisar:

```text
Remote Syslog / Remote Logger
Destino 10.10.18.117:5514 TCP
Audit Process con Write to SYSLOG
```

---

## 19. Casos validados

### Caso VERIFY VIEJO

Se observó que `alleria 10.70` tenía engines con verify viejo. El dashboard lo pintó amarillo. Luego se ejecutó Verify y pasó a verde.

Esto validó:

```text
Guardium genera nuevo verify
Syslog llega a pl-collect
Logstash recalcula health_code
Elasticsearch actualiza current
Grafana cambia de amarillo a verde
```

### Caso FAIL histórico

Se observaron eventos `FAIL` históricos para `Medivh 10.92` en el archivo debug. Esto confirma que si vuelve a ocurrir, el dashboard puede marcarlo rojo.

---

## 20. Pendientes recomendados

1. Exportar el JSON del dashboard de Grafana y guardarlo como backup.
2. Crear panel secundario “Detalle de motores con problema” usando:

```text
guardium.data_type:stap_verification AND guardium.stap_verification.health_code:[1 TO 2]
```

3. Evaluar alerta automática si:

```text
health_code = 2
```

4. Definir si el umbral de `VERIFY VIEJO` queda en 5 días o baja para agentes críticos.
5. Documentar en Guardium la ubicación exacta del Audit Process que envía los reportes.

---

## 21. Resumen ejecutivo

El sistema actual reemplaza el prototipo anterior por correo/SQLite y deja un flujo más directo y operativo:

```text
Guardium → Syslog TCP → Logstash → Elasticsearch → Grafana
```

El dashboard permite ver:

```text
1. Agentes vivos / última respuesta.
2. Estado real de Verification del inspection engine.
```

El motivo de tener ambos controles es operativo:

```text
Última respuesta:
Detecta pérdida de comunicación con el collector y riesgo de buffer/consumo de recursos.

Verification:
Detecta si el agente realmente está verificando/capturando correctamente el tráfico.
```

Esto evita falsos verdes. Un agente puede responder, pero si su verification falla, el dashboard lo marca como problema. También permite reaccionar antes de que una pérdida de comunicación sostenida llene buffers y afecte recursos del servidor de base de datos.

Acción base ante problema:

```text
Verify manual → si no corrige, restart del S-TAP → si no corrige, proveedor/soporte IBM Guardium.
```

