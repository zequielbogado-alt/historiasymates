# PDR — Player Hub MVP para Historias y Mates

**Proyecto:** Portal de perfiles, stats y rankings para Historias y Mates  
**Versión:** MVP v1  
**Destino del documento:** `C:\Users\bogad\Desktop\PAGINA Historias y Mates\docs`  
**Objetivo para el agente:** analizar la web actual, respetar su arquitectura y diseñar/implementar el MVP sin pisar backend, API, admin panel, estilos ni componentes existentes.

---

## 1. Contexto

Historias y Mates ya tiene una página web funcionando con:

- Frontend publicado.
- Backend/API existente.
- Panel admin robusto existente.
- Navbar, hero, background, fuentes, paleta de colores y estilo visual ya definidos.
- Secciones actuales como Inicio, Guías, Anuncios y Economía/Tokens.

Este MVP debe sumarse como una **evolución del sitio actual**, no como una web nueva ni como un rediseño general.

El servidor ARK ASA ya cuenta con datos vivos provenientes de:

1. **ShadowBot / ArkLinker / Shadow Linker**  
   Resuelve la vinculación `DiscordID ↔ EOSID`.

2. **Kxrse ASA Plugins** instalados y probados:
   - `SurvivorTracker`
   - `SurvivorStats`
   - `ResourceStats`

3. **StructureStats** no está instalado actualmente.  
   El ranking de construcción debe diseñarse como módulo opcional/feature flag. Si las tablas no existen, debe ocultarse o mostrarse como “próximamente”, sin romper el MVP.

---

## 2. Objetivo del MVP

Construir un primer **Player Hub** dentro de la web actual para que un usuario pueda:

1. Iniciar sesión con Discord.
2. Detectar automáticamente su `EOSID` usando la tabla de linkeo de ShadowBot.
3. Ver sus personajes/survivors detectados por mapa.
4. Ver estadísticas básicas del personaje.
5. Ver ranking de farmeo.
6. Ver ranking de construcción si existe `StructureStats`.
7. Permitir al admin buscar jugadores y ver un resumen útil desde el panel admin existente.

---

## 3. Principios obligatorios

### 3.1 No pisar lo existente

El agente debe inspeccionar primero la estructura real del proyecto y detectar:

- Framework frontend.
- Framework backend.
- Sistema de rutas.
- Sistema de layout.
- Sistema de estilos.
- Sistema de autenticación existente, si lo hay.
- Estructura actual del panel admin.
- Patrón actual de componentes.
- Patrón actual de servicios/API.
- Convenciones de nombres.
- Manejo actual de variables de entorno.

**Prohibido:**

- Crear una app nueva dentro del proyecto salvo que la arquitectura actual lo requiera explícitamente.
- Reemplazar navbar, hero, layout global, paleta, fuentes o background.
- Romper rutas actuales.
- Romper el panel admin existente.
- Duplicar sistemas de auth si ya existe uno.
- Conectar el frontend directamente a MariaDB.

### 3.2 Reutilización

Todo debe intentar reutilizar:

- Layouts existentes.
- Componentes base existentes.
- Cards existentes.
- Botones existentes.
- Tablas existentes.
- Modales existentes.
- Toasts/notificaciones existentes.
- Guards/middlewares/admin checks existentes.
- Servicios HTTP existentes.

Si falta un componente, crear uno nuevo siguiendo el patrón visual del proyecto.

---

## 4. Fuentes de datos confirmadas

### 4.1 DB de linkeo Discord ↔ EOSID

Schema observado:

```txt
s2646_fusionshop.discordlinks
```

Columnas visibles:

```txt
ID
DiscordID
DiscordName
EOSID
VerifiedGiven
OnlineRoleGiven
```

Uso principal:

```sql
SELECT DiscordID, DiscordName, EOSID
FROM s2646_fusionshop.discordlinks
WHERE DiscordID = ?;
```

Notas:

- `DiscordID` debe tratarse siempre como `string`, nunca como número JS.
- `EOSID` es la clave para cruzar con las tablas de stats.
- Puede haber usuarios no vinculados. El frontend debe tener un estado claro para eso.

---

### 4.2 DB de tracking web

Schema:

```txt
info_web_tracker
```

Tablas confirmadas:

```txt
gathering_stats
pvpve_stats
survivors
tribe_stats
```

#### `survivors`

Columnas visibles:

```txt
eos_id
survivor_name
survivor_id
tribe_name
tribe_id
map_name
```

