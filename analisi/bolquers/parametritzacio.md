# Parametrització de l'Aplicació

**Versió**: 0.1  
**Data**: 2026-08-13  
**Estat**: Esborrany

---

## Índex

1. [Principis de disseny](#1-principis-de-disseny)
2. [Nivells de configuració](#2-nivells-de-configuració)
3. [Paràmetres globals](#3-paràmetres-globals)
4. [Paràmetres per ubicació](#4-paràmetres-per-ubicació)
5. [Paràmetres per professional](#5-paràmetres-per-professional)
6. [Gestió de la configuració](#6-gestió-de-la-configuració)
7. [Model de dades](#7-model-de-dades)
8. [Historial de canvis](#8-historial-de-canvis)

---

## 1. Principis de disseny

- **Cap valor hardcoded al codi**: qualsevol paràmetre que pugui variar amb el temps ha de ser configurable.
- **Jerarquia amb herència**: els paràmetres per ubicació sobreescriuen els globals; els per professional sobreescriuen els per ubicació.
- **Traçabilitat**: tot canvi de configuració queda registrat al log d'auditoria (qui, quan, valor anterior, valor nou).
- **Validació**: els valors es validen en guardar (tipus, rangs, coherència entre paràmetres relacionats).
- **Efecte immediat vs diferit**: alguns canvis s'apliquen immediatament (ex: finestra mensual), d'altres s'apliquen a partir del mes següent per no afectar comandes en curs.

---

## 2. Nivells de configuració

```
┌─────────────────────────────────────────────────────┐
│              CONFIGURACIÓ GLOBAL                    │
│  Valors per defecte per a tota l'aplicació          │
│  Modificable per: ADMIN                             │
├─────────────────────────────────────────────────────┤
│           CONFIGURACIÓ PER UBICACIÓ                 │
│  Sobreescriu globals per a una ubicació concreta    │
│  Modificable per: ADMIN                             │
├─────────────────────────────────────────────────────┤
│          CONFIGURACIÓ PER PROFESSIONAL              │
│  Excepcions individuals (casos especials)           │
│  Modificable per: ADMIN, GESTOR                     │
└─────────────────────────────────────────────────────┘
```

**Resolució de valors en temps d'execució**:
```
valor_efectiu = professional.config ?? ubicacio.config ?? global.config
```

---

## 3. Paràmetres globals

### 3.1 Servei i elegibilitat

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `edat_maxima_fill_anys` | integer | `3` | Edat màxima del fill per tenir accés al servei |
| `max_fills_actius_professional` | integer | `5` | Nombre màxim de fills actius per professional |
| `dies_validesa_sol·licitud` | integer | `30` | Dies que té el professional per completar la sol·licitud d'accés abans que caduqui |
| `permet_acces_30km` | boolean | `true` | Activa el règim especial per a professionals >30km |

### 3.2 Finestra de comandes

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `finestra_min_dia` | integer (1–28) | `1` | Dia del mes d'obertura de la finestra de comandes |
| `finestra_max_dia` | integer (1–28) | `20` | Dia del mes de tancament de la finestra |
| `mesos_tancats` | array | `[]` | Mesos en què el servei no opera (ex: `[8]` per agost) |
| `hores_expiracio_comanda_pendent` | integer | `24` | Hores fins que una comanda `pendent_pagament` expira |
| `finestra_cancel·lacio_hores` | integer | `48` | Hores després de la confirmació durant les quals el professional pot cancel·lar |

### 3.3 Límits de productes (per defecte, sobreescrivibles per ubicació)

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `max_bolquers_mes` | integer | `60` | Bolquers màxims per professional per mes |
| `max_tovalloletes_mes` | integer | `120` | Tovalloletes màximes per professional per mes |
| `max_empapadors_mes` | integer | `30` | Empapadors màxims per professional per mes |
| `tovalloletes_tot_personal` | integer | `null` | Límit global de tovalloletes per a tot el personal de la ubicació (null = sense límit global) |
| `empapadors_tot_personal` | integer | `null` | Límit global d'empapadors per a tot el personal |

### 3.4 Pagament

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `passarela_activa` | enum `[redsys\|stripe]` | `redsys` | Passarel·la de pagament activa |
| `permet_comandes_sense_pagament` | boolean | `false` | Activa comandes gratuïtes (sense pas de pagament) |
| `import_minim_comanda` | decimal | `0.01` | Import mínim per processar un pagament |

### 3.5 Validació de fills

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `validacio_pica_activa` | boolean | `false` | Activa la validació automàtica via PICA |
| `validacio_document_activa` | boolean | `true` | Permet adjuntar document PDF per validar |
| `dies_retencio_document_pdf` | integer | `0` | Dies que es conserva el PDF post-validació (0 = eliminació immediata) |
| `max_intents_validacio` | integer | `3` | Vegades que el professional pot intentar validar un fill |

### 3.6 Notificacions

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `email_gestio_bolquers` | email | — | Email receptor d'alertes internes del servei |
| `notificacio_comanda_confirmada` | boolean | `true` | Envia email en confirmar comanda |
| `notificacio_pagament_rebutjat` | boolean | `true` | Envia email si el pagament falla |
| `notificacio_comanda_llesta` | boolean | `false` | Envia email quan la comanda està llesta per recollir |
| `notificacio_recordatori_finestra` | boolean | `false` | Envia recordatori quan s'obre la finestra mensual |
| `dies_antelacio_recordatori` | integer | `1` | Dies d'antelació per al recordatori d'obertura |

### 3.7 Incidències

| Paràmetre | Tipus | Defecte | Descripció |
|---|---|---|---|
| `dies_resolucio_incidencia` | integer | `5` | Dies hàbils màxims per resoldre una incidència (per a SLA intern) |
| `escalat_incidencia_dies` | integer | `7` | Dies sense resposta fins que es notifica a l'ADMIN |

---

## 4. Paràmetres per ubicació

Cada ubicació pot sobreescriure qualsevol paràmetre global dels grups **Finestra de comandes**, **Límits de productes** i **Notificacions**. A més, té paràmetres propis:

| Paràmetre | Tipus | Descripció |
|---|---|---|
| `activa` | boolean | Si la ubicació té el servei actiu |
| `nom_ubicacio` | string | Nom visible de la ubicació |
| `email_gestio_local` | email | Email del gestor local (sobreescriu el global) |
| `missatge_recollida` | text | Text personalitzat als fulls de recollida |
| `dia_recollida_mes` | integer | Dia habitual de recollida en aquesta ubicació |
| `max_comandes_30km` | integer | Límit de comandes per al règim >30km d'aquesta ubicació |
| `bolquers_30km` | integer | Límit de bolquers per al règim >30km |
| `gestio_documents_local` | boolean | Sobreescriu si cal gestionar documents en aquesta ubicació |

**Exemple de resolució**:
```
Global:   finestra_min_dia = 1,  finestra_max_dia = 20
Ubicació Trueta: finestra_min_dia = 5, finestra_max_dia = 25
→ El personal de Trueta té finestra del 5 al 25
→ La resta d'ubicacions: del 1 al 20
```

---

## 5. Paràmetres per professional

Excepcions individuals aplicades per ADMIN o GESTOR per a casos especials. Sobreescriuen ubicació i globals.

| Paràmetre | Tipus | Descripció |
|---|---|---|
| `bolquers_il·limitats` | boolean | El professional no té límit mensual de bolquers |
| `max_bolquers_mes_override` | integer \| null | Límit mensual personalitzat de bolquers (null = usa el de la ubicació) |
| `max_tovalloletes_mes_override` | integer \| null | Límit mensual personalitzat de tovalloletes |
| `max_empapadors_mes_override` | integer \| null | Límit mensual personalitzat d'empapadors |
| `max_fills_actius_override` | integer \| null | Nombre màxim de fills personalitzat |
| `edat_maxima_fill_override` | integer \| null | Edat màxima del fill personalitzada (en anys) |
| `exclou_finestra_mensual` | boolean | El professional pot fer comandes fora de la finestra |
| `actiu_fins` | date \| null | Data de caducitat del servei per a aquest professional (null = indefinit) |
| `observacions_admin` | text | Notes internes visibles només per ADMIN i GESTOR |

**Totes les excepcions individuals requereixen**:
- Justificació documentada en crear-les
- Registre a l'auditoria amb qui l'ha aplicat i quan
- Revisió periòdica (opcional: data d'expiració de l'excepció)

---

## 6. Gestió de la configuració

### 6.1 Panell d'administració

La configuració és accessible des del panell d'ADMIN organitzada en pestanyes:

```
Configuració
├── General          (paràmetres globals — servei, fills, pagament, validació)
├── Finestra         (finestra mensual, mesos tancats, expiració)
├── Límits           (bolquers, tovalloletes, empapadors per defecte)
├── Notificacions    (emails, triggers, recordatoris)
├── Incidències      (SLA, escalat)
├── Per ubicació     (sobreescriptures per cada centre)
└── Historial        (log de tots els canvis de configuració)
```

### 6.2 Validacions en guardar

| Paràmetre | Validació |
|---|---|
| `finestra_min_dia` / `finestra_max_dia` | `min < max`, ambdós entre 1 i 28 |
| `edat_maxima_fill_anys` | Entre 1 i 6 |
| `max_fills_actius_professional` | Entre 1 i 10 |
| `hores_expiracio_comanda_pendent` | Entre 1 i 168 (1 setmana) |
| `finestra_cancel·lacio_hores` | ≤ `hores_expiracio_comanda_pendent` |
| `dies_retencio_document_pdf` | Entre 0 i 365 |
| `email_gestio_bolquers` | Format email vàlid |
| `mesos_tancats` | Array d'enters entre 1 i 12, sense duplicats |

### 6.3 Efecte dels canvis

| Canvi | Efecte |
|---|---|
| Finestra mensual | Immediat — afecta si es pot crear comanda ara |
| Límits de productes | A partir del mes següent — no afecta comandes en curs |
| Edat màxima fill | Immediat per a noves altes; les existents no es revoquen |
| Passarel·la de pagament | Immediat — afecta la pròxima comanda |
| Notificacions | Immediat |
| Mesos tancats | Al mes corresponent |

---

## 7. Model de dades

### Taula `configuracio_global`

Taula amb una sola fila (singleton) o estructura clau-valor:

```
configuracio_global
├── id
├── clau              VARCHAR — nom del paràmetre
├── valor             TEXT — valor serialitzat (JSON per a arrays)
├── tipus             enum [integer|boolean|decimal|string|email|array|enum]
├── descripcio        TEXT — descripció llegible
├── grup              VARCHAR — agrupació per a la UI
├── modificable_per   enum [admin|sistema]
└── timestamps
```

### Taula `configuracio_ubicacio`

```
configuracio_ubicacio
├── id
├── idUbicacio        FK → ubicacions
├── clau              VARCHAR
├── valor             TEXT
└── timestamps
```

### Taula `configuracio_professional`

```
configuracio_professional
├── id
├── idProfessionalBolquer   FK → professionals_bolquers
├── clau                    VARCHAR
├── valor                   TEXT
├── justificacio            TEXT — obligatori
├── idAdminCreador          FK → professionals
├── data_expiracio          DATE nullable
└── timestamps
```

### Servei d'accés (PHP)

```php
// Resolució de valor amb jerarquia
ConfigService::get('max_bolquers_mes', $idProfessional, $idUbicacio);
// → professional.config ?? ubicacio.config ?? global.config

// Exemples d'ús al codi
$maxBolquers = ConfigService::get('max_bolquers_mes', $professional->id, $professional->idUbicacio);
$finestraOberta = ConfigService::finestraComandaActiva($idUbicacio);
$edat = ConfigService::get('edat_maxima_fill_anys'); // global, sense context
```

---

## 8. Historial de canvis

Tot canvi de configuració genera un registre al log d'auditoria:

```
{
  "tipus_event": "canvi_configuracio",
  "clau": "max_bolquers_mes",
  "ambit": "ubicacio",          // global | ubicacio | professional
  "id_ambit": 3,                // idUbicacio o idProfessional si escau
  "valor_anterior": "60",
  "valor_nou": "80",
  "justificacio": "Increment autoritzat per direcció ref. acord 2025-03",
  "id_professional": 42,
  "timestamp": "2025-03-01T09:15:00Z"
}
```

La pestanya **Historial** del panell mostra tots els canvis amb filtres per data, paràmetre i qui l'ha fet. Permet veure "com estava configurada l'app el dia X" per a auditories.
