# Gestió de Comandes — Disseny del Sistema

**Versió**: 0.1  
**Data**: 2026-08-13  
**Estat**: Esborrany

---

## Índex

1. [Arquitectura de l'aplicació](#1-arquitectura-de-laplicació)
2. [Estats d'una comanda](#2-estats-duna-comanda)
3. [Flux de compra](#3-flux-de-compra)
4. [Gestió d'incidències](#4-gestió-dincidències)
5. [Gestió massiva d'incidències](#5-gestió-massiva-dincidències)
6. [Model de dades](#6-model-de-dades)
7. [Notificacions associades](#7-notificacions-associades)

---

## 1. Arquitectura de l'aplicació

| Capa | Tecnologia | Estat |
|---|---|---|
| **Backend** | Laravel 11 — API REST | ✅ Confirmat |
| **Frontend** | Vue 3 (SPA) | ✅ Confirmat |
| **PWA / Mòbil** | Ionic (sobre Vue) | ⏳ Opcional — Fase 3 |

### Comunicació frontend ↔ backend

```
Vue 3 (SPA)
    ↕ HTTP/JSON (Laravel Sanctum — token bearer)
Laravel 11 API
    ↕
MySQL + Redis (cues) + Serveis externs (LDAP, PICA, Redsys/Stripe)
```

- Tota la lògica de negoci viu al backend
- El frontend Vue consumeix l'API REST
- Si en el futur es vol Ionic, reutilitza la mateixa API sense canvis al backend

---

## 2. Estats d'una comanda

### Diagrama de transicions

```
                    ┌─────────────────┐
                    │  PENDENT_PAGAMENT│ ← comanda creada
                    └────────┬────────┘
                             │ professional inicia pagament
                    ┌────────▼────────┐
                    │  PAGAMENT_EN_CURS│ ← redirigit a passarel·la
                    └────────┬────────┘
               ┌─────────────┼─────────────┐
               │ OK          │ KO          │ timeout / error
    ┌──────────▼──────┐  ┌───▼──────┐  ┌──▼──────────────┐
    │   CONFIRMADA    │  │ PAGAMENT │  │ EXPIRADA        │
    │                 │  │ REBUTJAT │  │ (cleanup auto)  │
    └──────────┬──────┘  └───┬──────┘  └─────────────────┘
               │             │ professional pot reintentar
               │         ┌───▼──────┐
               │         │PENDENT_  │
               │         │PAGAMENT  │ (nova comanda)
               │         └──────────┘
               │
    ┌──────────▼──────┐
    │PENDENT_LLIURAMENT│ ← confirmada, esperant recollida
    └──────────┬──────┘
               │
        ┌──────┴──────┐
        │ incidència? │
   ┌────▼────┐   ┌────▼────────────┐
   │ENTREGADA│   │  AMB_INCIDÈNCIA │
   └─────────┘   └────────┬────────┘
                          │ resolta
                 ┌────────▼────────┐
                 │    RESOLTA      │ → pot derivar a ENTREGADA,
                 └─────────────────┘   CANCEL·LADA o REEMBORSADA

    Des de CONFIRMADA o PENDENT_LLIURAMENT:
    ┌─────────────────┐
    │   CANCEL·LADA   │ ← cancel·lació manual (gestor o professional)
    └────────┬────────┘
             │ si pagament ja realitzat
    ┌────────▼────────┐
    │   REEMBORSADA   │ ← devolució completada via passarel·la
    └─────────────────┘
```

### Taula d'estats

| Estat | Descripció | Qui pot transicionar |
|---|---|---|
| `pendent_pagament` | Comanda creada, esperant que el professional pagui | Sistema (expiració) |
| `pagament_en_curs` | Professional redirigit a la passarel·la | Sistema (callback) |
| `pagament_rebutjat` | Passarel·la ha rebutjat el pagament | Professional (reintent) |
| `expirada` | Comanda no pagada en >24h | Sistema (Job nocturn) |
| `confirmada` | Pagament confirmat, pendent de preparar | GESTOR, Sistema |
| `pendent_lliurament` | Preparada, disponible per recollir | GESTOR |
| `entregada` | Lliurada al professional | GESTOR |
| `amb_incidència` | Incidència oberta associada | GESTOR, ADMIN |
| `resolta` | Incidència tancada | GESTOR, ADMIN |
| `cancel·lada` | Cancel·lada (amb o sense pagament previ) | GESTOR, ADMIN, Professional* |
| `reemborsada` | Devolució completada | Sistema (API passarel·la) |

\* Professional només pot cancel·lar si la comanda és en `confirmada` i la finestra de cancel·lació no ha expirat (configurable).

---

## 3. Flux de compra

### 3.1 Precondicions (validades al backend)

Abans de permetre crear una comanda, el sistema verifica:

- [ ] Professional té accés actiu al servei (`actiu = true`)
- [ ] Fill associat existeix, és actiu i té menys de 3 anys
- [ ] Estem dins la finestra mensual (`minDiaComanda` ≤ avui ≤ `maxDiaComanda`)
- [ ] El professional no té ja una comanda del mateix mes per al mateix fill en estat `confirmada` o posterior
- [ ] No s'han superat els límits mensuals de producte
- [ ] El professional té punt de distribució assignat

### 3.2 Pas a pas

```
1. SELECCIÓ DE PRODUCTES (Vue)
   ├── Llista de productes disponibles (per subcategoria)
   ├── Indicador visual: "X de Y bolquers disponibles aquest mes"
   ├── Selector de fill (si té múltiples fills actius)
   └── Validació en temps real del límit mensual

2. RESUM I CONFIRMACIÓ (Vue)
   ├── Llista de productes seleccionats + preus
   ├── Import total
   ├── Punt de distribució assignat
   └── Botó "Procedir al pagament"

3. CREACIÓ DE COMANDA (Backend)
   ├── POST /api/comandes
   ├── Validació de precondicions (backend — no confiar en el frontend)
   ├── Crear registre comanda en estat [pendent_pagament]
   ├── Generar idempotency_key (UUID)
   └── Retornar paràmetres per a la passarel·la

4. REDIRECCIÓ A LA PASSAREL·LA (Vue → Redsys/Stripe)
   ├── Comanda passa a [pagament_en_curs]
   └── Professional completa el pagament externament

5. CALLBACK DE PAGAMENT (Backend — POST webhook)
   ├── Verificació HMAC-SHA256 (obligatori)
   ├── Comprovar idempotency_key no processat
   ├── DB::transaction:
   │   ├── Comanda → [confirmada]
   │   └── Registre pagament creat
   └── Job: email confirmació (asíncron)

6. RETORN DE L'USUARI (Vue)
   ├── Si pagament OK: pàgina "Comanda confirmada" amb detall
   └── Si error: pàgina "Pagament no completat" amb opcions (reintentar / contactar)
```

### 3.3 Gestió de reintentos de pagament

Si el pagament falla (`pagament_rebutjat`) o expira (`expirada`), el professional pot crear una nova comanda pel mateix mes. La comanda anterior queda en estat `pagament_rebutjat` o `expirada` (no s'esborra, queda al historial).

---

## 4. Gestió d'incidències

### 4.1 Tipus d'incidència

| Codi | Tipus | Descripció |
|---|---|---|
| `INC-PAG` | Pagament | Error de pagament, doble càrrec, import incorrecte |
| `INC-CAN` | Cancel·lació | El professional vol cancel·lar una comanda confirmada |
| `INC-ENT` | Lliurament | Retard, comanda no disponible al punt de distribució |
| `INC-PRO` | Producte | Producte equivocat, danyat o mancat |
| `INC-ALT` | Altres | Qualsevol incidència no categoritzada |

### 4.2 Estat d'una incidència

```
OBERTA → EN_GESTIÓ → RESOLTA
                  ↘ TANCADA_SENSE_RESOLUCIÓ (amb justificació)
```

| Estat | Descripció |
|---|---|
| `oberta` | Reportada, pendent d'assignar |
| `en_gestió` | Assignada a un gestor, en curs |
| `resolta` | Resolta amb acció aplicada |
| `tancada_sense_resolució` | Tancada sense acció (amb motiu documentat) |

### 4.3 Accions disponibles en resoldre una incidència

| Acció | Quan s'aplica | Efecte |
|---|---|---|
| `reembors_total` | Pagament incorrecte, cancel·lació | Devolució 100% via API passarel·la; comanda → `reemborsada` |
| `reembors_parcial` | Error parcial | Devolució d'import especificat |
| `re_lliurament` | Producte erroni o no rebut | Comanda torna a `pendent_lliurament` |
| `substitucio_producte` | Producte no disponible | Modifica línies de comanda, notifica professional |
| `extensio_finestra` | Problema de timing | Permet nova comanda fora de la finestra (per a aquest professional) |
| `tancar_sense_accio` | Incidència no procedent | Tanca amb justificació, sense canvis a la comanda |

### 4.4 Flux d'una incidència individual

```
1. OBERTURA (Professional o GESTOR)
   ├── Via formulari de l'app (professional) o panell admin (gestor)
   ├── Camps: tipus, descripció, comanda afectada
   └── Incidència creada en [oberta], comanda passa a [amb_incidència]

2. ASSIGNACIÓ (GESTOR)
   ├── GESTOR s'assigna la incidència
   └── Incidència → [en_gestió]

3. COMUNICACIÓ
   ├── Gestor pot afegir notes internes (no visibles al professional)
   ├── Gestor pot enviar missatge al professional (registrat a l'historial)
   └── Professional rep notificació per email

4. RESOLUCIÓ (GESTOR o ADMIN)
   ├── Selecciona acció de resolució
   ├── Afegeix comentari de resolució
   ├── Sistema executa l'acció (reembors via API, canvi d'estat comanda...)
   └── Professional notificat per email amb el resultat

5. REGISTRE
   └── Tot el cicle queda al log d'auditoria
```

---

## 5. Gestió massiva d'incidències

Per a situacions que afecten múltiples professionals simultàniament (retard de proveïdor, error de sistema, cancel·lació de lliurament d'un mes...).

### 5.1 Creació d'una incidència massiva

El ADMIN (i opcionalment el GESTOR) pot crear una **incidència massiva** definint:

**Criteri de selecció de comandes afectades:**

| Filtre | Exemple |
|---|---|
| Ubicació / centre | Totes les comandes del CAP Girona |
| Punt de distribució | Totes les del punt "Màquina Planta 2" |
| Mes de comanda | Totes les comandes de març 2025 |
| Estat de comanda | Totes les que estan en `pendent_lliurament` |
| Producte | Totes les que inclouen "Bolquer Talla 4" |
| Combinació | Ubicació + mes + estat |

**Previsualització abans d'aplicar:**
- Nombre de comandes afectades
- Nombre de professionals afectats
- Llistat exportable per revisar

### 5.2 Accions massives disponibles

| Acció | Descripció | Confirmació requerida |
|---|---|---|
| `notificació_massiva` | Enviar email informatiu a tots els afectats | Sí — revisar text |
| `reembors_massiu` | Devolució automàtica a tots els afectats | Sí — doble confirmació |
| `cancel·lació_massiva` | Cancel·lar totes les comandes seleccionades | Sí — doble confirmació |
| `extensió_finestra_massiva` | Obrir nova finestra de comanda per als afectats | Sí |
| `re_lliurament_massiu` | Tornar totes a `pendent_lliurament` | Sí |

### 5.3 Personalització per casos concrets dins d'una incidència massiva

Una incidència massiva pot tenir **excepcions individuals**:

```
Incidència massiva: "Retard lliurament març — Punt distribució Planta 2"
├── Acció general: notificació a tots + extensió finestra
└── Excepcions individuals:
    ├── Professional A: ja ha recollit → marcar com entregada manualment
    ├── Professional B: vol cancel·lar → processar reembors individual
    └── Professional C: producte equivocat → re_lliurament amb substitució
```

Cada excepció individual genera una sub-incidència vinculada a la massiva, amb el seu propi flux de resolució i historial.

### 5.4 Exemple de flux complet d'incidència massiva

```
Situació: el proveïdor avisa que el lliurament de abril quedarà ajornat 2 setmanes.

1. ADMIN crea incidència massiva
   Filtre: mes_comanda = 2025-04 + estat = pendent_lliurament
   → Sistema detecta: 87 comandes, 62 professionals afectats

2. ADMIN previsualitza i confirma

3. Sistema executa:
   ├── Totes les comandes → [amb_incidència]
   ├── Incidència individual creada per a cada comanda, vinculada a la massiva
   └── Job: email a 62 professionals (asíncron, amb throttling)
       "El lliurament d'abril s'ha ajornat. Nova data estimada: 15/05/2025."

4. A mesura que arriben els lliuraments:
   ├── GESTOR marca comandes com a entregades individualment
   └── O executa "re_lliurament_massiu" quan el proveïdor confirma disponibilitat

5. Si 3 professionals sol·liciten cancel·lació:
   └── GESTOR crea excepció individual per a cada un → reembors individual

6. Incidència massiva es tanca quan totes les sub-incidències estan resoltes
   └── ADMIN pot forçar tancament amb resum si queden casos pendents documentats
```

---

## 6. Model de dades

### Taula `comandes`

```
comandes
├── id
├── idProfessionalBolquer     FK → professionals_bolquers
├── idFillProfessional        FK → fills_professional
├── idPuntDistribucio         FK → punts_distribucio
├── idPagament                FK → pagaments (nullable fins confirmació)
├── estat                     enum (vegeu secció 2)
├── mes_comanda               VARCHAR(7) — format YYYY-MM
├── preu_total
├── idempotency_key           UNIQUE — UUID
├── data_comanda
├── data_confirmacio          nullable
├── data_lliurament           nullable
├── cancel·lada_per           nullable — FK → professionals (qui ha cancel·lat)
├── motiu_cancelacio          nullable
└── timestamps
```

### Taula `linies_comanda`

```
linies_comanda
├── id
├── idComanda                 FK → comandes
├── idProducte                FK → productes
├── quantitat
├── preu_unitari
└── timestamps
```

### Taula `pagaments`

```
pagaments
├── id
├── idComanda                 FK → comandes
├── passarela                 enum [redsys|stripe]
├── gateway_order_id          UNIQUE — id de l'ordre a la passarel·la
├── import
├── estat                     enum [pendent|completat|rebutjat|reemborsat]
├── referencia_passarela      resposta de la passarel·la
├── data_pagament             nullable
├── data_reembors             nullable
└── timestamps
```

### Taula `incidencies`

```
incidencies
├── id
├── idComanda                 FK → comandes (nullable per a massives sense comanda concreta)
├── idIncidenciaMassiva       FK → incidencies_massives (nullable)
├── idProfessionalReporta     FK → professionals
├── idGestorAssignat          FK → professionals (nullable)
├── tipus                     enum [INC-PAG|INC-CAN|INC-ENT|INC-PRO|INC-ALT]
├── descripcio
├── estat                     enum [oberta|en_gestió|resolta|tancada_sense_resolució]
├── accio_resolucio           nullable — enum d'accions
├── comentari_resolucio       nullable
├── data_obertura
├── data_resolucio            nullable
└── timestamps
```

### Taula `incidencies_massives`

```
incidencies_massives
├── id
├── idAdminCreador            FK → professionals
├── descripcio
├── criteri_filtre            JSON — filtre aplicat per seleccionar comandes
├── total_comandes_afectades
├── total_professionals_afectats
├── accio_general             enum d'accions massives
├── estat                     enum [activa|parcial|tancada]
├── data_creacio
├── data_tancament            nullable
└── timestamps
```

### Taula `comunicacions_incidencia`

```
comunicacions_incidencia
├── id
├── idIncidencia              FK → incidencies
├── idAutor                   FK → professionals
├── tipus                     enum [nota_interna|missatge_professional]
├── missatge
└── created_at
```

---

## 7. Notificacions associades

| Event | Destinatari | Canal |
|---|---|---|
| Comanda confirmada | Professional | Email |
| Pagament rebutjat | Professional | Email |
| Incidència oberta (per professional) | GESTOR (mail gestió) | Email |
| Incidència assignada | Professional | Email |
| Missatge de gestor en incidència | Professional | Email |
| Incidència resolta | Professional | Email amb detall de l'acció |
| Notificació massiva | Professionals afectats | Email (Job amb throttling) |
| Reembors processat | Professional | Email amb confirmació import |
| Comanda llesta per recollir | Professional | Email (opcional, pendent P-NO03) |