#### `pvpve_stats`

Columnas visibles:

```txt
eos_id
survivor_id
survivor_level
survivor_kills
survivor_kills_mounted
survivor_kills_victim_mounted
deaths_by_survivor
deaths_by_survivor_mounted
deaths_while_mounted
wild_dino_kills
wild_dino_kills_mounted
tamed_dino_kills
tamed_dino_kills_mounted
deaths_by_wild_dino
deaths_by_wild_dino_while_mounted
deaths_by_tamed_dino
```

#### `gathering_stats`

Columnas visibles:

```txt
eos_id
survivor_id
item_name
total_harvested
```

#### `tribe_stats`

Columnas visibles:

```txt
tribe_id
```

---

## 5. Tablas opcionales de StructureStats

`StructureStats` no está activo ahora. Si se instala más adelante, deberían aparecer tablas similares a:

```txt
building_stats_player
building_stats_tribe
destruction_player
destruction_tribe
```

Según el plugin, la tabla útil para ranking de construcción sería principalmente:

```txt
info_web_tracker.building_stats_player
```

Columnas esperadas:

```txt
eos_id
survivor_id
survivor_name
tribe_name
tribe_id
structures_placed
```

El módulo de construcción debe:

- Detectar si la tabla existe.
- No fallar si no existe.
- Ocultar ranking o mostrar estado “próximamente”.
- No bloquear el resto del Player Hub.

---

## 6. Modelo funcional del Player Hub

### 6.1 Flujo de usuario

```txt
Usuario entra a /perfil
        ↓
Si no tiene sesión → botón “Entrar con Discord”
        ↓
Discord OAuth2 identify
        ↓
Backend obtiene Discord user id
        ↓
Backend busca EOSID en s2646_fusionshop.discordlinks
        ↓
Si no está vinculado → mostrar estado “Discord no vinculado al servidor”
        ↓
Si está vinculado → consultar info_web_tracker
        ↓
Mostrar perfil, survivors, stats y recursos
```

### 6.2 Usuario no vinculado

Debe mostrar una pantalla amable:

- “Tu Discord todavía no está vinculado a un personaje del servidor.”
- Explicar que debe usar el sistema de vinculación de ShadowBot/ArkLinker.
- Botón para ir a Discord.
- No mostrar errores técnicos.

### 6.3 Usuario vinculado sin stats todavía

Puede pasar si:

- Se vinculó pero aún no entró desde que los plugins están activos.
- El mapa no tiene plugins cargados.
- Todavía no farmeó/mató nada.

Mostrar:

- Vinculación activa.
- EOS parcialmente oculto.
- Mensaje: “Todavía no hay datos suficientes. Entrá al server y jugá un ratito para que aparezcan tus stats.”

---

## 7. Rutas frontend sugeridas

El agente debe adaptar nombres según la estructura actual del proyecto.

### Ruta pública/autenticada

```txt
/perfil
```

Función:

- Login Discord si no hay sesión.
- Perfil del jugador si hay sesión.
- Estado de no vinculado si no existe EOS.

### Ruta pública de rankings

```txt
/rankings
```

Función:

- Ranking de farmeo.
- Ranking de construcción si está disponible.
- Filtros por recurso/mapa si el backend lo soporta.

### Ruta admin

Usar el sistema admin actual. No crear un panel paralelo.

Sugerencia:

```txt
/admin/player-hub
```

o equivalente según la arquitectura existente.

Función:

- Buscar jugador por DiscordName, DiscordID, EOSID, survivor_name, tribe_name.
- Ver resumen de linkeo y stats.

---

## 8. Endpoints API sugeridos

Usar un prefijo que no choque con la API existente. Ejemplo:

```txt
/api/player-hub/*
```

Si el backend actual ya tiene convención distinta, respetarla.

### 8.1 Auth

Si ya existe autenticación con Discord, reutilizarla.

Si no existe, agregar:

```txt
GET /api/auth/discord/login
GET /api/auth/discord/callback
POST /api/auth/logout
GET /api/auth/me
```

Requisitos:

- Usar OAuth2 Authorization Code Flow.
- Scope mínimo: `identify`.
- Usar `state` anti-CSRF.
- No exponer access tokens al frontend.
- Guardar sesión segura server-side o cookie segura según patrón existente.

### 8.2 Perfil propio

```txt
GET /api/player-hub/me
```

Respuesta sugerida:

