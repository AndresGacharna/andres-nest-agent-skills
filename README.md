# andres-agent-skills

Skills de agente propias, en el formato estándar `SKILL.md`, para consumirlas desde cualquier runtime que las soporte.

Licencia MIT: úsalas, cópialas y modifícalas sin pedir permiso.

## Skills

| Skill | Qué hace |
|---|---|
| [`andres-nestjs-architecture`](skills/andres-nestjs-architecture/SKILL.md) | Arquitectura y convenciones para APIs NestJS 12 (ESM + TypeORM + PostgreSQL): estructura de módulos, contrato de respuestas, catálogos de errores y mensajes, configuración validada. Incluye una [plantilla completa de módulo](skills/andres-nestjs-architecture/references/module-template.md) con CRUD. |

## Instalación

Cada skill es una carpeta con un `SKILL.md` y, opcionalmente, un `references/`. Instalarla es copiar esa carpeta donde tu herramienta las busque.

```bash
git clone --depth 1 https://github.com/<tu-usuario>/andres-agent-skills /tmp/andres-agent-skills
```

**Claude Code** — por proyecto o para todos tus proyectos:

```bash
# solo este proyecto
mkdir -p .claude/skills
cp -r /tmp/andres-agent-skills/skills/andres-nestjs-architecture .claude/skills/

# todos tus proyectos
mkdir -p ~/.claude/skills
cp -r /tmp/andres-agent-skills/skills/andres-nestjs-architecture ~/.claude/skills/
```

**Otros runtimes** que sigan la convención `.agents/skills/`:

```bash
mkdir -p .agents/skills
cp -r /tmp/andres-agent-skills/skills/andres-nestjs-architecture .agents/skills/
```

Si tu herramienta lee desde `.claude/skills/` pero prefieres versionar en `.agents/skills/`, enlaza una a la otra en vez de duplicar:

```bash
# Windows (PowerShell)
New-Item -ItemType Junction -Path .claude/skills/andres-nestjs-architecture -Target .agents/skills/andres-nestjs-architecture

# macOS / Linux
ln -s ../../.agents/skills/andres-nestjs-architecture .claude/skills/andres-nestjs-architecture
```

En Windows conviene no commitear el enlace: git lo guardaría como una copia duplicada. Agrégalo al `.gitignore`.

## Proyectos que las usan

- [`andres-nestjs-starter`](https://github.com/<tu-usuario>/andres-nestjs-starter) — plantilla base para arrancar APIs NestJS 12 con estas convenciones ya aplicadas y un módulo de ejemplo con CRUD. Trae una copia de `andres-nestjs-architecture` para funcionar recién clonada; **este repo es la fuente canónica**, así que las correcciones van aquí primero.

## Origen

Las convenciones son mías. Se construyeron tomando como referencia convenciones de equipo y la skill de terceros [`nestjs-best-practices`](https://github.com/kadajett/agent-nestjs-skills) de `kadajett`, que **no** se redistribuye aquí (ese proyecto no declara licencia). Si la quieres, instálala desde su repo.
