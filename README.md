# SecOps AI — Agente de Ciberseguridad Autónomo
## Estudiantes
- Isabella Plaza Diaz
- Maria Cristina Dominguez

**Universidad del Cauca**  
**Profesor:** Javier Alexander Hurtado  
**Fecha:** Junio 2026
**Framework:** OpenClaw v2026.5.28  
**Canal:** Telegram Bot (@mi_agente_enfasis_bot)

---

## 1. Descripción del Proyecto
Agente de ciberseguridad autónomo que opera a través de Telegram,
capaz de ejecutar auditorías de red, gestionar reglas de firewall
y analizar logs de autenticación en un servidor Linux Ubuntu,
usando un modelo de IA gratuito compatible con la API de OpenAI.

---

## 2. Arquitectura del Sistema

Usuario en Telegram
        ↓
Plugin Telegram (OpenClaw Gateway - systemd)
        ↓
Contexto: SOUL.md + AGENTS.md + SKILL.md
        ↓
OpenRouter API → Modelo gratuito (cohere/north-mini-code:free)
        ↓
tool_calls estructurados → exec-approvals.json (allowlist)
        ↓
Scripts bash en ~/.openclaw/workspace/skills/
        ↓
Resultado → Modelo → Respuesta en lenguaje natural → Telegram

---

## 3. Stack Tecnológico
- SO: Ubuntu Linux
- Framework: OpenClaw v2026.5.28
- LLM Provider: OpenRouter (openrouter/free)
- Canal: Telegram Bot API
- Herramientas locales: nmap, ufw
- Servicio: systemd (usuario)

---

## 4. Errores Encontrados y Soluciones

### Error 1: TOOLS.md e IDENTITY.md no funcionaban
**Síntoma:** El modelo generaba JSON en el chat en vez de ejecutar scripts.  
**Causa:** Estos archivos no son una convención de OpenClaw. No existen
en el framework. El modelo recibía instrucciones como texto plano,
no como definiciones de herramientas estructuradas.  
**Solución:** Migrar a la convención real de OpenClaw:
SOUL.md, AGENTS.md y SKILL.md dentro de skills/network-audit/.

### Error 2: Modelo qwen2.5:1.5B no soporta tool calling
**Síntoma:** El modelo respondía con JSON de llamada a función
como texto en el chat, nunca como tool_calls estructurados.  
**Causa:** Un modelo de 1.5B parámetros no tiene capacidad cognitiva
suficiente para usar el sistema de herramientas de forma confiable.
**Prueba diagnóstica:**
```bash
curl http://127.0.0.1:11434/api/chat -d '{
  "model": "qwen2.5:1.5b",
  "tools": [...],
  "messages": [...]
}'
# Resultado: tool_calls vacío, respuesta en content como texto
```
**Solución:** Migrar a qwen2.5:7b-instruct-q4_K_M (mínimo recomendado).

### Error 3: Timeout con modelo 7B en CPU local
**Síntoma:** "LLM request timed out" después de 365 segundos.  
**Log exacto:**
[diagnostic] stalled session: activeWorkKind=model_call

lastProgressAge=365s classification=stalled_agent_run

[telegram] embedded run agent end: error=LLM request timed out.

**Causa:** El modelo corre en CPU de la VM. A ~87ms por token de
prefill, un prompt con 9 plugins activos (browser, canvas, etc.)
generaba miles de tokens que tardaban minutos en procesarse.
OpenClaw abortaba la operación antes de que el modelo respondiera.  
**Solución:** Migrar a OpenRouter (modelo corre en GPU remota).
El mismo prompt que tardaba 365s tardó 2-3 segundos en la nube.

### Error 4: openrouter/auto no soporta tool calling en tier free
**Síntoma:** "LLM request timed out. rawError=Provider finish_reason: error"
en exactamente 2998ms (3 segundos).  
**Causa:** openrouter/auto selecciona el "mejor modelo disponible"
que generalmente es de pago. Con cuenta gratuita, la solicitud
falla inmediatamente al no tener créditos para ese modelo.  
**Solución:** Usar openrouter/free que filtra específicamente
modelos gratuitos con soporte de tool calling.

### Error 5: approval timed out en exec
**Síntoma:** "Command did not run: approval timed out"  
**Causa:** exec-approvals.json tenía "ask": "on-miss" que requería
aprobación manual vía web UI. En una sesión de Telegram nadie
aprueba esa solicitud y expira.  
**Solución:** Cambiar a "ask": "off" con allowlist explícito
de los 4 scripts autorizados.

### Error 6: Modelos :free específicos deprecados
**Síntoma:** {"error":"This model is unavailable for free","code":404}  
**Modelos afectados:**
- meta-llama/llama-3.1-8b-instruct:free → removido del tier free
- meta-llama/llama-4-scout:free → removido del tier free  
- meta-llama/llama-4-maverick:free → removido del tier free
- openai/gpt-oss-120b:free → requiere aceptar términos en web  
**Causa:** OpenRouter rota modelos gratuitos sin previo aviso.  
**Solución:** Usar openrouter/free (router automático) en vez
de hardcodear un slug específico. Selecciona automáticamente
el mejor modelo gratuito disponible con tool calling.

### Error 7: UFW sin --force en contexto no interactivo
**Síntoma:** El script enable_firewall se colgaba sin responder.  
**Causa:** ufw enable pregunta "¿Estás seguro? [y|N]" de forma
interactiva. En un proceso lanzado por OpenClaw no hay terminal
para responder, el proceso queda bloqueado esperando input.  
**Solución:** Usar sudo ufw --force enable que omite la pregunta.

---

## 5. Estructura de Archivos Final
~/.openclaw/

├── openclaw.json          # Configuración principal del gateway

├── exec-approvals.json    # Allowlist de seguridad para exec

└── workspace/

├── SOUL.md            # Personalidad del agente

├── AGENTS.md          # Reglas de comportamiento

└── skills/

├── audit_network        # Script nmap

├── ufw_status           # Script estado firewall

├── apply_firewall_rule  # Script bloquear puerto

├── enable_firewall      # Script activar UFW

├── auth_log             # Script logs de autenticación

└── network-audit/

└── SKILL.md         # Instrucciones de herramientas
---

## 6. Comandos de Verificación del Sistema

```bash
# Ver estado del servicio
systemctl --user status openclaw-gateway.service

# Ver logs en tiempo real
journalctl --user -u openclaw-gateway.service -f

# Verificar tool calling del modelo directamente
curl -s https://openrouter.ai/api/v1/chat/completions \
  -H "Authorization: Bearer TU_KEY" \
  -H "Content-Type: application/json" \
  -d @/tmp/test.json

# Verificar estado del firewall
sudo ufw status

# Probar script directamente
time ~/.openclaw/workspace/skills/audit_network
```

---

## 7. Flujo de Demo

1. Mostrar servicio activo: `systemctl --user status openclaw-gateway`
2. En Telegram: `/new` → verificar personalidad SecOps AI
3. En Telegram: `/audit_network` → reporte de puertos automático
4. En Telegram: `quiero cerrar el puerto 7070` → confirmación → bloqueo
5. En Telegram: `habilita el firewall` → UFW activo
6. En terminal: `sudo ufw status` → verificar reglas aplicadas en vivo

