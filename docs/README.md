# Índice de Documentación OBSEA3

## Documentos Principales

### 0. [WORKFLOW.md](WORKFLOW.md) - **Flujo de Trabajo de Desarrollo** ⭐
**Para**: Desarrolladores (lectura obligatoria)

**Contenido**:
- Estructura de directorios (`~/obsea/` vs `/opt/obsea/`)
- Ciclo diario: editar → compilar → probar → commit
- Dónde editar código (respuesta: `~/obsea/`)
- Dónde se compila (respuesta: `/opt/obsea/`)
- Gestión de configuración
- Tareas comunes (agregar nodo, endpoint API, etc.)
- Troubleshooting de desarrollo

**Cuándo usar**: Antes de empezar a desarrollar. Explica el flujo completo.

---

## ⚠️ EMPIEZA AQUÍ SI TIENES PROBLEMAS

**¿Los nodos ROS2 no se ven? ¿Service call timeout?**

→ Lee primero: **[ROS_DOMAIN_ID.md](ROS_DOMAIN_ID.md)** - Solución al 99% de problemas de conectividad ROS2

### 1. [SETUP.md](SETUP.md) - Configuración Inicial
**Para**: Nuevas instalaciones y administradores de sistema

**Contenido**:
- Instalación completa paso a paso
- Requisitos previos
- Configuración de usuarios y permisos
- Variables de entorno
- Verificación post-instalación
- Troubleshooting de instalación

**Cuándo usar**: Primera vez configurando OBSEA3 en un sistema nuevo.

---

### 2. [DESARROLLO.md](DESARROLLO.md) - Guía de Desarrollo
**Para**: Desarrolladores y programadores

**Contenido**:
- Ciclo de desarrollo diario
- Compilar y probar cambios
- Agregar nuevas funcionalidades:
  - Mensajes ROS2
  - Endpoints API
  - Configuraciones YAML
  - Tablas de base de datos
- Modo dummy para testing
- Errores comunes y soluciones
- Tests unitarios y de integración

**Cuándo usar**: Desarrollando nuevas features o modificando código existente.

---

### 3. [ALARMAS.md](ALARMAS.md) - Sistema de Alarmas
**Para**: Operadores, desarrolladores y administradores

**Contenido**:
- Tipos de alarmas (overcurrent, undervolt, overvolt)
- Configuración de límites
- Métodos para consultar alarmas:
  - Script CLI (`check_alarms.sh`)
  - API REST (`/telemetry/alarms`)
  - Consulta SQL directa
- Lógica de deduplicación
- Ciclo de vida de alarmas
- Troubleshooting de alarmas

**Cuándo usar**: Monitoreando el sistema, configurando límites de seguridad, o debuggeando problemas de alarmas.

---

### 4. [.github/ARCHITECTURE.md](../.github/ARCHITECTURE.md) - Guía para AI Agents
**Para**: Asistentes de IA y programadores usando herramientas AI

**Contenido**:
- Arquitectura completa del sistema
- Componentes ROS2 (paquetes, nodos, topics, services)
- Patrones de configuración
- Workflows de desarrollo
- Convenciones de código
- Common pitfalls
- Referencias de archivos clave

**Cuándo usar**: Contextualizando GitHub Copilot u otros AI coding assistants.

---

## Guías Rápidas por Tarea

### Quiero... Instalar OBSEA3 por primera vez
→ Lee: **[SETUP.md](SETUP.md)** → Sección "Instalación Completa"

### Quiero... Desarrollar una nueva feature
→ Lee: **[DESARROLLO.md](DESARROLLO.md)** → Sección "Agregar Nueva Funcionalidad"

### Quiero... Monitorear alarmas del sistema
→ Lee: **[ALARMAS.md](ALARMAS.md)** → Sección "Consultar Alarmas Activas"

### Quiero... Cambiar límites de seguridad
→ Lee: **[ALARMAS.md](ALARMAS.md)** → Sección "Configuración de Límites"

### Quiero... Entender la arquitectura general
→ Lee: **[.github/ARCHITECTURE.md](../.github/ARCHITECTURE.md)** → Sección "Architecture Overview"

### Quiero... Compilar después de cambios en código
→ Lee: **[DESARROLLO.md](DESARROLLO.md)** → Sección "Ciclo de Desarrollo Diario" → Paso 3

### Quiero... Agregar un nuevo usuario
→ Lee: **[SETUP.md](SETUP.md)** → Sección "Crear Usuarios" o ejecuta:
```bash
python bin/more/obsea_user_admin.py --add nombre_usuario --role admin
```

### Quiero... Ver logs del sistema
→ Lee: **[DESARROLLO.md](DESARROLLO.md)** → Sección "Monitorear Logs" o ejecuta:
```bash
tail -f log/current/api_gateway.log
tail -f log/current/hw_supervisor.log
```

### Quiero... Probar sin hardware (modo dummy)
→ Lee: **[DESARROLLO.md](DESARROLLO.md)** → Sección "Modo Dummy para Testing"

