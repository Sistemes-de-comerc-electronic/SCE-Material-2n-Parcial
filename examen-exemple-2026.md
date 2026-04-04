## Exercici 1 (4 punts)

Treballes en una empresa que ofereix una plataforma SaaS de **gestió de botigues en linia**. Ha arribat un company nou (que no ha cursat Sistemes de Comer Electronic) i l'heu posat a fer codi ja des del primer dia.

Realitza una revisio exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala practica o l'error conceptual.
2.	**Explica per que es un problema:** Justifica la teva observacio fent referencia als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestio d'errors, etc.) i explica les seves consequencies negatives.
3.	**Proposa una solucio:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

[Codi a revisar](codi-exemple-2026-sense-solucions.md)

## Exercici 2 - 2 punts

L'empresa de logística "FastDelivery" té un departament d'Enviaments que ha creat una API pública per consultar les tarifes i els temps de lliurament. El departament de Vendes d'una botiga online de la mateixa empresa utilitza aquesta API en el seu propi procés de compra (Bounded Context de Compres).

Darrerament, l'equip d'Enviaments canvia l'estructura de dades de la seva API de forma freqüent i sense previ avís, la qual cosa provoca que el sistema de Vendes falli contínuament perquè el seu codi s'hi acobla directament. L'equip d'Enviaments afirma que no tenen temps per atendre les necessitats de Vendes.

- Quina relació o patró d'integració (Context Map) creus que està patint l'equip de Vendes actualment? (1 punt)

- Quin patró d'integració hauria d'implementar l'equip de Vendes per protegir el seu Bounded Context d'aquests canvis inesperats? Justifica la teva resposta. (1 punt)

## Exercici 3 - 2 punts

Una plataforma de venda d'entrades per a concerts massius pateix caigudes recurrents. Quan s'anuncia un concert molt esperat, milions d'usuaris entren alhora per veure la disponibilitat d'entrades i, al mateix temps, milers d'usuaris intenten comprar-les. Això provoca bloquejos (deadlocks) a la base de dades relacional, ja que les operacions de lectura (veure disponibilitat) i escriptura (comprar entrada i descomptar estoc) ataquen les mateixes taules simultàniament.

- Com podríeu solucionar aquest problema? Quin patró faríeu servir? Perquè? Argumenta com l'implementaries. (2 punts)

## Exercici 4 - 2 punts

L'estudi de videojocs independent "PixelDreams" està a punt de publicar el seu nou joc propietari. Per a les físiques del joc, els desenvolupadors han integrat una llibreria de codi obert que està sota la llicència GNU General Public License (GNU GPL). 

A més, per evitar la pirateria, l'estudi vol afegir una marca oculta al codi de cada joc venut. Aquesta marca ha de contenir la informació específica del comprador perquè, si el joc es filtra a internet, puguin identificar exactament quin usuari l'ha piratejat. Un desenvolupador suggereix fer servir la tècnica d'esteganografia anomenada **"Watermark"**.

La sessio de l'usuari es gestiona amb un JWT que es guarda dins del codi font del joc. Un altre desenvolupador suggereix posar totes les dades del perfil de l'usuari (nom, email, adreça) dins del payload del JWT per estalviar consultes a la base de dades.

- Poden vendre el seu joc com a programari propietari (tancat) si inclou codi sota la llicència GNU GPL? Per què? (0.5 punts)
- És correcta la tècnica que especifiquen per identificar l'usuari que ha piratejat el joc? Perquè? Argumenta com ho implementaries. (1 punt)
- Que opines de la proposta de posar totes les dades del perfil de l'usuari dins del payload del JWT? Perquè? Argumenta la teva resposta. (0.5 punts)


# Solucions

## Exercici 2

- **Quina relació o patró d'integració (Context Map) creus que està patint l'equip de Vendes actualment?**

S'espera que l'alumne identifiqui que actualment l'equip de Vendes està en una posició de **Conformist**, ja que s'adhereixen estrictament al model de l'equip upstream (Enviaments) sense cap traducció i pateixen els canvis directament, o bé que és un Open Host Service mal gestionat per part d'Enviaments.

- **Quin patró d'integració hauria d'implementar l'equip de Vendes per protegir el seu Bounded Context d'aquests canvis inesperats? Justifica la teva resposta.**
  
La solució que ha d'implementar l'equip de Vendes (downstream) és una **Anti-Corruption Layer (ACL)**. Això consisteix a implementar una capa de traducció per aïllar-se dels possibles canvis de l'upstream BC. D'aquesta manera, modelen una entitat amb els termes del seu propi Bounded Context i, si l'API d'Enviaments canvia, només hauran d'actualitzar la capa de traducció, deixant intacte el seu model de domini.

A la pràctica heu fet algo molt similar amb els DTO, si el backend afegeix un paràmetre perquè la API el necessita, no afectarà al frontend perquè el DTO no el té, i si el backend canvia el nom d'un paràmetre, només haureu de canviar la capa de traducció (DTO) i no tot el codi que utilitza aquest paràmetre.

## Exercici 3 

- **Quin patró faries servir? Perquè? Argumenta com l'implementaries.**

S'espera que l'alumne faci una proposta, si té sentit i no hi han problemes es donarà per bona.

Una possible solució: l'Event Store actua com a base de dades d'escriptura (Write Database), on s'enregistren les compres ràpidament sense bloquejos complexos. D'altra banda, es manté una base de dades de lectura (Read Database) optimitzada només per consultar la disponibilitat, la qual s'actualitza a partir dels esdeveniments de l'escriptura. Això separa la càrrega i elimina el coll d'ampolla central. S'espera que l'alumne mencioni afegir **caché** a les consultes que no requereixin dades en temps real (o netejar el caché quan es produeixi una compra) per millorar encara més el rendiment.

### Exercici 4

- **Poden vendre el seu joc com a programari propietari (tancat) si inclou codi sota la llicència GNU GPL? Per què?**

No, no poden vendre'l com a programari propietari tancat. La llicència GNU GPL exigeix que qualsevol treball derivat tingui la mateixa llicència de programari lliure. Si haguessin utilitzat LGPL o BSD, sí que estaria permès integrar-ho com a part d'un programari propietari sense estar obligats a obrir la resta del codi.

- **És correcta la tècnica que especifiquen per identificar l'usuari que ha piratejat el joc? Perquè? Argumenta com ho implementaries.**

No, el desenvolupador s'equivoca. La tècnica Watermark s'utilitza per contenir la informació del propietari del contingut (en aquest cas, PixelDreams). Per poder identificar el comprador específic i traçar la redistribució il·legal d'una còpia concreta, la tècnica correcta és el **Fingerprint**, que posa una marca diferent a cada còpia venuda amb la informació de l'usuari final.

Si l'alumne raona de manera correcta la implementació es donarà per vàlid, no cal que entri en detalls. Un exemple: *Afegint un camp a la base de dades, al carregar el joc, es genera un identificador únic per a cada còpia que es ven, i aquest identificador es guarda xifrat en un fitxer del joc. Si el joc es filtra, poden recuperar aquest identificador i consultar la base de dades per veure a quin usuari correspon.*

- **Que opines de la proposta de posar totes les dades del perfil de l'usuari dins del payload del JWT? Perquè? Argumenta la teva resposta.**

No és una bona pràctica. El JWT està pensat per portar informació mínima i essencial per a l'autenticació i autorització, com ara l'identificador de l'usuari i els seus rols. Posar totes les dades del perfil de l'usuari dins del payload del JWT pot fer que si el token es filtra, es comprometi la privacitat de l'usuari. 