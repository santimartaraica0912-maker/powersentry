# PowerSentry — Evaluador de Seguridad Web Externo (prototipo)

Escáner de **postura de seguridad** para activos web, operado **100 % desde el exterior**:
sin agentes, sin credenciales, sin acceso a la red interna del cliente. Diseñado para
**complementar** —no repetir— los servicios de detección y mitigación de contenido hostil
que ya presta la empresa.

> Prototipo con fines de evaluación. Ejecútelo solo contra activos propios o con
> autorización escrita.

---

## Los dos rasgos pedidos

### 1. "Ver el proceso detrás de cada análisis"
Cada comprobación registra un **paso de traza** (`Txxx`) con: la acción realizada, la
**regla** evaluada, lo **observado** y el **veredicto** (PASS/FAIL/WARN/INFO). Cada
hallazgo cita los pasos que lo originaron, así cualquiera puede **reproducir y verificar**.
- En vivo: opción `--explain`.
- En el informe HTML: sección "Proceso / evidencia" por hallazgo + tabla de traza completa.
- En JSON: array `trace`.

### 2. "Pensar como un ciberdelincuente" (para ser más efectivo)
La **lente de atacante** razona, a partir de la evidencia **pasiva**, cómo un adversario
externo *descubriría → armaría → impactaría* cada debilidad, y las ordena en una
**cadena de ataque** (reconocimiento → entrada → escalada → impacto).
Es razonamiento **defensivo de priorización**: **no ejecuta** ninguna acción ofensiva.

---

## Alcance (menú de comprobaciones)

| Módulo | Qué revisa |
|---|---|
| **transport** | Versiones TLS, cifrados, certificado (vigencia, clave, firma, SAN), HSTS, contenido mixto |
| **headers** | Cabeceras de seguridad, **calidad de CSP**, cookies (Secure/HttpOnly/SameSite), fuga de versión |
| **supply_chain** | Inventario de terceros, **SRI**, riesgo de **formjacking**, componentes JS con CVE |
| **surface** | security.txt, formularios; y (modo activo) rutas sensibles, paneles, Swagger/GraphQL |
| **dns** | DNSSEC, CAA, registros colgantes (subdomain takeover), AXFR (activo) |
| **privacy** | Rastreo previo al consentimiento, datos personales en cliente/URL |
| **resilience** | WAF/CDN, exposición de IP de origen, redirecciones abiertas, (rate-limit **no** ejecutado) |

**Excluido por diseño:** autenticación de correo **SPF / DKIM / DMARC** (ya lo cubre la empresa).
La exclusión queda registrada en la traza para que sea auditable.

---

## Modos y seguridad operativa
- **Pasivo (por defecto):** equivale a una visita de navegador + inspección tipo SSL Labs. No intrusivo.
- **Activo:** sondeos intrusivos (rutas sensibles, AXFR). Requiere `--mode active --i-am-authorized`
  y una ventana/alcance pactados. La prueba de DoS de capa 7 **nunca** se ejecuta automáticamente.
- Cliente con **alcance de host único**, **límite de tasa** y **tope duro de peticiones**;
  solo GET/HEAD; sin seguir redirecciones a ciegas.

---

## Uso

### Opción A — Página web (recomendada)
Metes una URL, pulsas **Analizar** y ves el proceso en vivo + el informe.

- **Windows:** doble clic en `iniciar_powersentry.bat` (abre el navegador solo).
- **Manual:**
  ```bash
  python -m webguard --serve --port 8181
  ```
  Luego abre <http://127.0.0.1:8181/> en tu navegador.

### Opción B — Línea de comandos

```bash
# Instalación opcional (mejora certificados y DNS; funciona sin ellas)
pip install -r requirements.txt

# Escaneo pasivo con informe HTML + JSON
python -m webguard seguridadmaxima.net --html informe.html --json hallazgos.json

# Ver el proceso paso a paso en vivo
python -m webguard seguridadmaxima.net --explain

# Un solo módulo
python -m webguard ejemplo.com --only transport,headers

# Modo activo (requiere autorización)
python -m webguard ejemplo.com --mode active --i-am-authorized --html informe.html
```

Salidas: informe de consola, `--html` (informe navegable, tema oscuro), `--json` (integración).
Código de salida: `10` si hay hallazgos altos/críticos, `11` si medios, `0` limpio.

---

## Arquitectura

```
webguard/
  model.py        # Severidad, Finding, AttackNarrative, Compliance
  trace.py        # Motor de trazabilidad  (rasgo #1)
  attacker.py     # Cadena de ataque       (rasgo #2)
  net.py          # Cliente HTTP pasivo + TLS + DNS (seguro y acotado)
  context.py      # Contexto de escaneo (fetch con traza automática)
  htmlutil.py     # Parseo HTML (stdlib)
  analyzers/      # Un módulo por dominio del menú
  scoring.py      # Puntaje 0–100 y grado A–F
  report.py       # Render consola + HTML + JSON (report_to_dict)
  runner.py       # Orquestación
  cli.py          # Interfaz de línea de comandos
  webapp.py       # Servidor web + SSE (proceso en vivo)
  web/            # index.html, app.css, app.js (la página)
```

Sin dependencias obligatorias (Python 3.10+). `cryptography` y `dnspython` son opcionales.