```json
{
  "linked": true,
  "discord": {
    "id": "95029253462765618",
    "name": "Abogado"
  },
  "eosIdMasked": "000287...711",
  "survivors": [],
  "stats": {},
  "topResources": []
}
```

Si no está vinculado:

```json
{
  "linked": false,
  "reason": "DISCORD_NOT_LINKED"
}
```

### 8.3 Survivors del usuario

```txt
GET /api/player-hub/me/survivors
```

### 8.4 Recursos del usuario

```txt
GET /api/player-hub/me/resources
```

Query params opcionales:

```txt
?limit=20
?survivorId=...
```

### 8.5 Ranking de farmeo

```txt
GET /api/player-hub/rankings/farming
```

Query params opcionales:

```txt
?item=Metal
?limit=10
```

### 8.6 Ranking de construcción

```txt
GET /api/player-hub/rankings/construction
```

Si `StructureStats` no está disponible:

```json
{
  "available": false,
  "reason": "STRUCTURE_STATS_NOT_INSTALLED"
}
```

### 8.7 Admin search

```txt
GET /api/player-hub/admin/search?q=...
GET /api/player-hub/admin/player/:eosId
```

Debe estar protegido por el sistema admin existente.

---

## 9. SQL recomendado

### 9.1 Vista de perfiles

Crear solo si el proyecto usa SQL migrations o si el admin acepta crear views manuales. Si no, implementar el JOIN directamente en el backend.

```sql
CREATE OR REPLACE VIEW info_web_tracker.web_player_profiles AS
SELECT
    dl.DiscordID,
    dl.DiscordName,
    dl.EOSID,

    s.survivor_name,
    s.survivor_id,
    s.tribe_name,
    s.tribe_id,
    s.map_name,

    ps.survivor_level,
    ps.survivor_kills,
    ps.survivor_kills_mounted,
    ps.survivor_kills_victim_mounted,
    ps.deaths_by_survivor,
    ps.deaths_by_survivor_mounted,
    ps.deaths_while_mounted,
    ps.wild_dino_kills,
    ps.wild_dino_kills_mounted,
    ps.tamed_dino_kills,
    ps.tamed_dino_kills_mounted,
    ps.deaths_by_wild_dino,
    ps.deaths_by_wild_dino_while_mounted,
    ps.deaths_by_tamed_dino

FROM s2646_fusionshop.discordlinks dl

LEFT JOIN info_web_tracker.survivors s
    ON s.eos_id = dl.EOSID

LEFT JOIN info_web_tracker.pvpve_stats ps
    ON ps.eos_id = dl.EOSID
    AND ps.survivor_id = s.survivor_id;
```

Consulta:

```sql
SELECT *
FROM info_web_tracker.web_player_profiles
WHERE DiscordID = ?;
```

### 9.2 Recursos del usuario

```sql
SELECT
    gs.item_name,
    SUM(gs.total_harvested) AS total_harvested
FROM info_web_tracker.gathering_stats gs
INNER JOIN s2646_fusionshop.discordlinks dl
    ON dl.EOSID = gs.eos_id
WHERE dl.DiscordID = ?
GROUP BY gs.item_name
ORDER BY total_harvested DESC
LIMIT ?;
```

### 9.3 Ranking global de farmeo

```sql
SELECT
    dl.DiscordName,
    dl.DiscordID,
    gs.item_name,
    SUM(gs.total_harvested) AS total_harvested
FROM info_web_tracker.gathering_stats gs
INNER JOIN s2646_fusionshop.discordlinks dl
    ON dl.EOSID = gs.eos_id
GROUP BY dl.DiscordName, dl.DiscordID, gs.item_name
ORDER BY total_harvested DESC
LIMIT ?;
```

### 9.4 Ranking por recurso

```sql
SELECT
    dl.DiscordName,
    dl.DiscordID,
    SUM(gs.total_harvested) AS total_harvested
FROM info_web_tracker.gathering_stats gs
INNER JOIN s2646_fusionshop.discordlinks dl
    ON dl.EOSID = gs.eos_id
WHERE gs.item_name LIKE ?
GROUP BY dl.DiscordName, dl.DiscordID
ORDER BY total_harvested DESC
LIMIT ?;
```

### 9.5 Chequear si StructureStats existe

```sql
SELECT COUNT(*) AS table_exists
FROM information_schema.tables
WHERE table_schema = 'info_web_tracker'
  AND table_name = 'building_stats_player';
```