### Quiero... Debuggear un problema
1. **Error de build**: [DESARROLLO.md](DESARROLLO.md) → "Errores Comunes"
2. **Error de alarmas**: [ALARMAS.md](ALARMAS.md) → "Troubleshooting"
3. **Error de instalación**: [SETUP.md](SETUP.md) → "Troubleshooting Instalación"

---

## Estructura de Archivos Relacionados

```
/opt/obsea/
├── docs/                          # ← ESTÁS AQUÍ
│   ├── README.md                  # Este índice
│   ├── SETUP.md                   # Instalación inicial
│   ├── DESARROLLO.md              # Guía de desarrollo
│   └── ALARMAS.md                 # Sistema de alarmas
├── .github/
│   └── ARCHITECTURE.md    # Contexto para AI
├── bin/
│   ├── README.md                  # Descripción de scripts
│   ├── .venv                      # Variables de entorno
│   ├── secrets                    # Tokens (NO commitear)
│   └── *.sh                       # Scripts de operación
├── ros2_ws/                       # Workspace ROS2
│   └── src/
│       ├── obsea3_interfaces/     # Mensajes/servicios
│       ├── obsea3_hw_interfaces/  # Hardware supervisor
│       ├── obsea3_core/           # Lógica de negocio
│       ├── obsea3_api_gateway/    # API REST/WebSocket
│       └── obsea3_web/            # Frontend React
├── data/                          # Configuraciones runtime
│   ├── *.yaml                     # Configs activas
│   └── *.db                       # Bases de datos SQLite
└── log/                           # Logs del sistema
    └── current/                   # Symlink a hoy
```

---

## Flujo de Trabajo Sugerido

### Para Nuevos Desarrolladores

1. **Día 1**: Leer [.github/ARCHITECTURE.md](../.github/ARCHITECTURE.md) para entender arquitectura
2. **Día 2**: Seguir [SETUP.md](SETUP.md) para instalar sistema localmente en modo dummy
3. **Día 3**: Practicar ciclo de desarrollo con [DESARROLLO.md](DESARROLLO.md)
4. **Día 4+**: Desarrollar features, consultar [ALARMAS.md](ALARMAS.md) cuando sea necesario

### Para Operadores

1. Instalar siguiendo [SETUP.md](SETUP.md)
2. Aprender a monitorear con [ALARMAS.md](ALARMAS.md)
3. Consultar [DESARROLLO.md](DESARROLLO.md) solo para troubleshooting básico

### Para Administradores de Sistema

1. [SETUP.md](SETUP.md) → Instalación completa en modo producción
2. [ALARMAS.md](ALARMAS.md) → Configurar límites apropiados
3. [DESARROLLO.md](DESARROLLO.md) → Diagnosticar problemas complejos

---

## Convenciones de Documentación

- **Comandos**: Siempre con ruta absoluta o indicando `cd` requerido
- **Ejemplos**: Incluyen salida esperada para verificación
- **Advertencias**: Marcadas con **IMPORTANTE** o **NOTA**
- **Referencias cruzadas**: Links relativos entre documentos

---

## Archivos Históricos

Documentos de debugging y análisis de problemas específicos:

- **[archive/ALARM_DEDUP_FIX.md](archive/ALARM_DEDUP_FIX.md)** - Análisis y solución del spam de alarmas (2025-12-09)
- **[archive/CHANGELOG_2025-12-10.md](archive/CHANGELOG_2025-12-10.md)** - Cambios del 10 de diciembre 2025
- **[archive/CHANGELOG_2025-12-11.md](archive/CHANGELOG_2025-12-11.md)** - Cambios del 11 de diciembre 2025

**Nota**: El changelog consolidado actual está en la raíz: [`../CHANGELOG.md`](../CHANGELOG.md)

---

## Contribuir a la Documentación

### Agregar Nueva Documentación

1. Crear archivo en `docs/` con nombre descriptivo en MAYÚSCULAS (ej: `NUEVO_TEMA.md`)
2. Agregar entrada en este índice (README.md)
3. Seguir estructura de documentos existentes:
   - Descripción general
   - Secciones con headers (`##`)
   - Ejemplos de comandos con salida esperada
   - Troubleshooting al final

### Actualizar Documentación Existente

1. Mantener consistencia con estilo actual
2. Actualizar fecha de modificación en commit
3. Si cambios mayores, notificar en changelog del sistema

---

## Recursos Adicionales

- **ROS2 Jazzy Docs**: https://docs.ros.org/en/jazzy/
- **FastAPI Docs**: https://fastapi.tiangolo.com/
- **SQLite Docs**: https://www.sqlite.org/docs.html
- **React Docs**: https://react.dev/

---

**Última actualización**: 2025-12-11  
**Mantenedores**: Equipo OBSEA3  
**Contacto**: Ver archivo AUTHORS en repositorio
