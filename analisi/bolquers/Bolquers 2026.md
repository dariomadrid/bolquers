# **Bolquers 2026**

# **Requeriments**

**Arquitectura**

- Backend Laravel  
- Frontend vue  
- PWA? \=\> Valorar Frontend Ionic (si es necessita PWA).  
- DMZ corporativa (accés a aplicacions internes com GT,  WS DA, etc).  
- Única DB

**Auth**

- local  
- Integrar amb WS validacio per obtenir dades d’usuari (externs)

**Usuari**

- Info usuari: guardant el mínim possible de dades: nom, email, telf  
- Workflow Alta usuari  
  - Usuari es dona d’alta  
    - Sistema pregunta colectiu: DT, IAS, DT externs (banc sang, ICO…), IAS Externs (...).  
      - Caldrà definir gestors per centre. Implementar vista gestors amb filtres: centre, data solicitud, professional…  
    - Pregunta per perfil compra (progenitor o compra\_especial)  
      - Progenitor  
      - Compra especial (empapadors, etc): De moment només per colectiu IAS i per empapadors però cal tenir en compte que es poden entrar altres productes en aquesta excepció.  
        - Si “progenitor”:  
          - Alta fill  
            - Dades: CIP (identificatiu, data naix)  
            - Si extern \=\> cal demanar document acreditatiu (procés upload document per a que validi gestor i un cop validat caldrà esborrar fitxer)  
            - Queda pendent de validació usuari/fill pel gestor  
        - Si “compra especial”  
          - només es pot registrar si usuari IAS (definir regles d’acceptació per si es modifica en un futur).  
          - Queda pendent de validació usuari per gestor centre  
    -   
  - Gestor accepta/rebutja l’alta  
  - Avisar sempre tant si OK com no OK. Si no OK, petició queda cancelada amb motiu i indiques que torni a fer'ho.  
  - Definir als paràmetres aplicació quins colectius es poden registrar.  
  - Un professional pot tenir accés simultani com a GESTOR i com a usuari del servei (tenir fills registrats i fer comandes)  
  - Si professional es dona de baixa, cal donar de baixa lògica els fills associats. Revisar si el fill pertany a un altre professional abans de donar-lo de baixa.  
  - Integració amb sistemes de RRHH de l'hospital per verificar que el professional segueix en actiu i donar-lo de baixa automàticament en cas de finalització contracte.  
    - Facilitar la recuperació de l’usuari si ja estava donat d’alta.

**Fills**