### 9.6 Ranking de construcción opcional

Solo ejecutar si existe `building_stats_player`.

```sql
SELECT
    dl.DiscordName,
    dl.DiscordID,
    bsp.survivor_name,
    bsp.tribe_name,
    SUM(bsp.structures_placed) AS structures_placed
FROM info_web_tracker.building_stats_player bsp
INNER JOIN s2646_fusionshop.discordlinks dl
    ON dl.EOSID = bsp.eos_id
GROUP BY dl.DiscordName, dl.DiscordID, bsp.survivor_name, bsp.tribe_name
ORDER BY structures_placed DESC
LIMIT ?;
```

---

## 10. Seguridad y privacidad

### 10.1 DB

- No conectar frontend directo a MariaDB.
- Usar backend/API existente.
- Crear usuario DB de solo lectura para la web, si es posible.
- Permisos mínimos recomendados:

```sql
GRANT SELECT ON s2646_fusionshop.discordlinks TO 'web_reader'@'localhost';
GRANT SELECT ON info_web_tracker.* TO 'web_reader'@'localhost';
```

- Si se usan views, otorgar permisos adecuados según cómo esté configurado MariaDB.

### 10.2 DiscordID

- Tratar `DiscordID` como string.
- Nunca parsearlo como number en JS/TS.

### 10.3 EOSID

- No mostrar EOSID completo a usuarios normales.
- Mostrarlo parcialmente:

```txt
000287...711
```

- Admins sí pueden ver EOSID completo si el panel actual ya maneja datos sensibles.

### 10.4 OAuth

- Usar `state` para prevenir CSRF.
- No guardar access token en `localStorage`.
- Preferir sesión httpOnly cookie o mecanismo actual del backend.
- No pedir scope `email` salvo que el proyecto lo necesite.
- Scope mínimo suficiente: `identify`.

### 10.5 Consultas

- Usar queries parametrizadas.
- Nada de concatenar `q`, `DiscordID`, `EOSID`, `item_name` directo en SQL.
- Agregar límites máximos a rankings.

---

## 11. Componentes frontend sugeridos

Reutilizar componentes existentes antes de crear nuevos.

### Perfil

- `DiscordLoginButton`
- `LinkedAccountBadge`
- `PlayerProfileHeader`
- `SurvivorCard`
- `SurvivorSelector`
- `StatsGrid`
- `TopResourcesList`
- `EmptyStateNotLinked`
- `EmptyStateNoStats`

### Rankings

- `RankingTabs`
- `RankingTable`
- `RankingFilterBar`
- `ResourceFilterSelect`
- `ConstructionUnavailableCard`

### Admin

- `AdminPlayerSearch`
- `AdminPlayerResultTable`
- `AdminPlayerProfileDrawer` o modal equivalente
- `AdminStatsSummary`

---

## 12. UI/UX

### 12.1 Estética

La UI debe respetar el estilo actual de Historias y Mates:

- Background actual.
- Paleta actual.
- Fuentes actuales.
- Cards/containers actuales.
- Hero y navbar actuales.
- Tono visual de comunidad PvE, amigable, no competitivo agresivo.

### 12.2 Textos sugeridos

#### Usuario no logueado

```txt
Entrá con Discord para ver tu perfil de survivor, personajes detectados y estadísticas del servidor.
```

#### Usuario no vinculado

```txt
Tu Discord todavía no está vinculado a un personaje del servidor.
Vinculá tu cuenta desde Discord/in-game usando el sistema de Shadow Linker y volvé a intentarlo.
```

#### Usuario sin stats

```txt
Tu cuenta está vinculada, pero todavía no tenemos estadísticas suficientes.
Entrá al server, jugá un ratito y tus datos van a aparecer acá.
```

#### Ranking construcción no disponible

```txt
El ranking de construcción todavía no está activo.
Cuando habilitemos el tracking de estructuras, vas a poder ver este ranking acá.
```

---

## 13. Admin panel MVP

Integrar dentro del panel admin actual.

### Funciones mínimas

- Buscar por:
  - `DiscordName`
  - `DiscordID`
  - `EOSID`
  - `survivor_name`
  - `tribe_name`

- Mostrar:
  - DiscordName.
  - DiscordID.
  - EOSID completo.
  - Estado de vinculación.
  - Survivors detectados.
  - Mapas.
  - Tribus.
  - Nivel.
  - Kills/muertes básicas.
  - Top recursos farmeados.

