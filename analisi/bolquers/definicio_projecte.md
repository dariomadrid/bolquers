# Definició de Projecte — Aplicació Bolquers

**Versió**: 0.1  
**Data**: 2026-08-14  
**Estat**: Esborrany — pendent de validació  
**Autor**: Equip de desenvolupament  

---

## Índex

1. [Visió general](#1-visió-general)
2. [Context i origen](#2-context-i-origen)
3. [Abast del projecte](#3-abast-del-projecte)
4. [Actors i rols](#4-actors-i-rols)
5. [Funcionalitats principals](#5-funcionalitats-principals)
6. [Arquitectura i stack tecnològic](#6-arquitectura-i-stack-tecnològic)
7. [Integracions externes](#7-integracions-externes)
8. [Restriccions i condicionants](#8-restriccions-i-condicionants)
9. [Decisions pendents clau](#9-decisions-pendents-clau)
10. [Riscos identificats](#10-riscos-identificats)
11. [Roadmap proposat](#11-roadmap-proposat)
12. [Estimació d'hores de desenvolupament](#12-estimació-dhores-de-desenvolupament)

---

## 1. Visió general

L'**Aplicació Bolquers** és un sistema independent de gestió del servei de distribució de productes d'higiene infantil (bolquers, tovalloletes, empapadors) destinat al personal hospitalari dels organismes **ICS** (Institut Català de la Salut, Hospital Universitari de Girona Dr. Josep Trueta) i **IAS** (Institut d'Assistència Sanitària) que tingui fills menors de 3 anys.

El sistema gestiona tot el cicle: **alta del professional → validació de fills → compra periòdica → pagament → distribució → lliurament presencial**, amb total traçabilitat de cada operació.

---

## 2. Context i origen

### 2.1 Situació actual

El servei de bolquers és avui un mòdul dins l'aplicació `cafetiquet` (Laravel 4.2, EOL 2015), compartint base de dades i infraestructura amb el servei de cafeteria. Aquesta arquitectura presenta problemes greus de mantenibilitat, seguretat i escalabilitat.

### 2.2 Decisió adoptada

L'aplicació `cafetiquet` queda **congelada**. El projecte consisteix en una **nova aplicació independent**, construïda des de zero, que assumeix únicament la funcionalitat de bolquers. `cafetiquet` s'usa exclusivament com a **referència funcional** per identificar el comportament actual i les regles de negoci existents.

### 2.3 Problemes de l'aplicació anterior que no s'han de repetir

| Problema | Causa |
|---|---|
| Verificació de pagament inoperativa | `$trobat = true` hardcoded al callback |
| Credencials FTP en codi font | Secrets no gestionats per entorn |
| Bypass d'autenticació per NIF | Camí de login sense contrasenya |
| Ruta administrativa sense autenticació | `/recalculaCIPS` pública |
| Callback de pagament GET amb dades sensibles | Disseny de ruta incorrecte |
| Clau de passarel·la en text pla a BD | Absència de gestió de secrets |
| Sense transaccions DB al flux de pagament | Risc d'inconsistència en errors |
| Sense log d'auditoria | Cap traçabilitat de cap operació |
| Sense rols diferenciats | Tot o res amb `admBolquers` |
| Comptador `numFillsActius` mai decrementat | Lògica de manteniment absent |

---

## 3. Abast del projecte

### 3.1 Inclòs

- Gestió completa del cicle de vida del professional (alta, validació, baixa, recuperació de compte)
- Gestió de fills: alta, validació, límits per edat i ampliacions
- Catàleg de productes per col·lectiu, amb historial de canvis
- Compra periòdica amb finestra mensual configurable i pagament en línia
- Gestió del flux de comandes: des de la compra fins a l'entrega presencial
- Gestió de punts de distribució
- Devolucions totals i parcials (a nivell de línia de comanda)
- Gestió d'incidències individuals i massives
- Notificacions per email parametritzables
- Informes i exportacions (Excel, PDF)
- Log d'auditoria complet i immutable
- Panell d'administració per a gestors i administradors

### 3.2 Exclòs

- Servei de cafeteria i qualsevol altra funcionalitat de `cafetiquet`
- Gestió comptable o ERP
- Integració directa amb el sistema de nòmines (només verificació d'actiu via RRHH)
- App nativa (iOS/Android) — possible en una fase posterior via PWA/Ionic

---

## 4. Actors i rols

### 4.1 Rols del sistema

| Rol | Descripció |
|---|---|
| **ADMIN** | Administrador global. Gestió de configuració, rols, col·lectius, productes i auditoria. |
| **GESTOR_COMPRA** | Gestor de l'IAS. Tramita la compra global al proveïdor, valida recepció i distribueix als punts. |
| **GESTOR_DISTRIBUCIO** | Gestor d'un punt de distribució. Agrupa comandes, recepciona productes i registra lliuraments. |
| **PROFESSIONAL** | Usuari final del servei. Fa comandes i gestiona els seus fills. |
| **SUPERVISOR** | Perfil de consulta avançada. Accés a informes i estadístiques, sense capacitat d'edició. |

> Un professional pot tenir simultàniament el rol GESTOR i ser usuari del servei (tenir fills i fer comandes).

### 4.2 Col·lectius de professionals

| Col·lectiu | Descripció | Autenticació |
|---|---|---|
| **DT** | Personal Hospital Universitari Dr. Josep Trueta (ICS) | Local (bcrypt) |
| **DT Externs** | Personal extern associat a Trueta (banc sang, ICO, etc.) | Local (bcrypt) |
| **IAS** | Personal de l'Institut d'Assistència Sanitària | Local (bcrypt) |
| **IAS Externs** | Personal extern associat a IAS | Local (bcrypt) |

> **Decisió adoptada**: tots els col·lectius s'autentiquen localment (bcrypt + Sanctum). No hi ha integració amb AD/LDAP ni WS SOAP d'autenticació. La identitat del professional es verifica en el moment del registre (document acreditatiu o validació manual pel gestor), no en cada login.

> Els col·lectius que poden registrar-se al servei es defineixen als paràmetres de l'aplicació.

### 4.3 Perfils de compra

| Perfil | Descripció | Condicions |
|---|---|---|
| **Progenitor** | Professional amb fill menor de 3 anys | Qualsevol col·lectiu actiu |
| **Compra especial** | Accés a productes específics (empapadors, etc.) sense fill registrat | Restringit a col·lectiu IAS (ampliable per paràmetre) |

---

## 5. Funcionalitats principals

### 5.1 Autenticació i gestió d'usuaris

**Mètode d'autenticació:**

```
┌──────────────────────────────┐
│      Capa d'autenticació     │
├──────────────────────────────┤
│         LOCAL (bcrypt)       │
│  Tots els rols i col·lectius │
│  ADMIN, GESTOR, PROFESSIONAL │
└──────────────────────────────┘
          ↓ genera ↓
     Token Sanctum (sessió local)
```

> La verificació de la identitat del professional (que pertany al col·lectiu declarat i té el fill indicat) es realitza **en el moment del registre** (alta pendent → aprovació del gestor), no en cada login.

**Workflow d'alta de professional:**

1. El professional emplena el formulari d'alta
2. El sistema demana col·lectiu (DT, IAS, DT externs, IAS externs…)
3. El sistema demana perfil de compra (progenitor / compra especial)
4. Si **progenitor**: s'afegeix el fill (CIP + data naixement); si és extern, cal document acreditatiu
5. Si **compra especial**: validació que el col·lectiu ho permet
6. La sol·licitud queda en estat `pendent` fins que el gestor del centre la resol
7. El gestor aprova o rebutja amb comentari; el professional rep notificació en ambdós casos
8. Si rebutjada: la petició es cancel·la amb motiu i s'indica que torni a intentar-ho

**Altres regles d'usuari:**
- Cada gestor s'assigna a un centre; la vista de gestors filtra per centre, data sol·licitud i professional
- Si el professional es dona de baixa, els fills associats es donen de baixa lògica (prèvia comprovació que no pertanyen a un altre professional)
- La verificació que el professional segueix en actiu es fa via integració GT (Fase 2, DEC-06) o, en absència d'aquesta, manualment pel gestor
- Es facilita la recuperació del compte si el professional ja estava donat d'alta anteriorment

---

### 5.2 Gestió de fills

- El **CIP** és l'identificador del fill: `substr(cognom1,0,2) + substr(cognom2,0,2) + sexe + any(2d) + mes + dia`
  - Té relació amb el CIP Catsalut (pendent de confirmar si s'ha de sincronitzar)
- Un fill pot estar associat a **varis professionals** (co-progenitors); cadascun pot gestionar les comandes
- El fill és elegible mentre tingui **menys de 3 anys** (1.095 dies)
- Es pot sol·licitar **ampliació del límit** amb justificació del progenitor; pendent d'aprovació per gestor
- No hi ha límit de fills per professional
- La **baixa automàtica per edat** s'executa via Job programat diari
- Si el fill és d'un professional extern, cal document acreditatiu (upload, validació per gestor, eliminació post-validació per RGPD)

---

### 5.3 Catàleg de productes

- Vista de productes amb filtres i manteniment complet (preu, descripció, imatges, col·lectius als quals pertany)
- Preu únic per producte (possibilitat de preus per col·lectiu, parametritzable)
- Funcionalitat de **clonar un producte** per facilitar la creació de variants
- La descripció del producte es desa a la línia de comanda en el moment de la compra (historial)
- **Historial de canvis de producte** per traçabilitat

---

### 5.4 Compra i pagament

**Finestra de compra:**
- Del dia 1 al 20 del mes (parametritzable per aplicació)
- Pot haver-hi mesos sense finestra oberta (vacances, tancaments)
- Durant la finestra, l'usuari pot modificar o cancel·lar la seva comanda
- Fora de la finestra, no es poden fer noves comandes

**Passarel·la de pagament:**
- Opcions avaluades: **Redsys** (preferida si suporta devolucions via API) o **Stripe**
- Decisió pendent de confirmar amb Tiscar (IAS) i de verificar les capacitats de l'API de Redsys
- El professional rep **rebut per email** un cop confirmat el pagament

**Flux de pagament segur:**

```
1. Professional inicia comanda
   → Comanda creada en estat [pendent_pagament]
   → idempotency_key generat (UUID)

2. Professional completa pagament a la passarel·la

3. Passarel·la notifica webhook (POST)
   → Verificació signatura HMAC
   → Idempotència: rebutjar si ja processat
   → DB::transaction: comanda confirmada + pagament registrat
   → Job: email de confirmació (asíncron)

4. Navegador rep confirmació (independent del webhook)

5. Job programat: neteja comandes en [pendent_pagament] >24h
```

---

### 5.5 Workflow de comandes

**Visió general del flux:**

```
Usuari client                   Gestor Distribució            Gestor Compra (IAS)
──────────────                  ──────────────────            ───────────────────
Compra productes                                              
(finestra 1-20 mes)             
↓                               
[pendent_pagament]              
↓ (pagament OK)                 
[pagada]                        
                                Agrupa comandes               
                                del període                   
                                ↓                             
                                Tramita al Gestor Compra      
                                                              Fa compra agrupada
                                                              al proveïdor
                                                              (números absoluts,
                                                              sense distinció
                                                              d'origen/destinació)
                                                              ↓
                                                              Proveïdor entrega
                                                              ↓
                                                              Valida recepció
                                                              ↓
                                                              Distribueix als
                                                              punts de distribució
                                Recepciona productes          
                                ↓                             
                                [pendent_entrega]             
                                → Email a usuari              
↓                               
Usuari recull comanda           
presencialment                  
↓                               
[entregada]                     
```

**Estats de la comanda:**

```
pendent_pagament → pagada → pendent_compra → comprada → rebuda → pendent_entrega → entregada
                                                              ↓
                                                       rebuda_amb_incidència
```

**Excepcions i incidències:**

- **Falta d'estoc o canvi de preu**: la línia de comanda afectada es marca com `falta_estoc`; es busquen totes les comandes afectades i s'inicien devolucions parcials
- **Error de pagament**: la comanda no es processa per la compra
- **Gestió massiva d'incidències**: vista filtrable per estat/error, amb botó d'enviament massiu o individual d'emails explicant el procediment (devolució total, parcial, etc.)
- Tota comunicació amb l'usuari (telèfon, email) queda registrada

**Devolucions:**
- A nivell de **línia de comanda** (no necessàriament la comanda sencera)
- Automàtiques via API de la passarel·la si és possible, o manuals
- Notificació per email al professional afectat

---

### 5.6 Punts de distribució

- Alta, edició i baixa de punts de distribució (nom, localització, horaris d'entrega)
- El professional pot triar entre els punts de distribució disponibles per al seu centre
- El gestor del punt recepciona els productes i registra el lliurament a l'usuari
- Possibilitat de validació via **codi QR** per facilitar la recepció i l'entrega (a avaluar)

---

### 5.7 Accions massives i exportació

Les vistes de gestió permeten accions massives sobre:

| Entitat | Accions massives |
|---|---|
| Usuaris | Desactivació, canvi de col·lectiu, notificació... |
| Comandes | Canvis d'estat, enviament d'emails, devolucions... |
| Productes | Activació/desactivació, canvi de preu... |
| Punts de distribució | Activació/desactivació... |

Exportació disponible en **Excel, PDF i email** per a: comandes, productes, usuaris, estadístiques.

---

### 5.8 Notificacions per email

**Al professional** (parametritzable per aplicació):

| Moment | Contingut |
|---|---|
| Sol·licitud d'alta aprovada/rebutjada | Resultat + motiu si rebutjada |
| Comanda confirmada | Rebut, productes, import, punt de distribució |
| Comanda disponible per recollir | Localització, horari |
| Incidència (devolució, retard, falta d'estoc) | Explicació del procediment |
| Baixa automàtica del fill per edat | Avís previ i notificació de la baixa |

**Al gestor** (parametritzable per aplicació):

| Moment | Contingut |
|---|---|
| Nova sol·licitud d'alta pendent | Professional i centre |
| Tancament de finestra de compra | Resum de comandes del període |
| Informes mensuals | Estadístiques, fulls de recollida |

---

### 5.9 Informes i estadístiques

- **Fulls de recollida** per punt de distribució, generats sota demanda
- **Estoc gastat** per període, producte i col·lectiu
- Enviament automàtic periòdic (periodicitat parametritzable)
- Accés per perfil: GESTOR, ADMIN, SUPERVISOR

---

### 5.10 Traçabilitat i auditoria

Taula `audit_log` **immutable** (només inserció, cap rol pot modificar-la):

| Event registrat | Informació capturada |
|---|---|
| Login / logout (ok i ko) | IP, mètode d'autenticació |
| Alta / baixa / modificació de professional | Dades abans/després |
| Alta / baixa / modificació de fill | Dades abans/després |
| Validació (accés, fill, ampliació) | Resultat, comentari del gestor |
| Inici / confirmació / error de pagament | Import, `gateway_order_id` |
| Accés i eliminació de documents | Nom del fitxer, motiu |
| Canvis de configuració | Dades abans/després |
| Assignació / revocació de rols | Qui assigna, a qui, quin rol |
| Lliurament de comandes | Qui lliura, quines comandes |
| Comunicacions amb l'usuari | Canal, contingut, resultat |

---

## 6. Arquitectura i stack tecnològic

### 6.1 Stack confirmat

| Capa | Tecnologia | Notes |
|---|---|---|
| **Backend** | Laravel 11 (PHP 8.2+) — API REST | LTS actiu, Eloquent + Jobs + Sanctum natius |
| **Frontend** | Vue 3 (SPA) | Consumeix l'API REST via HTTP/JSON amb token Sanctum |
| **PWA / Mòbil** | Ionic (sobre Vue) | Opcional — Fase 3 |
| **Base de dades** | MySQL / MariaDB | BD pròpia, cap taula compartida amb cafeteria |
| **Cues** | Laravel Queue (driver DB) | Suficient per al volum previst |
| **Emmagatzematge** | Filesystem local o S3-compatible (MinIO) | Accés per URL signada amb TTL |
| **Autenticació** | Laravel Sanctum (tokens bearer) | Tots els mètodes d'auth generen token local |

### 6.2 Principis d'arquitectura

- **Aplicació totalment independent** de `cafetiquet` — cap dependència de codi ni de BD
- **API REST** per a totes les operacions (frontend desacoblat)
- **Cap valor hardcoded**: tots els paràmetres d'entorn a `.env`; en producció, secret manager
- **FTP eliminat**: els documents es gestionen via filesystem/S3 amb URLs signades
- **Serveis hospitalaris encapsulats** darrere d'una capa de servei amb interfície, timeout explícit i fallback definit
- **DMZ corporativa**: l'aplicació ha de poder accedir a serveis interns (GT, WS DA, etc.) des de la DMZ hospitalària

### 6.3 Jerarquia de configuració

```
CONFIGURACIÓ GLOBAL (ADMIN)
        ↓ sobreescrita per
CONFIGURACIÓ PER UBICACIÓ (ADMIN)
        ↓ sobreescrita per
CONFIGURACIÓ PER PROFESSIONAL (ADMIN / GESTOR)
```

Resolució en temps d'execució: `professional.config ?? ubicacio.config ?? global.config`

---

## 7. Integracions externes

| Sistema | Tipus | Funció | Estat |
|---|---|---|---|
| **GT** (RRHH hospitalari) | REST / intern | Validació de fills per a personal de l'àmbit (DT, IAS) + verificació d'actiu | Fase 2 — DEC-06 |
| **Passarel·la de pagament** | REST | Pagament i devolucions | Redsys o Stripe — DEC-01 |

> **Integracions eliminades**: AD/LDAP ICS, AD/LDAP IAS i WS SOAP d'usuaris externs no s'implementen. L'autenticació és exclusivament local per a tots els col·lectius.

---

## 8. Restriccions i condicionants

| Restricció | Descripció |
|---|---|
| **Entorn hospitalari** | L'aplicació ha de funcionar dins la DMZ corporativa de l'ICS/IAS |
| **RGPD** | Documents acreditatius s'eliminen un cop validats. Dades personals s'anonimitzen transcorregut el període de retenció (a definir amb DPO) |
| **Lliurament presencial** | Cap modalitat d'enviament postal ni entrega a domicili |
| **Finestra de compra** | La compra es restringeix al període configurat (exemple: dies 1-20 del mes) |
| **Elegibilitat per edat** | Fills elegibles fins als 3 anys; ampliació possible per excepcions documentades |
| **Col·lectius configurables** | Els col·lectius que poden accedir al servei es defineixen als paràmetres, sense canvis de codi |

---

## 9. Decisions pendents clau

> Aquestes decisions han de ser resoltes abans de poder dissenyar o implementar les funcionalitats afectades.

| ID | Decisió | Impacte | Estat |
|---|---|---|---|
| **DEC-01** | Passarel·la de pagament: Redsys o Stripe | Disseny del flux de pagament i devolucions | ⬜ Pendent |
| **DEC-02** | Límits de compra: per professional o per fill actiu | Model de dades de compra | ⬜ Pendent |
| **DEC-03** | Cost del servei per al professional (gratuït, cofinançat, per col·lectiu) | Flux de pagament sencer | ⬜ Pendent |
| **DEC-04** | Passarel·la de pagament: suport de devolucions API (Redsys) | Gestió de devolucions parcials | ⬜ Pendent |
| **DEC-05** | Qui tramita les devolucions parcials: gestor IAS o gestor Trueta | Disseny del workflow d'incidències | ⬜ Pendent |
| **DEC-06** | Viabilitat i abast de la integració GT (RRHH): validació de fills + verificació d'actiu | Workflow de validació i baixa automàtica | ⬜ Pendent |
| **DEC-07** | Gestió de comandes pendents quan arriba un mes nou sense lliurar | Cicle de vida de comandes | ⬜ Pendent |
| **DEC-09** | Validació QR per a recepció/entrega de comandes | Disseny interfície de gestors | ⬜ Pendent |
| ~~**DEC-10**~~ | ~~Endpoint del WS de validació d'usuaris externs~~ | ~~Eliminat~~ | ✅ Descartat — auth local per a tots |

---

## 10. Riscos identificats

| Risc | Probabilitat | Impacte | Mitigació |
|---|---|---|---|
| ~~Servei AD/LDAP hospitalari no disponible~~ | — | — | Risc eliminat — auth exclusivament local |
| Devolucions via API Redsys no suportades | Mitjana | Alt | Avaluar Stripe com a alternativa; definir procés manual com a fallback |
| Integració GT no disponible o incompleta | Mitjana | Mig | Fallback a validació documental per a usuaris de l'àmbit; baixa manual per gestor |
| Volum de comandes superior al previst | Baixa | Mig | Disseny escalable; cues per a operacions asíncrones |
| Canvis de normativa RGPD sobre dades sanitàries | Baixa | Alt | Disseny modular de la capa de dades personals; DPO implicat des de l'inici |

---

## 11. Roadmap proposat

### Fase 1 — Fonaments (3–4 mesos)

> Aplicació funcional, segura i testable amb les capacitats equivalents al sistema actual.

| Àmbit | Contingut |
|---|---|
| **Arquitectura** | Nova app Laravel 11 + Vue 3, BD independent, API REST, configuració per entorn |
| **Autenticació** | Local (bcrypt) → token Sanctum (tots els col·lectius) |
| **Rols** | ADMIN, GESTOR_COMPRA, GESTOR_DISTRIBUCIO, PROFESSIONAL, SUPERVISOR |
| **Col·lectius** | DT, IAS, DT Externs, IAS Externs — configurables per paràmetre |
| **Perfils de compra** | Progenitor + Compra especial (IAS) |
| **Gestió de fills** | Alta, validació via GT (interns) o document acreditatiu (externs), límit per edat, baixa automàtica |
| **Productes** | Catàleg, preus, col·lectius, clonar, historial |
| **Compra i pagament** | Finestra mensual, passarel·la (decisió DEC-01), webhook segur, idempotència |
| **Workflow comandes** | Tots els estats: del client fins a l'entrega presencial |
| **Punts de distribució** | Alta, gestió, assignació per professional |
| **Notificacions** | Email per tots els events clau |
| **Auditoria** | `audit_log` immutable amb cobertura completa |
| **Seguretat** | Totes les mesures (M-SEC-01..05) des del dia 1 |
| **Tests** | Unitaris, integració i feature des del primer dia |

---

### Fase 2 — Millores funcionals (2–3 mesos)

| Àmbit | Contingut |
|---|---|
| **Integració GT** | Validació de fills via GT (RRHH) per a personal de l'àmbit + baixa automàtica per fi de contracte (requereix DEC-06) |
| **Devolucions** | Flux complet de devolucions parcials per línia de comanda |
| **Incidències massives** | Vista d'incidències, comunicació massiva, registre de comunicacions |
| **Integració RRHH** | Verificació d'actius i baixa automàtica (requereix DEC-06) |
| **QR entrega** | Validació per codi QR a la recepció/entrega (requereix DEC-09) |
| **Informes avançats** | Fulls de recollida, estoc gastat, enviament automàtic periòdic |
| **Accions massives** | Sobre usuaris, comandes, productes i punts de distribució |
| **Exportació** | Excel, PDF i email per a totes les entitats principals |
| **Observabilitat** | Logging estructurat JSON, alertes per errors crítics, health check |

---

### Fase 3 — Experiència mòbil (1 mes, condicionat a DEC-01 i validació)

| Àmbit | Contingut |
|---|---|
| **PWA** | Aplicació instal·lable, responsive, sense app store |
| **Notificacions push** | Recordatori obertura finestra mensual (iOS 16.4+) |
| **Offline parcial** | Consulta d'historial de comandes sense connexió |
| **Ionic** | Wrapper opcional si es necessita experiència nativa millorada |

---

### Resum executiu

| Fase | Contingut | Estimació |
|---|---|---|
| **Fase 1** | Base, seguretat, funcionalitat equivalent a l'actual | 3–4 mesos |
| **Fase 2** | GT/RRHH, devolucions, incidències, informes | 2–3 mesos |
| **Fase 3** | PWA / mòbil (si es valida) | 1 mes |

---

---

## 12. Estimació d'hores de desenvolupament

> Estimació aproximada per a un equip d'**1 desenvolupador full-stack** amb experiència en Laravel i Vue 3. Les xifres inclouen disseny, implementació, tests i revisió, però exclouen desplegament en producció i formació d'usuaris.
>
> S'ofereixen dues columnes: estimació **sense assistència IA** i estimació **amb Claude Code** com a assistent de desenvolupament principal. Veure la [nota metodològica](#nota-sobre-lús-de-claude-code) al final de la secció.

### Fase 1 — Fonaments

| Mòdul | Tasques principals | Sense IA | Amb Claude Code |
|---|---|---:|---:|
| **Infraestructura base** | Scaffold Laravel 11 + Vue 3, CI/CD local, `.env`, BD independent, migrations inicials | 16 | 7 |
| **Autenticació** | Local (bcrypt) + Sanctum tokens (tots els col·lectius) | 12 | 5 |
| **Rols i permisos** | 5 rols, guards, middleware, polítiques | 10 | 4 |
| **Gestió de professionals** | Alta, aprovació/rebuig, baixa lògica, recuperació compte | 24 | 10 |
| **Gestió de fills** | Alta, CIP, validació (GT / document), límit edat, Job baixa automàtica | 32 | 14 |
| **Catàleg de productes** | CRUD, preus, col·lectius, clonar, historial de canvis | 16 | 6 |
| **Compra i pagament** | Finestra mensual, passarel·la (webhook + idempotència + transacció DB) | 44 | 24 |
| **Workflow de comandes** | Tots els estats, transicions, canvis d'estat per gestor | 32 | 14 |
| **Punts de distribució** | CRUD, assignació per professional i centre | 10 | 4 |
| **Notificacions email** | Plantilles, queue jobs, events clau | 14 | 6 |
| **Auditoria** | Taula `audit_log` immutable, listeners, cobertura completa | 16 | 6 |
| **Seguretat** | HMAC webhook, URLs signades, secrets a `.env`, sanitització inputs | 14 | 8 |
| **Tests Fase 1** | Unitaris, integració i feature per a tots els mòduls anteriors | 32 | 13 |
| **Frontend Vue 3 — Fase 1** | SPA: login, dashboard professional, compra, gestió fills, panell gestor bàsic | 60 | 26 |
| **Subtotal Fase 1** | | **332** | **147** |

### Fase 2 — Millores funcionals

| Mòdul | Tasques principals | Sense IA | Amb Claude Code |
|---|---|---:|---:|
| **Integració GT (RRHH)** | Client REST/SOAP, validació fills àmbit, verificació actiu, baixa automàtica | 28 | 16 |
| **Devolucions** | Devolucions parcials per línia, API passarel·la o manual, notificació | 24 | 12 |
| **Incidències massives** | Vista filtrable, email massiu/individual, registre comunicació | 18 | 8 |
| **QR entrega** | Generació codi QR, escàner, validació recepció/entrega | 12 | 5 |
| **Informes avançats** | Fulls recollida, estoc per període, exportació Excel/PDF, enviament automàtic | 24 | 10 |
| **Accions massives** | Usuaris, comandes, productes, punts — selecció i acció en bloc | 14 | 6 |
| **Observabilitat** | Logging JSON estructurat, alertes errors crítics, health check endpoint | 8 | 3 |
| **Tests Fase 2** | Tests per als nous mòduls | 18 | 7 |
| **Frontend Vue 3 — Fase 2** | Vistes incidències, QR, informes, accions massives | 36 | 15 |
| **Subtotal Fase 2** | | **182** | **82** |

### Fase 3 — Experiència mòbil *(opcional)*

| Mòdul | Tasques principals | Sense IA | Amb Claude Code |
|---|---|---:|---:|
| **PWA** | Manifest, service worker, responsive complet | 14 | 6 |
| **Notificacions push** | Web Push API, subscripcions, recordatoris finestra | 12 | 5 |
| **Offline parcial** | Cache historial comandes, sincronització | 8 | 3 |
| **Ionic (opcional)** | Wrapper natiu si es valida la necessitat | 18 | 8 |
| **Subtotal Fase 3** | | **52** | **22** |

### Resum global

| Fase | Sense IA | Amb Claude Code | Estalvi |
|---|---:|---:|---:|
| **Fase 1** — Fonaments | 332 h (~8 set.) | 147 h (~3,7 set.) | ~56% |
| **Fase 2** — Millores funcionals | 182 h (~4,5 set.) | 82 h (~2 set.) | ~55% |
| **Fase 3** — Mòbil (opcional) | 52 h (~1,5 set.) | 22 h (~0,5 set.) | ~58% |
| **Total Fase 1 + 2** | **514 h** (~13 set.) | **229 h** (~5,7 set.) | **~55%** |
| **Total complet (3 fases)** | **566 h** (~14 set.) | **251 h** (~6,3 set.) | **~56%** |

### Nota sobre l'ús de Claude Code

L'estimació "Amb Claude Code" assumeix que Claude Code s'utilitza com a **assistent de desenvolupament principal** al llarg de tot el projecte (no puntualment). El factor d'estalvi és desigual per tipus de tasca:

| Tipus de tasca | Estalvi típic | Motiu |
|---|---|---|
| Scaffold, migrations, models, CRUD | 60–70% | Patrons molt repetitius, ben coberts per LLM |
| Tests (unitaris i feature) | 55–65% | Generació automàtica a partir del codi existent |
| Frontend Vue (components, formularis) | 50–60% | Estructures predecibles i documentades |
| Lògica de negoci complexa | 35–45% | Cal supervisió humana; risc d'errors subtils |
| Integracions externes (GT) | 25–35% | Depèn de documentació disponible i proves reals |
| Seguretat i criptografia | 20–30% | Revisió humana obligatòria en tot cas |

> **Advertència**: les hores "Amb Claude Code" representen **temps de l'enginyera/enginyer** (supervisió, revisió, integració, decisions de disseny), no el temps de generació del model. Claude Code pot generar codi incorrecte o insegur; la revisió crítica és imprescindible, especialment en els mòduls de pagament, seguretat i integracions hospitalàries.

---

## Documents relacionats

| Document | Descripció |
|---|---|
| [parametritzacio.md](parametritzacio.md) | Disseny del model de parametrització |
| [gestio_comandes.md](gestio_comandes.md) | Workflow detallat de comandes |
| [Bolquers 2026.md](Bolquers%202026.md) | Notes de requeriments originals (iteració 2026) |
