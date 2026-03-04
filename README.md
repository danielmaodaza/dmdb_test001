# OWASP Top 10 · Seguridad en Entornos Cloud y DevSecOps

> **Estándar OWASP 2021** · Análisis técnico adaptado a pipelines CI/CD, infraestructura cloud y prácticas DevSecOps modernas.

---

## Índice

- [A05 · Injection](#a05--injection)
- [A06 · Insecure Design](#a06--insecure-design)
- [A09 · Security Logging & Alerting Failures](#a09--security-logging--alerting-failures)
- [Conclusiones Generales](#conclusiones-generales)
- [Referencias](#referencias)

---

## Resúmen

| OWASP ID | Vulnerabilidad | Nivel de Riesgo | Impacto Principal | Control Primario |
|---|---|---|---|---|
| **A05** | Injection | 🔴 Crítico | Brecha de datos, RCE | Consultas parametrizadas + SAST |
| **A06** | Insecure Design | 🟠 Alto | Compromiso sistémico | Threat modeling + SSDLC |
| **A09** | Logging & Alerting Failures | 🟡 Medio-Alto | Brechas indetectables | SIEM centralizado + alertas |

---

## A05 · Injection

> `SQL` · `NoSQL` · `OS Command` · `LDAP` · `SSTI` · `XXE`  
> ↳ **DevSecOps:** afecta contenedores, pipelines CI/CD y microservicios

---

### 1. Descripción de la Vulnerabilidad

**¿Qué es?**  
Clase de ataque donde datos no confiables son enviados a un intérprete como parte de un comando o consulta, que los ejecuta como instrucción legítima.

**¿Cómo funciona?**  
Sin sanitización adecuada, el atacante manipula el intérprete (DB, OS, LDAP) para alterar la lógica del comando, extrayendo datos o ejecutando código arbitrario.

**Impacto**  
Filtración masiva de datos, RCE, escalada de privilegios, destrucción de datos y compromiso total. CVSS frecuentemente **9.0–10.0**.

**Flujo de ataque:**
```
[INPUT NO CONFIABLE] → [SIN VALIDACIÓN] → [⚠ INTÉRPRETE] → [EXPLOTACIÓN] → [IMPACTO]
 form / URL / header    Concatena directo   DB / OS / LDAP   Extracción/RCE   Brecha total
                        No escapa chars     Ejecuta payload
                                            ↑ Punto crítico
```

---

### 2. Métodos de Explotación

**Técnicas de ataque:**
- Probar campos con `' OR '1'='1` para detectar respuestas booleanas o errores
- UNION-based / blind SQLi para extraer esquema y datos columna por columna
- Escalar a OS vía `xp_cmdshell`, stacked queries o stored procedures
- Inyectar en build args de Docker o variables de entorno del pipeline CI/CD
- Establecer persistencia con web shells o backdoors post-explotación

**Ejemplos reales:**

| Caso | Año | Vector | Impacto |
|---|---|---|---|
| Heartland Payment Systems | 2008 | SQLi | 130M+ registros de tarjetas |
| Equifax | 2017 | OGNL injection (Apache Struts) | 147M afectados |
| Tesla (sistema interno) | 2022 | SQLi | Datos de empleados y vehículos |
| SolarWinds | 2020 | Command injection en pipeline | Compromiso supply chain |

**Herramientas comunes:**

| Herramienta | Tipo | Uso en DevSecOps |
|---|---|---|
| `sqlmap` | Automático | Detección y explotación de SQLi — integrable en pipelines |
| `Burp Suite` | Manual/DAST | Intercepción HTTP, scanner activo, repetidor de requests |
| `OWASP ZAP` | DAST/CI | Escaneo dinámico nativo para pipelines CI/CD en staging |
| `Commix` | CMDi | OS command injection automatizado en endpoints web |
| `Semgrep` | SAST | Detección estática de patrones inseguros en cada PR |

---

### 3. Prevención y Mitigación

**Código seguro:**
- Consultas parametrizadas / prepared statements en *toda* interacción con DB
- ORMs (Hibernate, SQLAlchemy) que abstraen SQL raw
- Escapar caracteres especiales solo como último recurso

**Validación de entradas:**
- Whitelist server-side: tipo, longitud y formato estrictos
- Nunca confiar en datos del cliente — toda entrada es potencialmente hostil
- Separar código de datos en todos los intérpretes

**Infraestructura:**
- Mínimo privilegio en cuentas DB (sin `DROP`/`CREATE` para la app)
- WAF con reglas CRS de injection activas
- Deshabilitar: `xp_cmdshell`, `LOAD_FILE`

---

### 4. Buenas Prácticas en DevSecOps

- Integrar SAST (Semgrep, Checkmarx) en cada PR — bloquear merges con hallazgos críticos
- Ejecutar DAST (OWASP ZAP) automáticamente en pipelines de staging
- Auditar llamadas a intérpretes en code reviews como checklist obligatorio
- Rotar credenciales de DB con HashiCorp Vault o AWS Secrets Manager
- Nunca concatenar inputs de usuario en queries o comandos de shell
- Aplicar dependency scanning (Snyk, Dependabot) para libs con CVEs conocidos
- Usar policy-as-code (OPA, Conftest) para validar configuraciones de pipeline
- Capacitar developers en OWASP Secure Coding Practices cada trimestre

---

### 5. Configuraciones Recomendadas

**Prepared Statement · Java JDBC**
```java
// ❌ INSEGURO — concatenación directa
// "SELECT * FROM users WHERE id=" + id

// ✅ SEGURO — Prepared Statement
String sql = "SELECT * FROM users WHERE id = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setInt(1, userId); // Parámetro tipado
ResultSet rs = ps.executeQuery();
```

**WAF ModSecurity CRS + GitHub Actions DAST**
```apache
# modsecurity.conf
SecRuleEngine On
SecRequestBodyAccess On
Include 'coreruleset/crs-setup.conf'
Include 'coreruleset/rules/*.conf'
SecAction "setvar:tx.paranoia_level=2"
```

```yaml
# .github/workflows/dast.yml
- name: OWASP ZAP Full Scan
  uses: zaproxy/action-full-scan@v0.9
  with:
    target: 'https://staging.miapp.com'
```

---

### 6. Controles de Seguridad

| 🛡 Preventivos | 🔍 Detectivos | 🔧 Correctivos |
|---|---|---|
| Consultas parametrizadas / ORMs | Alertas WAF por patrones SQLi / CMDi | Parchear endpoints vulnerables inmediatamente |
| Validación whitelist de entradas | Monitoreo de anomalías en queries DB | Revocar y rotar credenciales DB |
| SAST en cada PR (quality gate) | DAST automático en pipelines staging | Análisis forense de logs de queries |
| WAF con CRS ruleset activo | SIEM con reglas para payloads comunes | Rollback a versión segura del deploy |
| Mínimo privilegio en cuentas DB | Database Activity Monitoring (DAM) | Plan de respuesta a incidentes documentado |

---

### 7. Conclusión

En entornos DevSecOps, Injection es especialmente peligroso porque los pipelines CI/CD, contenedores y microservicios amplían la superficie de ataque. La estrategia clave es el principio **shift-left**: SAST en cada PR, DAST en staging, y gestión de secrets con herramientas dedicadas. No hay excusa técnica para exponer intérpretes a entradas no sanitizadas.

---

## A06 · Insecure Design

> `Diseño inseguro` · `Lógica de negocio` · `Threat Modeling` · `SSDLC`  
> ↳ **DevSecOps:** requiere rediseño arquitectónico; no basta con parches

---

### 1. Descripción de la Vulnerabilidad

**¿Qué es?**  
Ausencia de controles de seguridad que *nunca fueron diseñados* en el sistema — distinto de un bug de implementación. Cubre lógica de negocio defectuosa y debilidades arquitectónicas estructurales.

**¿Cómo funciona?**  
Surge al omitir threat modeling o requisitos de seguridad del backlog. El sistema resultante es estructuralmente incapaz de resistir ciertas clases de ataque — no existe hotfix de una línea.

**Impacto**  
Bypass de autenticación, escalada de privilegios por lógica de negocio, IDOR masivo, exposición de datos. Requiere **rediseño completo** de componentes afectados.

**SSDLC — Con vs. Sin controles de seguridad:**
```
SIN SEC:  [Requisitos] → [Diseño ✗] → [Desarrollo] → [Deploy ✗] → ❌ Sistema vulnerable · rediseño
CON SEC:  [Requisitos] → [Diseño ✓] → [Desarrollo] → [Deploy ✓] → ✅ Sistema seguro por diseño
                 ↑ Threat Model        ↑ Security gates en cada fase
```
> El costo de corrección crece exponencialmente cuando no hay security gates desde el diseño.

---

### 2. Métodos de Explotación

**Técnicas de ataque:**
- Mapear flujos de trabajo y detectar operaciones sin control de acceso
- Saltar pasos en workflows multi-etapa (checkout sin pago real)
- Explotar ausencia de rate limiting para enumerar cuentas o hacer brute-force
- IDOR: aprovechar IDs secuenciales sin verificación de autorización
- Credential stuffing masivo por ausencia de anti-automatización

**Ejemplos reales:**

| Caso | Año | Vector | Impacto |
|---|---|---|---|
| Instagram | 2019 | Sin rate limiting en verificación | Enumeración de 6M usuarios |
| Parler | 2021 | IDs secuenciales sin auth | Extracción masiva de 70TB |
| Facebook | 2019 | Lookup inseguro por número de teléfono | 533M registros expuestos |
| Peloton API | 2021 | Endpoints sin autenticación por diseño | 4M usuarios públicos |

**Herramientas comunes:**

| Herramienta | Tipo | Uso en DevSecOps |
|---|---|---|
| `OWASP Threat Dragon` | Diseño | Modelado de amenazas visual — artefacto del sprint |
| `Burp Suite Pro` | Lógica | Testing de flujos de negocio y workflows multi-paso |
| `Semgrep` | SAST | Detección de anti-patrones de diseño inseguro |
| `Postman / Insomnia` | API | Testing de flujos API para detectar controles faltantes |
| `STRIDE / PASTA` | Framework | Metodologías de threat modeling arquitectónico |

---

### 3. Prevención y Mitigación

**Fase de diseño:**
- Threat modeling (STRIDE) obligatorio por cada feature nueva
- Requisitos de seguridad junto a los funcionales en user stories
- Fail-safe defaults y separación de privilegios desde el inicio

**Arquitectura:**
- Control de acceso por defecto negativo (default-deny)
- Trust boundaries explícitos en diagramas de arquitectura
- Zero-trust en microservicios con service mesh (Istio + mTLS)

**Proceso SSDLC:**
- Security gates en cada fase del pipeline CI/CD
- Criterios de aceptación de seguridad antes de cada sprint
- Red team exercises enfocados en lógica de negocio

---

### 4. Buenas Prácticas en DevSecOps

- Incorporar **security champions** en cada squad de desarrollo
- Mantener catálogo de patrones de diseño seguro aprobados (reference architectures)
- Documentar decisiones de seguridad en ADRs (Architecture Decision Records)
- Incluir **abuse cases** en el backlog junto a los user stories normales
- Usar DFDs para identificar trust boundaries antes de codificar
- Revisar IaC con Checkov o Terrascan en el pipeline
- Aplicar separación de tareas (four-eyes) para operaciones sensibles
- Design Security Reviews obligatorios antes de cada release mayor

---

### 5. Configuraciones Recomendadas

**Rate Limiting · Kong API Gateway**
```yaml
# kong.yml — protección por diseño
plugins:
  - name: rate-limiting
    config:
      minute: 60       # máx req/min/IP
      hour: 500
      policy: redis    # compartido entre pods
      limit_by: ip
      error_code: 429
  - name: bot-detection
    config:
      deny: ["curl", "python-requests"]
```

**Istio · Zero-Trust Service Mesh**
```yaml
# PeerAuthentication — mTLS estricto
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  namespace: production
spec:
  mtls:
    mode: STRICT    # Sin plaintext
---
# AuthorizationPolicy — default-deny
kind: AuthorizationPolicy
spec:
  action: DENY
  rules: [{}]       # Niega todo por defecto
```

---

### 6. Controles de Seguridad

| 🛡 Preventivos | 🔍 Detectivos | 🔧 Correctivos |
|---|---|---|
| Threat modeling obligatorio (STRIDE/PASTA) | Monitoreo de anomalías en flujos de negocio | Rediseño de componentes vulnerables |
| Requisitos de seguridad en user stories | Alertas por salteo de pasos en workflows | Implementar controles faltantes post-auditoría |
| Rate limiting y anti-automatización | Abuse case testing en QA y staging | Threat modeling retrospectivo |
| Arquitectura zero-trust con mTLS | API gateway metrics y alertas | Actualizar estándares de diseño del equipo |
| Security gate pre-desarrollo | Security design review pre-release | Bug bounty para lógica de negocio |

---

### 7. Conclusión

A diferencia de otros riesgos, Insecure Design no se resuelve con un parche. En DevSecOps exige un **cambio cultural**: el threat modeling debe ser un ritual antes de cada sprint, los criterios de seguridad deben vivir en el mismo backlog que los requisitos funcionales, y los **security champions** por squad son el mecanismo humano más efectivo para garantizar diseño seguro de forma continua.

---

## A09 · Security Logging & Alerting Failures

> `Logging` · `SIEM` · `Observabilidad` · `Incident Response` · `Compliance`  
> ↳ **DevSecOps:** pipelines sin auditoría = cadena de suministro ciega

---

### 1. Descripción de la Vulnerabilidad

**¿Qué es?**  
Falla en generar, proteger o actuar sobre logs de eventos de seguridad. Sin visibilidad adecuada, los ataques permanecen sin detectarse. El tiempo medio global de permanencia de un atacante **supera los 200 días**.

**¿Cómo funciona?**  
Omitir logging de autenticación, cambios de privilegios o errores de acceso da tiempo ilimitado al atacante. Sin SIEM ni alertas, las brechas persisten meses sin respuesta.

**Impacto**  
Brechas indetectables, imposibilidad de análisis forense, violaciones de cumplimiento (PCI-DSS, HIPAA, SOC 2) y daño reputacional por divulgación tardía.

**Flujo de detección — Con vs. Sin logging:**
```
SIN LOG:  [Ataque] → [Sin logs]      → [Sin SIEM]      → [Sin respuesta] → ❌ Brecha silenciosa (200+ días)
CON LOG:  [Ataque] → [Logging JSON]  → [SIEM + Alerta] → [Respuesta SOAR] → ✅ Contención en minutos
                      estructurado      umbral disparado   playbook automático
```
> La diferencia está en: logging estructurado + SIEM + alertas accionables + respuesta automatizada.

---

### 2. Métodos de Explotación

**Técnicas de ataque:**
- Ataques lentos y de bajo volumen para mantenerse bajo umbrales de detección
- Eliminar o sobrescribir archivos de log en hosts comprometidos
- Inundar logs con ruido para provocar rotación y borrar evidencias
- Operar en horarios de baja cobertura del SOC (madrugadas, fin de semana)
- En CI/CD: deshabilitar logging en pipelines modificados sin alertar al equipo

**Ejemplos reales:**

| Caso | Año | Vector | Impacto |
|---|---|---|---|
| Uber | 2016 | Ausencia de alertas | 57M registros; brecha sin detectar por 1+ año |
| SolarWinds | 2020 | Gaps en logging | Atacantes activos 9+ meses |
| Capital One | 2019 | Alertas WAF ignoradas | 100M+ registros robados |
| Target | 2013 | Alertas correctas, ignoradas | 40M tarjetas comprometidas |

**Herramientas comunes:**

| Herramienta | Tipo | Uso en DevSecOps |
|---|---|---|
| `Splunk / Elastic SIEM` | SIEM | Agregación centralizada, correlación y visualización |
| `AWS CloudTrail + GuardDuty` | Cloud-native | Auditoría de APIs cloud y detección de amenazas con ML |
| `Falco` | Runtime | Detección a nivel syscall para contenedores y K8s |
| `Wazuh` | Open Source | SIEM open-source con File Integrity Monitoring (FIM) |
| `Grafana + Loki` | Observabilidad | Stack de logs estructurados + alerting cloud-native |

---

### 3. Prevención y Mitigación

**Qué registrar:**
- Todos los eventos de autenticación: éxitos, fallos, bloqueos y bypass de MFA
- Escaladas de privilegios y cambios en roles o permisos
- Fallos de control de acceso y violaciones de política
- Eventos de pipeline CI/CD: builds, deploys y cambios de configuración

**Infraestructura de logs:**
- SIEM centralizado, inmutable y a prueba de manipulaciones
- Retención mínima de **12 meses** (PCI-DSS / HIPAA)
- Almacenamiento WORM (Write Once Read Many)
- Red y credenciales separadas para la infra de logs

**Alertas efectivas:**
- Umbrales para patrones anómalos (≥5 fallos login / 5 minutos)
- SLA de triage: alertas críticas respondidas en **≤15 minutos**
- Probar pipeline de alertas con eventos sintéticos periódicamente
- Integrar alertas en playbooks SOAR automatizados

---

### 4. Buenas Prácticas en DevSecOps

- Logging estructurado **(JSON)** con campos consistentes: `timestamp`, `user`, `ip`, `acción`, `resultado`
- Incluir **correlation ID** en cada log para trazabilidad end-to-end
- Aplicar integridad de logs: firmas digitales o S3 Object Lock (WORM)
- Incorporar logging como **requisito no funcional** en user stories del sprint
- Incluir eventos de pipeline CI/CD y cloud en el SIEM central — no solo app logs
- Usar **Falco** en Kubernetes para detección de anomalías a nivel de syscall
- Definir runbooks SOAR para alertas frecuentes — respuesta automatizada en segundos
- Realizar **threat hunting proactivo** sobre logs históricos al menos mensualmente

---

### 5. Configuraciones Recomendadas

**AWS CloudTrail · Logs Inmutables WORM**
```bash
# Trail multi-región con validación de integridad
aws cloudtrail create-trail \
  --name prod-audit-trail \
  --s3-bucket immutable-logs-prod \
  --is-multi-region-trail \
  --enable-log-file-validation

# S3 Object Lock — retención 1 año en modo COMPLIANCE
aws s3api put-object-lock-configuration \
  --bucket immutable-logs-prod \
  --object-lock-configuration \
  '{"ObjectLockEnabled":"Enabled",
    "Rule":{"DefaultRetention":
    {"Mode":"COMPLIANCE","Years":1}}}'
```

**Falco · Regla de detección runtime en Kubernetes**
```yaml
# Detectar shell inesperado en contenedor de producción
- rule: Shell en Contenedor Producción
  desc: Shell inesperado en contenedor prod
  condition: >
    spawned_process and container
    and proc.name in (shell_binaries)
    and k8s.ns.name = "production"
  output: >
    "Shell detectado (user=%user.name
     pod=%k8s.pod.name ns=%k8s.ns.name
     cmd=%proc.cmdline)"
  priority: WARNING
  tags: [container, shell, T1059]
```

---

### 6. Controles de Seguridad

| 🛡 Preventivos | 🔍 Detectivos | 🔧 Correctivos |
|---|---|---|
| Política de logging obligatorio en todos los servicios | Reglas SIEM: brute force y movimiento lateral | Runbooks SOAR de respuesta automatizada |
| Logging estructurado (JSON) con campos estándar | Alertas por viaje imposible y accesos anómalos | Análisis forense de logs post-incidente |
| Envío de logs al SIEM en tiempo real | Threat hunting mensual sobre logs históricos | Ajuste periódico de umbrales de alerta |
| Almacenamiento WORM para logs críticos | Correlación IOC en tiempo real (MITRE ATT&CK) | Revisión de gaps en logging post-incidente |
| Logging como requisito en user stories | Falco para detección runtime en Kubernetes | Actualizar playbooks con lecciones aprendidas |

---

### 7. Conclusión

La observabilidad de seguridad no es opcional — es un **requisito de arquitectura**. Pipelines CI/CD, contenedores y APIs cloud deben estar instrumentados desde el primer día. La regla más importante: **una alerta que no se revisa equivale a no tener alerta**. Invertir en SOAR reduce el tiempo de contención de horas a minutos, liberando al equipo de seguridad para el análisis de alta complejidad.

---

## Conclusiones Generales

### DevSecOps: Seguridad como Parte del Pipeline

Las tres vulnerabilidades representan riesgos de naturaleza complementaria: Injection ataca en runtime mediante entradas maliciosas; Insecure Design compromete el sistema desde su concepción arquitectónica; y los fallos de Logging permiten que ambos tipos de ataque pasen desapercibidos durante meses.

La respuesta efectiva no es tratar la seguridad como una fase final, sino como un atributo transversal en cada etapa del ciclo de vida:

```
[Diseño]        → Threat modeling, requisitos de seguridad, abuse cases
[Código]        → SAST en cada PR, revisiones de código, secure coding
[Pipeline]      → DAST en staging, policy-as-code (OPA), dependency scanning
[Runtime]       → WAF, Falco, rate limiting, zero-trust (mTLS)
[Observabilidad]→ SIEM centralizado, alertas accionables, SOAR, threat hunting
```

El principio de **shift-left** aplicado a las tres categorías OWASP es la forma más efectiva de reducir el costo y el impacto de las vulnerabilidades.

---

### Síntesis por Vulnerabilidad

#### 🔴 A05 · Injection — Prioridad Inmediata
Los controles técnicos son maduros y disponibles. No hay justificación para exponer intérpretes a inputs sin sanitizar: **consultas parametrizadas + SAST en cada PR** es el mínimo no negociable.

#### 🟠 A06 · Insecure Design — Cambio Cultural
Threat modeling en cada sprint, security champions en cada squad, y criterios de aceptación de seguridad en cada user story — **antes de escribir una línea de código**.

#### 🟡 A09 · Logging Failures — Visibilidad Total
Sin observabilidad los otros controles son ciegos. **Logs inmutables + umbrales de alerta + respuesta automatizada (SOAR)** es la base de cualquier estrategia de detección y respuesta.

---

## Referencias

| Recurso | URL |
|---|---|
| OWASP Top 10 · 2021 | https://owasp.org/Top10/ |
| NIST SP 800-53 Rev. 5 | https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final |
| MITRE ATT&CK Framework | https://attack.mitre.org/ |
| PCI DSS v4.0 — Req. 10 (Logging) | https://www.pcisecuritystandards.org/ |
| OWASP Testing Guide v4.2 | https://owasp.org/www-project-web-security-testing-guide/ |
| OWASP Secure Coding Practices | https://owasp.org/www-project-secure-coding-practices-quick-reference-guide/ |
| CWE Top 25 · MITRE | https://cwe.mitre.org/top25/ |

---

*OWASP Top 10 · 2021 Standard · Seguridad en Entornos Cloud y DevSecOps · Marzo 2026*
