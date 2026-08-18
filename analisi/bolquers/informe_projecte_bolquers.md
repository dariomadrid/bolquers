# Informe de Projecte — Aplicació Bolquers

**Data**: 2026-08-14 · **Estat**: Esborrany pendent de validació  
**Organisme**: ICS (Hospital Universitari Dr. Josep Trueta) + IAS (Institut d'Assistència Sanitària)

---

## 1. Què és i per què

L'**Aplicació Bolquers** és un sistema independent de gestió del servei de distribució de productes d'higiene infantil (bolquers, tovalloletes, empapadors) per al personal hospitalari d'ICS i IAS amb fills menors de 3 anys.

**Problema actual**: el servei és avui un mòdul dins `cafetiquet` (Laravel 4.2, EOL 2015), amb vulnerabilitats de seguretat documentades, sense auditoria i sense possibilitat de manteniment. Queda **congelat**; el nou projecte és una aplicació totalment independent.

**Vulnerabilitats de l'aplicació actual a no repetir:**

| Problema | Causa |
|---|---|
| Verificació de pagament inoperativa | `$trobat = true` hardcoded |
| Credencials FTP en codi font | Secrets sense gestió d'entorn |
| Bypass d'autenticació per NIF | Login sense contrasenya |
| Ruta administrativa pública | `/recalculaCIPS` sense auth |
| Callback de pagament GET amb dades sensibles | Disseny de ruta incorrecte |
| Sense log d'auditoria | Cap traçabilitat |
| Sense rols diferenciats | Tot o res |

---

## 2. Cicle complet del servei

```
Alta professional → Aprovació gestor → Alta fill (validació GT o document)
        ↓
Compra (finestra 1–20 mes) → Pagament en línia
        ↓
Gestor agrupa comandes → Compra al proveïdor (IAS)
        ↓
Recepció → Distribució als punts → Lliurament presencial
        ↓
               Auditoria completa de cada pas
```

---

## 3. Actors i rols

| Rol | Funció |
|---|---|
| **PROFESSIONAL** | Usuari final: fa comandes, gestiona fills |
| **GESTOR_DISTRIBUCIO** | Recepciona productes i registra lliuraments per punt |
| **GESTOR_COMPRA** | Tramita la compra global al proveïdor (IAS) |
| **ADMIN** | Configuració global, rols, col·lectius, auditoria |
| **SUPERVISOR** | Consulta d'informes i estadístiques (sense edició) |

**Col·lectius**: DT (Trueta/ICS), DT Externs, IAS, IAS Externs — tots amb autenticació local (bcrypt + Sanctum). No hi ha integració AD/LDAP ni WS SOAP.

---

## 4. Funcionalitats principals

### Gestió d'usuaris i fills
- Alta de professional amb sol·licitud pendent → aprovació/rebuig pel gestor
- **Validació de fills**: personal de l'àmbit (DT, IAS) via **GT** (aplicació RRHH hospitalària); personal extern via **document acreditatiu** (upload → validació gestor → eliminació RGPD); fallback manual excepcional
- CIP del fill: `substr(cognom1,0,2) + substr(cognom2,0,2) + sexe + any + mes + dia`
- Fills elegibles fins als 3 anys; baixa automàtica via Job diari
- Un fill pot tenir co-progenitors (tots poden fer comandes)

### Compra i pagament
- Finestra mensual configurable (exemple: dies 1–20)
- Passarel·la de pagament: Redsys (preferida) o Stripe — **pendent decisió**
- Flux segur: `idempotency_key` + `DB::transaction` + webhook HMAC + Job de neteja

### Workflow de comandes

```
pendent_pagament → pagada → pendent_compra → comprada → rebuda → pendent_entrega → entregada
                                                    ↓
                                           falta_estoc → gestió incidència
cancel·lada (des de pendent_pagament o pagada)
```

### Altres
- Catàleg de productes per col·lectiu, amb historial de canvis i clonació
- Punts de distribució: alta, assignació per professional, validació QR (Fase 2)
- Devolucions parcials per línia de comanda (Fase 2)
- Notificacions email parametritzables per tots els events clau
- Informes: fulls de recollida, estoc per període, exportació Excel/PDF
- `audit_log` immutable: cobertura completa (login, pagaments, validacions, lliuraments, configuració)

---

## 5. Arquitectura i stack tecnològic

```
┌─────────────────────────────────────────────────────┐
│  NAVEGADOR                                          │
│  Vue 3 SPA  ←→  API REST (JSON + Sanctum token)    │
└─────────────────────────────────────────────────────┘
              ↕ HTTPS
┌─────────────────────────────────────────────────────┐
│  BACKEND (DMZ hospitalària)                         │
│  Laravel 11 · PHP 8.2+                              │
│  MySQL / MariaDB (BD independent)                   │
│  Laravel Queue (driver DB)                          │
│  Filesystem / MinIO (URLs signades TTL)             │
└─────────────────────────────────────────────────────┘
              ↕
┌─────────────────────────────────────────────────────┐
│  SERVEIS EXTERNS                                    │
│  GT (RRHH hospitalari) — Fase 2                    │
│  Passarel·la de pagament (Redsys / Stripe)         │
└─────────────────────────────────────────────────────┘
```

**Principis**: cap valor hardcoded, API REST per a tot, serveis externs encapsulats amb timeout i fallback, FTP eliminat, documents via URLs signades.

---

## 6. Integracions externes

| Sistema | Funció | Estat |
|---|---|---|
| **GT** (RRHH hospitalari) | Validació de fills (àmbit) + verificació d'actiu | Fase 2 — pendent DEC-06 |
| **Passarel·la de pagament** | Pagament + devolucions | Redsys o Stripe — pendent DEC-01 |

> AD/LDAP i WS SOAP d'autenticació **descartats**: tots els col·lectius usen autenticació local.

---

## 7. Decisions clau pendents

| ID | Decisió | Impacte |
|---|---|---|
| **DEC-01** | Passarel·la: Redsys o Stripe | Disseny del flux de pagament i devolucions |
| **DEC-02** | Límits de compra: per professional o per fill | Model de dades |
| **DEC-03** | Cost del servei (gratuït, cofinançat, per col·lectiu) | Flux de pagament sencer |
| **DEC-04** | Redsys suporta devolucions via API | Gestió d'incidències |
| **DEC-05** | Qui tramita devolucions parcials: gestor IAS o Trueta | Workflow d'incidències |
| **DEC-06** | Viabilitat integració GT (fills + actiu) | Validació i baixa automàtica |
| **DEC-07** | Comandes pendents a mes nou sense lliurar | Cicle de vida de comandes |
| **DEC-09** | Validació QR a recepció/entrega | Interfície de gestors |

---

## 8. Diagrames de referència

Els diagrames següents estan disponibles com a fitxers HTML/SVG autònoms:

| Diagrama | Fitxer | Contingut |
|---|---|---|
| **Arquitectura** | `arquitectura.html` | Zones: navegador · backend (Laravel, MySQL, Queue, Storage) · serveis externs |
| **Cicle de vida de la comanda** | `cicle_comanda.html` | Swimlane: Professional · Passarel·la · Gestor Distribució · Gestor Compra |
| **Model de dades (ER)** | `model_dades.html` | Entitats: professionals, fills, comandes, pagaments, productes, punts, audit_log |
| **Màquina d'estats de la comanda** | `estats_comanda.html` | Tots els estats i transicions, incloent cancel·lació i falta d'estoc |
| **Flux de validació de fills** | `validacio_fills.html` | 2 camins (àmbit via GT / externs via document) + fallback manual |

---

## 9. Roadmap

### Fase 1 — Fonaments (~8 set. sense IA · ~3,7 set. amb Claude Code)
Base completa equivalent a l'aplicació actual: auth local, rols, professionals, fills, catàleg, compra, pagament, comandes, punts, notificacions, auditoria, seguretat, tests i frontend bàsic.

### Fase 2 — Millores (~4,5 set. sense IA · ~2 set. amb Claude Code)
Integració GT, devolucions, incidències massives, QR, informes avançats, accions massives, observabilitat.

### Fase 3 — Mòbil, opcional (~1,5 set. sense IA · ~0,5 set. amb Claude Code)
PWA instal·lable, notificacions push, offline parcial, Ionic opcional.

---

## 10. Estimació d'hores

### Detall per mòdul

#### Fase 1 — Fonaments

| Mòdul | Sense IA | Amb Claude Code |
|---|---:|---:|
| Infraestructura base | 16 h | 7 h |
| Autenticació (local bcrypt + Sanctum) | 12 h | 5 h |
| Rols i permisos | 10 h | 4 h |
| Gestió de professionals | 24 h | 10 h |
| Gestió de fills + validació | 32 h | 14 h |
| Catàleg de productes | 16 h | 6 h |
| Compra i pagament (passarel·la + webhook) | 44 h | 24 h |
| Workflow de comandes | 32 h | 14 h |
| Punts de distribució | 10 h | 4 h |
| Notificacions email | 14 h | 6 h |
| Auditoria immutable | 16 h | 6 h |
| Seguretat | 14 h | 8 h |
| Tests Fase 1 | 32 h | 13 h |
| Frontend Vue 3 — Fase 1 | 60 h | 26 h |
| **Subtotal Fase 1** | **332 h** | **147 h** |

#### Fase 2 — Millores funcionals

| Mòdul | Sense IA | Amb Claude Code |
|---|---:|---:|
| Integració GT (RRHH) | 28 h | 16 h |
| Devolucions parcials | 24 h | 12 h |
| Incidències massives | 18 h | 8 h |
| QR entrega | 12 h | 5 h |
| Informes avançats + exportació | 24 h | 10 h |
| Accions massives | 14 h | 6 h |
| Observabilitat | 8 h | 3 h |
| Tests Fase 2 | 18 h | 7 h |
| Frontend Vue 3 — Fase 2 | 36 h | 15 h |
| **Subtotal Fase 2** | **182 h** | **82 h** |

#### Fase 3 — Mòbil (opcional)

| Mòdul | Sense IA | Amb Claude Code |
|---|---:|---:|
| PWA | 14 h | 6 h |
| Notificacions push | 12 h | 5 h |
| Offline parcial | 8 h | 3 h |
| Ionic (opcional) | 18 h | 8 h |
| **Subtotal Fase 3** | **52 h** | **22 h** |

### Resum global

| | Sense IA | Amb Claude Code | Estalvi |
|---|---:|---:|---:|
| **Fase 1** — Fonaments | 332 h · ~8 set. | **147 h · ~3,7 set.** | ~56% |
| **Fase 2** — Millores | 182 h · ~4,5 set. | **82 h · ~2 set.** | ~55% |
| **Fase 3** — Mòbil | 52 h · ~1,5 set. | **22 h · ~0,5 set.** | ~58% |
| **Total F1 + F2** | **514 h · ~13 set.** | **229 h · ~5,7 set.** | **~55%** |
| **Total 3 fases** | **566 h · ~14 set.** | **251 h · ~6,3 set.** | **~56%** |

> Estimació per a **1 desenvolupador full-stack** amb experiència en Laravel i Vue 3. Inclou disseny, implementació, tests i revisió. Exclou desplegament a producció i formació d'usuaris.

### Factor d'estalvi per tipus de tasca (amb Claude Code)

| Tipus | Estalvi | Motiu |
|---|---|---|
| Scaffold, migrations, CRUD | 60–70% | Patrons repetitius |
| Tests unitaris i feature | 55–65% | Generació automàtica |
| Frontend Vue (components) | 50–60% | Estructures predecibles |
| Lògica de negoci complexa | 35–45% | Supervisió humana necessària |
| Integració GT | 25–35% | Depèn de documentació externa |
| Seguretat i criptografia | 20–30% | Revisió humana obligatòria |

> ⚠️ Les hores "Amb Claude Code" representen **temps de l'enginyer** (supervisió, revisió, decisions de disseny), no temps del model. La revisió crítica és imprescindible en pagament, seguretat i integracions hospitalàries.

---

## 11. Riscos principals

| Risc | Probabilitat | Impacte | Mitigació |
|---|---|---|---|
| Devolucions API Redsys no suportades | Mitjana | Alt | Stripe com alternativa; procés manual de fallback |
| Integració GT no disponible | Mitjana | Mig | Validació documental per a àmbit; baixa manual |
| Volum comandes superior al previst | Baixa | Mig | Disseny escalable; cues asíncrones |
| Canvis normativa RGPD dades sanitàries | Baixa | Alt | DPO implicat des de l'inici; capa modular de dades personals |