### No hacer en MVP

- No modificar datos.
- No desvincular cuentas.
- No banear.
- No tocar puntos de FusionShop.
- No escribir en DB.

Este panel debe ser de solo lectura.

---

## 14. Manejo de mapas

Map names detectados en el cluster:

```txt
TheIsland_WP
Ragnarok_WP
Extinction_WP
Astraeos_WP
Valguero_WP
Genesis_WP
Amissa_WP
```

Crear helper de display si el proyecto no lo tiene:

```txt
TheIsland_WP  → The Island
Ragnarok_WP   → Ragnarok
Extinction_WP → Extinction
Astraeos_WP   → Astraeos
Valguero_WP   → Valguero
Genesis_WP    → Genesis
Amissa_WP     → Amissa
```

---

## 15. Feature flags sugeridos

Agregar flags según patrón del proyecto:

```env
PLAYER_HUB_ENABLED=true
PLAYER_HUB_CONSTRUCTION_RANKING_ENABLED=false
DISCORD_OAUTH_ENABLED=true
```

Si no existe sistema de feature flags, usar configuración simple en backend.

---

## 16. Variables de entorno sugeridas

Adaptar nombres a la convención actual.

```env
DISCORD_CLIENT_ID=
DISCORD_CLIENT_SECRET=
DISCORD_REDIRECT_URI=
DISCORD_OAUTH_SCOPE=identify

TRACKER_DB_HOST=127.0.0.1
TRACKER_DB_PORT=3306
TRACKER_DB_USER=web_reader
TRACKER_DB_PASSWORD=
TRACKER_DB_NAME=info_web_tracker
FUSIONSHOP_DB_NAME=s2646_fusionshop

PLAYER_HUB_ENABLED=true
PLAYER_HUB_CONSTRUCTION_RANKING_ENABLED=false
```

No commitear secretos.

---

## 17. Orden de implementación recomendado

### Fase 0 — Análisis obligatorio

1. Inspeccionar estructura del proyecto.
2. Identificar stack real.
3. Identificar cómo se definen rutas frontend.
4. Identificar cómo se definen endpoints API.
5. Identificar cómo se maneja el admin panel.
6. Identificar cómo se manejan env vars.
7. Identificar componentes reutilizables.
8. Generar un breve plan antes de tocar archivos.

### Fase 1 — Capa DB/backend

1. Crear servicio/repository de solo lectura para Player Hub.
2. Implementar conexión segura a MariaDB usando patrón existente.
3. Implementar queries parametrizadas.
4. Implementar detección de tabla `building_stats_player`.
5. Implementar DTOs/respuestas normalizadas.

### Fase 2 — Auth Discord

1. Revisar si ya existe auth.
2. Si existe, reutilizarla.
3. Si no existe, implementar OAuth2 con scope `identify`.
4. Guardar sesión según patrón del proyecto.
5. Implementar `/me`.

### Fase 3 — API Player Hub

1. Endpoint `GET /api/player-hub/me`.
2. Endpoint `GET /api/player-hub/me/survivors`.
3. Endpoint `GET /api/player-hub/me/resources`.
4. Endpoint `GET /api/player-hub/rankings/farming`.
5. Endpoint `GET /api/player-hub/rankings/construction` con fallback si no existe tabla.

### Fase 4 — UI Perfil

1. Crear ruta `/perfil`.
2. Reutilizar layout/navbar/background actual.
3. Agregar botón de login Discord.
4. Mostrar estados: no logueado, no vinculado, vinculado sin stats, vinculado con stats.
5. Mostrar cards de survivors.
6. Mostrar stats básicas.
7. Mostrar top recursos.

### Fase 5 — UI Rankings

1. Crear o extender ruta `/rankings`.
2. Ranking de farmeo.
3. Filtro por recurso si es simple.
4. Ranking de construcción opcional.

### Fase 6 — Admin simple

1. Integrar en admin existente.
2. Crear búsqueda read-only.
3. Mostrar perfil resumido.
4. Proteger con permisos admin actuales.

### Fase 7 — QA y hardening

1. Probar con usuario vinculado.
2. Probar con usuario no vinculado.
3. Probar sin datos de stats.
4. Probar ranking con DB vacía.
5. Probar que no rompa páginas existentes.
6. Probar responsive mobile.
7. Verificar que no se expongan secretos ni EOS completos a usuarios normales.

---

## 18. Criterios de aceptación

### Perfil

