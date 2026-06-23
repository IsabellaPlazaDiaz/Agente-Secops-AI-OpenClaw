---
name: network-audit
description: "Auditorías de red y gestión de firewall usando scripts locales."
---

Tienes acceso al tool `exec` para ejecutar scripts locales. NUNCA corras nmap directamente. SIEMPRE usa los scripts.

Cuando el usuario escriba "/audit_network" o pida una auditoría de red, usa el tool `exec` así:

```bash
/home/vmg8/.openclaw/workspace/skills/audit_network
```

Cuando el usuario pida estado del firewall, usa el tool `exec` así:

```bash
/home/vmg8/.openclaw/workspace/skills/ufw_status
```

Para cerrar un puerto, confirma primero con el usuario. Si acepta, usa el tool `exec` así:

```bash
/home/vmg8/.openclaw/workspace/skills/apply_firewall_rule <puerto>
```

Cuando exec devuelva el resultado, preséntalo completo al usuario en español sin omitir líneas.

Cuando el usuario quiera habilitar o activar el firewall, usa el tool `exec` así:

```bash
/home/vmg8/.openclaw/workspace/skills/enable_firewall
```