- CIP és l’identificador del nen  
- Un fill pot estar definit a varis pares (permetre que diferents pares puguin gestionar la compra)  
- Limitació compra per fill actiu  
- Validesa per edat de compra. Permetre ampliació via solicitud i justificació pel pare.  
- No hi ha limit de fills per professionals  
- El CIP generat (\`substr(cognom1,0,2)+substr(cognom2,0,2)+sexe+any+mes+dia\`) és un identificador intern i té relació amb algun sistema sanitari extern (CIP Catsalut)

**Productes**

- Vista productes amb filtres on poder mantenir dades de producte (preu, descripció, imatges, pertanyent a quins colectius…)  
- En principi el preu serà igual per tots els colectius però tenir en compte que pot variar  
- Facilitar per crear productes a partir dun altre  
- Guardar descripció producte a comanda  
- Històric canvis de producte

**Parametritzacions generals app**

- límits compra per nen  
- períodes compra  
- tots els colectius igual

**Compra**

- [Pasarel.la](http://Pasarel.la) de pagament  
  - Redsys vs Stripe  
  - Revisar si redsys te api que pugui gestionar devolucions  
  - En cas negatiu, valorar strype amb Tiscar IAS  
- Perídodes de comanda oberta del 1 al 20 (parametritzable per app)  
  - Pot haver-hi mesos sense finestra oberta (vacances, tancaments...)  
- Tancament de les compres del 20 al final de mes (parametritzable via app)  
- Cal emetre rebut per email al comprador

**Gestió**

- Gestió de Punts de distribució (entrega)  
  - Alta, localització, horari entrega, etc  
  - El professional por canviar el punt de distribució dels disponibles


**Comandes**

Els usuaris clients fan compra. Aquesta arriba a gestor punt distribució que tramita la comanda cap al gestor de compra (IAS). Aquest fa la compra agrupada i quan li arriba envia els productes al punts de distribució. La finestra de compra per l’usuari client està definida a paràmetres app (normalment de 1 al 20 del mes) i fora d’aquest tram es fa la gestió de compra i distribució productes als punts de distribució i entrega a l’usuari final. 

El lliurament és sempre presencial.

Workflow comanda:

- Usuari client compra productes   
- Només es pot comprar en una finestra de temps definida a l’app (Exemple de 1 al 20 del mes  
- Durant aquest període usuari pot fer modificacions/cancelacions de comanda  
- Gestor del punt de distribució agrupa les comandes i les tramita al gestor compra (IAS)  
- Gestor compra (IAS) fa la compra de tots els productes de les comandes del període al proveidor (no agrupa en funció de punts de distribució o orígen, fa compra en números absoluts)  
- Proveidor entrega productes  
- Gestor compra valida l’entrega de productes  
- Gestor compra distribueix productes als punts de distribició  
- Gestor punt de distribució valida recepció de productes  
- Un cop recepcionada comanda es marca com a pendent entrega i s’envia email a usuari client  
- Usuari client recull la comanda i es marca com entregada

Excepcions workflow comanda:

- Gestor compra no pot comprar producte per falta stock proveidor o hi ha canvi de preu  
  - Cal gestionar devolucions parcials de la comanda (Tenir en comopte que es fa a nivell de linia de comanda\!)  
  - Exemple:   
    - Comanda id 190872312  
      - Producte dodot talla 4 x100  
      - Producte dodot talla 6 x50  
      - El talla 6 no es pot comprar, la posaras amb estat "falta estock"  
      - Caldra buscar totes les comandes afectades i devolucionar diners  
      - Decidir qui és l’actor que ho fa: si ho fa el gestor compra de IAS o ho poden fer des de trueta  
- Si hi ha errror de pagament per l’usuari client no s’ha de processar la comanda per la compra.  
- Cal poder gestionar incidències de manera massiva  
  - vista comandes filtrant per error, etc  
  - Habilitar botó de comunicació per email (massiva o individual) per indicar quin procediment es farà per solucionar (devolució total, parcial)  
    - Poder enviar un mail a tothom afectat indicant que es devoluciona els diners i que tornin a començar  
  - Registrar tota comunicació amb l’usuari (per telèfon, mail, etc)

Traçament comanda

- estats: pendent pagament, pagada, pendent compra, comprada, rebuda, rebuda amb incidència…  
- Exportació a excel/email  
- Reprocés comandes  
- Tramitar devolucions  
  - Comunicació amb els usuaris per comanda

Més info

- Gestor compra la comanda amb números absoluts sense tenir en compte orígen/destinació  
  - X unitats de producte Y  
  - Z unitats de producte K  
- Quan arriba compra valida si el que arriba es correspon amb la compra  
- Distribueix productes als punts de distribució  
- Gestors de Punts distribució   
  - recepcionen comanda  
  - Desconec si aquí validen que ha arribat tot correctament  
  - Registrar entrega a usuari, cal pensar com fer-ho de manera fàcil:  
    - A través de QR que llegeixi i validi per facilitar entrega comanda o recepció?  
- Implementar vista comandes   
  - per gestors (compra o punt distribució) amb filtres : estat, període compra, dates, usuari, gestor….

    

  - per usuari amb filtres: estat, període compra, dates…

**Comunes**

- Traçabilitat a tota l’aplicació  
- Velocitat de càrrega de llistats  
- Habitar accions massives sobre:  
  - Usuaris:  desactivació,...  
  - Comandes: canvis estat, emails,...  
  - Productes  
  - Punts distribució  
  - …  
- Exportació a excel/PDF/email:  
  - comandes, productes, usuaris, etc

**Notificacions**

- El professional rep notificació (parametritzable per app)  
  - Comanda feta  
  - Comanda disponible per recollirt  
  - Possibles incidències (devolucions, retràs, falta stock…)  
- Gerstor rep notificació (parametritzable per app)  
  - Informes mensuals

**Informes i estadístiques**

- fulls de recollida, estoc gastat generats sota demanda i enviats auto amb periodicitat parametritzable  
- Perfil gestor/admin/supervisior


**Proposta millores**

\#\#\# M-PAG-01 — Flux de pagament segur i idempotent 🟡

\`\`\`1. Professional inicia comanda→ Comanda creada en estat \[pendent\_pagament\]→ idempotency\_key generat (UUID)2. Professional completa pagament a la passarel·la3. Passarel·la notifica webhook (POST)→ Verificació signatura HMAC (M-SEC-01)→ Idempotència: rebutjar si ja processat→ DB::transaction: comanda confirmada \+ pagament registrat→ Job: email de confirmació (asíncron)4. Navegador rep confirmació (independent del webhook)5. Job programat: neteja comandes en \[pendent\_pagament\] \>24h\`\`\`

**\#\#\# M-AUDIT-01 — Log d'auditoria transversal**

Taula \`audit\_log\` immutable

\*\*Cobertura obligatòria\*\*:

| Event | Inclou |

|---|---|

| Login / logout (ok i ko) | IP, mètode autenticació |

| Alta / baixa / modificació professional | Dades abans/després |

| Alta / baixa / modificació fill | Dades abans/després |

| Validació (accés, fill, ampliació) | Resultat, comentari del gestor |

| Inici / confirmació / error pagament | Import, gateway\_order\_id |

| Accés i eliminació de documents | Nom fitxer, motiu eliminació |

| Canvis de configuració | Dades abans/després |

| Assignació / revocació de rols | Qui assigna, a qui, quin rol |

| Lliurament de comandes | Qui lliura, quines comandes |

\*\*Restriccions\*\*: cap rol (inclòs ADMIN) pot modificar ni eliminar registres d'auditoria.

**\#\#\# M-AUDIT-02 — Gestió del cicle de vida de les dades**

\*\*Baixa voluntària del professional\*\*:

1\. Gestor inicia baixa

2\. Comandes en curs: lliurar o cancel·lar (decisió manual)

3\. Dades personals: anonimitzar transcorregut el període de retenció (pendent DPO)

4\. Registre a l'auditoria: motiu, data, qui ha fet la baixa

\*\*Baixa automàtica per edat del fill\*\* (Job programat diari):

1\. Detecta fills que superen els 3 anys

2\. Marca fill com a inactiu

3\. Si el professional no té més fills actius, avalua baixa del servei

4\. Notificació per email al professional

**\#\#\# M-UX-02 — Flux de comanda en un sol pas**

\- Selecció de productes amb indicador visual del límit mensual restant

\- Resum de comanda abans de pagar

\- Redirecció clara a la passarel·la i retorn a l'aplicació

\- Pàgina d'estat de comanda: pagada, pendent de lliurament, entregada

\- Historial de comandes per al professional

—

**\#\#\# M-UX-03 — Notificacions per email millorades**

\- Una plantilla HTML responsive per cada tipus d'event (no una plantilla genèrica)

\- Informació contextual: productes, import, punt de distribució, dates

\- Enviament asíncron via Job (no bloqueja la resposta HTTP)

\- Registre d'enviament (\`enviat\_ok\`, \`data\_enviament\`) amb possibilitat de re-enviament

—

**\#\# 10\. Operacions i manteniment**

**\#\#\# M-OPS-01 — Tests automatitzats des del primer dia**

| Àrea | Tipus |

|---|---|

| Verificació del callback de pagament | Test d'integració (mock passarel·la) |

| Càlcul de límits mensuals | Test unitari |

| Generació de CIP | Test unitari |

| Flux de validació de fills | Test de feature |

| Autenticació (Local, AD, WS) | Test d'integració |

| Control d'accés per rol | Test de feature |

| Cicle de vida de comandes (estats) | Test de feature |

—

**\#\#\# M-OPS-02 — Observabilitat**

\- Logging estructurat JSON (Laravel Log \+ canal extern)

\- Alertes per a errors crítics: callback de pagament fallat, servei LDAP caigut, job de email fallat

\- Health check endpoint: \`GET /health\` → estat DB, cua, serveis externs

\- Pàgines d'error personalitzades sense stack traces exposats

**Definir Roadmap**

**Calcular hores de desenvolupament**