- Un usuario puede entrar con Discord.
- Si su Discord está vinculado, ve sus survivors.
- Si no está vinculado, ve un mensaje claro.
- Si está vinculado pero sin stats, ve un mensaje claro.
- EOSID no se muestra completo a usuario normal.

### Rankings

- Ranking de farmeo carga desde `gathering_stats`.
- Ranking no rompe si no hay datos.
- Ranking de construcción no rompe si no existe `StructureStats`.

### Admin

- Admin puede buscar jugadores.
- Admin puede ver DiscordID, EOSID, survivors, tribu, mapa y stats.
- Usuario normal no puede acceder a endpoints admin.

### Integración

- Navbar, hero, background y estilos globales no son reemplazados.
- No se rompen rutas existentes.
- No se duplica backend ni admin panel.
- No hay conexión directa frontend → MariaDB.

---

## 19. Riesgos

| Riesgo | Mitigación |
|---|---|
| DiscordID pierde precisión en JS | Tratar siempre como string. |
| DB user sin permisos cross-schema | Crear usuario `web_reader` con SELECT en ambas DBs. |
| No existe StructureStats | Feature flag y detección de tabla. |
| Auth duplicada | Inspeccionar y reutilizar auth existente. |
| Romper admin actual | Integrar en módulo/ruta existente sin modificar núcleo. |
| Exponer EOSID completo | Mask para usuarios normales; completo solo admin. |
| SQL injection | Queries parametrizadas. |

---

## 20. No objetivos del MVP

No implementar todavía:

- Edición de linkeos.
- Desvinculación Discord/EOS.
- Gestión de puntos.
- Tienda web.
- Logros complejos.
- Temporadas con reset.
- Export CSV.
- App nativa Android/iOS.
- Escritura en tablas de ShadowBot o plugins.
- Modificar configs de ArkApi.

---

## 21. Entregables esperados del agente

1. Informe breve de arquitectura detectada.
2. Plan de archivos a tocar antes de modificar.
3. Backend/API implementado o PR parcial.
4. UI `/perfil`.
5. UI `/rankings` o extensión de ranking existente.
6. Admin read-only integrado.
7. Documentación breve de env vars.
8. Instrucciones de prueba local.
9. Lista de pendientes/no implementado.

---

## 22. Prompt inicial para el agente

Copiar y pegar este prompt en OpenCode IDE / agente GPT 5.4:

```txt
Necesito que analices este proyecto de la web Historias y Mates y diseñes la implementación del Player Hub MVP descrito en docs/PDR_Historias_y_Mates_Player_Hub_MVP.md.

Reglas críticas:
1. No crees una web nueva.
2. No reemplaces navbar, hero, background, fuentes, paleta, layout global ni panel admin existente.
3. Primero inspeccioná la arquitectura real del proyecto: frontend, backend/API, rutas, admin panel, componentes, estilos, auth y variables de entorno.
4. Antes de modificar archivos, devolveme un plan con:
   - stack detectado,
   - rutas existentes relevantes,
   - componentes reutilizables encontrados,
   - endpoints/API existentes relevantes,
   - propuesta de archivos nuevos/modificados,
   - riesgos de integración.
5. El MVP debe agregar:
   - Login con Discord o reutilización del login existente,
   - detección del EOSID desde s2646_fusionshop.discordlinks,
   - perfil del jugador en /perfil,
   - survivors por mapa,
   - stats básicas desde info_web_tracker.pvpve_stats,
   - recursos desde info_web_tracker.gathering_stats,
   - ranking de farmeo,
   - ranking de construcción opcional si existe building_stats_player,
   - panel admin simple read-only integrado al admin actual.
6. El frontend nunca debe conectarse directo a MariaDB. Todo debe pasar por el backend/API.
7. DiscordID y EOSID deben tratarse como string. No parsear DiscordID como number.
8. Usar queries parametrizadas y no exponer secretos.
9. Si StructureStats no está instalado, el ranking de construcción debe quedar oculto o mostrar estado “próximamente”, pero no debe romper la página.
10. Si ya existe auth/admin guard, reutilizalo en vez de duplicarlo.

Empezá solo con análisis y plan. No modifiques archivos todavía hasta mostrarme el plan.
```

---

## 23. Nota final

Este MVP debe sentirse como una función nativa de Historias y Mates, no como un módulo pegado con cinta. La prioridad es integración limpia, seguridad, lectura de datos confiable y componentes reutilizables.
