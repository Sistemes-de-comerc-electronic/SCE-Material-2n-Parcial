# Examen 2n parcial:

## Exercici 1 (4 punts)

Treballes en una empresa on fan sistemes de factures, ha arribat un company nou (que no ha cursat Sistemes de Comerç Electrònic) i l'heu posat a fer codi ja des del primer dia.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:

- **Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
- **Explica per què és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., DDD, SRP, seguretat, gestió d'errors, etc.) i explica les seves conseqüències negatives.
- **Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap pots posar "LGTM" (Looks Good To Me).

```
<?php

namespace App\Service;

use App\Entity\Invoice;
use App\Repository\InvoiceRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;
use Symfony\Contracts\HttpClient\HttpClientInterface;

class PaymentProcessor
{
    private $entityManager;
    private $client; // HTTP Client
    private $logger;
    private $invoiceRepository;

    public function __construct(
        EntityManagerInterface $em,
        HttpClientInterface $httpClient,
        LoggerInterface $logger,
        InvoiceRepository $invoiceRepository
    ) {
        $this->entityManager = $em;
        $this->client = $httpClient;
        $this->logger = $logger;
        $this->invoiceRepository = $invoiceRepository;
    }

    /**
     * Processa el pagament per a una factura i actualitza el seu estat.
     * Aquest mètode és cridat per una acció de controlador després que l'usuari faci clic a "Pagar".
     */
    public function doAction(int $invoiceId, string $paymentToken): bool
    {
        // ERROR 1: Lògica de negoci dins un servei genèric en lloc de l'entitat (Anemic Domain Model)
        $invoice = $this->invoiceRepository->find($invoiceId);
        if ($invoice->getStatus() == 'paid') { // ERROR 2: Ús de '==' enlloc de '===' pot portar a errors
            $this->logger->warning('Intent de pagar una factura ja pagada.', ['invoice_id' => $invoiceId]);
            return true;
        }

        $tax = 0.21; // ERROR 3: Valor Màgic (Hardcoded)
        $finalAmount = $invoice->getAmount() * (1 + $tax);

        try {
            // ERROR 4: Clau API Hardcodejada
            $apiKey = 'sk_test_12345ABCDE_hardcoded_key';
            
            $response = $this->client->request(
                'POST',
                'https://api.stripe.com/v1/charges',
                [
                    'headers' => ['Authorization' => 'Bearer ' . $apiKey],
                    'json' => [
                        'amount' => $finalAmount * 100, // Convertir a cèntims
                        'currency' => 'eur', // ERROR 5: Un altre Valor Màgic
                        'source' => $paymentToken,
                        'description' => 'Pagament per factura #' . $invoiceId,
                        'metadata' => ['invoice_id' => $invoiceId] // Risc de re-processament
                    ],
                ]
            );

            // ERROR 6: Falta de verificació robusta del pagament. Només comprova el codi d'estat.
            if ($response->getStatusCode() === 200) {
                // Actualitzar estat de la factura
                $invoice->setStatus('paid');
                $this->entityManager->flush();

                // ERROR 7: Procés síncron que afegeix latència
                $this->sendEmail($invoice); 

                return true;
            }
        } catch (\Exception $e) {
            // ERROR 8: Gestió d'errors deficient. S'empassa l'excepció.
            return false;
        }

        return false;
    }
    
    // ERROR 9: Codi duplicat per a la validació
    private function validateInvoice(Invoice $invoice) {
        if ($invoice->getAmount() <= 0) {
            $this->logger->error("Factura amb import invàlid.", ['invoice_id' => $invoice->getId()]);
            return false;
        }
        return true;
    }

    /**
     * Mètode per generar un informe.
     * Aquest mètode és cridat per una acció de controlador diferent.
     *
     * ERROR 10: La classe té més d'una responsabilitat (violació de SRP)
     */
    public function generateInvoicesReport(string $format = 'csv'): string
    {
        $data = $this->invoiceRepository->findAll();
        if ($format == 'csv') {
            $csv = "ID,Concepte,Import,Estat\n";
            foreach ($data as $invoice) {
                $csv .= "{$invoice->getId()},{$invoice->getConcept()},{$invoice->getAmount()},{$invoice->getStatus()}\n";
            }
            
            $vat = $invoice->getAmount() * 0.21;
            $vat += 0.5; // ERROR 11: Codi de negoci no relacionat amb la generació de l'informe
            $invoice->setVAT($vat);
            $this->invoiceRepository->save($invoice);

            return $csv;
        }
        // [...]
        return '';
    }

    private function sendEmail(Invoice $invoice)
    {
        $this->logger->info('Preparant per enviar email...');
        sleep(3); // Simula un servei de correu lent
        $this->logger->info("Email de confirmació enviat per a la factura #{$invoice->getId()}.");
    }
}
```

## Exercici 2 (2 punts)

L'empresa de comerç electrònic de moda "ModaFlow" està experimentant problemes greus a mesura que creix. 

La seva aplicació monolítica, on tot el codi està en un únic projecte, és cada cop més difícil de mantenir i escalar. Els equips de Catàleg (que gestiona productes, preus i estoc), Vendes (que gestiona comandes i promocions) i Clients (que gestiona usuaris i perfils) interfereixen constantment en el treball dels altres. 

Recentment, un canvi a la taula de \`Productes\` per part de l'equip de \`Catàleg\` va fer que el sistema de promocions de \`Vendes\` deixés de funcionar durant el Black Friday, causant pèrdues significatives. 

L'equip tècnic ha decidit migrar a una arquitectura de microserveis.

- **Per resoldre els conflictes entre equips, proposa una divisió lògica de l'antic monòlit en microserveis. Descriu breument quina seria la responsabilitat única i principal de cada microservei. (1 punt)**
  
- **A més, volen també aprofitar la divisió per oferir una API pública, com ho heu fet a la pràctica? (0.5 punt)**

- **El CEO, preocupat pels costos, suggereix que tots els microserveis continuïn utilitzant la mateixa base de dades centralitzada per estalviar en infraestructura i simplificar la generació d'informes. És una bona idea? (0.5 punts)**

## Exercici 3 (1 punts)

Una plataforma de streaming de videojocs, "GamerZone", té un èxit massiu. La seva arquitectura es compon de més de 100 servidors idèntics darrere d'un balancejador de càrrega. Tenen dos problemes de rendiment principals:

1\. La pàgina de perfil de cada usuari mostra una llista personal d'assoliments (trofeus). Aquesta consulta requereix JOINs complexos i es llegeix molt sovint, però s'actualitza poques vegades.

2\. La pàgina d'inici mostra el "Top 10 dels Streamers de la Setmana", una llista que és idèntica per a tots els usuaris i que es calcula amb una consulta extremadament pesada que triga diversos segons a executar-se.

- **Per a la llista del "Top 10 dels Streamers" (idèntica per a tothom), com ho arreglaries? Es pot forçar manualment el recàlcul dels Top 10, per exemple si hi ha alguna campanya de la plataforma. S'ha de fer algun canvi en aquest servei? (0.5 punt)**
  
- **La empresa ha llençat un videojoc propi "Chasing the AI on exams", que ha sigut un gran èxit. Entre els treballadors hi ha una preocupació sobre si algú pot copiar el videojoc i vendre'l com a seu. El CEO els hi ha dit que no es preocupin, que segons els drets de copyleft, que venen implícits amb qualsevol obra, això no ho poden fer. És correcta aquesta informació? (0.5 punt)**

## Exercici 4 (3 punts)

Una agència de viatges online, "ViatgesMon", té un procés de compra de vols que s'executa completament de forma síncrona i que triga uns 15 segons de mitjana. Els passos són:

1. L'usuari paga amb targeta.

2. El sistema verifica el pagament amb Stripe.

3. Es crida a l'API de l'aerolínia per confirmar el seient, aquest seient no és possible que s'esgoti ja que l'agència té unes places reservades. La crida es fa per enviar les dades del client.

4. Es genera un PDF amb el bitllet.

5. S'envia el PDF per correu electrònic.

Aquest flux està causant problemes: els usuaris abandonen la pàgina a mitges i de vegades es cobra al client per un vol que finalment no es pot reservar perquè l'API de l'aerolínia falla a l'últim moment.

- **Hi ha alguna part del procés de compra que canviaries? Explica-ho i què faries servir (1 punt)**
- **Que és un chargeback? Com pot afectar a "ViatgesMon" i què mesures es poden prendre per minimitzar el risc de chargebacks? (0.5 punts)**
- **Quina diferència hi ha entre fer servir Redsys i Stripe com a passarel·la de pagament? (0.5 punts)**
- **El banc us ha ofert una comissió de pagament per tarjeta del 0.5% (excepte per tarjetes Amercian Express que és del 1.5%). Stripe cobra un 1.4% + 0.25€ per transacció. Quina és la feina de Visa, Mastercard o American Express dins de l'ecosistema de pagaments? Com guanyen diners? (0.5 punts)**
- **La sessió de l'usuari es gestiona amb un JWT. Un desenvolupador suggereix posar totes les dades del perfil de l'usuari (nom, email, adreça) dins del payload del JWT i passar-ho a la url. Així s'aconsegueix estalviar un 80% de consultes a la base de dades i reduir un 30% els costos en servidors. Què opines? (0.5 punts)**

# Guia de correcció:

## Exercici 1 (4 punts)

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat:

- Obvis (Trobar-los tots són **2.5 punts** (0,5 punts per cada un))
  - Una clau d'API sensible està escrita directament al codi.
  - S'utilitzen valors fixos (com un tipus d'impost o una moneda) directament a la lògica sense ser constants o configurables.
  - A generateInvoicesReport, el bloc que calcula i persisteix el VAT queda fora del foreach, de manera que modifica només l'últim element de la llista (o falla si la llista és buida).
  - Una excepció crítica és capturada, però el programa la ignora.
  - Hi ha url hardcodejada, s'hauria de posar a un fitxer de configuració o variable d'entorn.
  
- Normal (Trobar-los tots és **1 punt** (0,2 punts per cada un))
  - La crida a find() no comprova si el resultat és null, cosa que provoca un error fatal si la factura no existeix. Cal comprovar el retorn de find() i retornar un error adient (p. ex. llençar una NotFoundException).
  - Una operació de llarga durada (com enviar un correu) bloqueja el flux principal d'execució (tenint al client esperant) i pot fallar.
  - El métode validateInvoice està escrit però no s'utilitza en cap lloc, al ser un mètode privat no pot ser cridat des de fora.
  - El mètode generateInvoicesReport té una responsabilitat addicional (calcular i persistir el VAT) que no està relacionada amb la generació de l'informe, violant el principi de responsabilitat única (SRP).
  - A generateInvoicesReport, es suma un valor fix (0.5) al VAT, cosa que no té sentit i indica que hi ha codi de negoci que no pertany a la generació de l'informe.
  
- Alta (Trobar-los tots és **0.5 punt** (0,125 punts per cada un))
  - La lògica de negoci important (com canviar l'estat d'una factura) està al controlador tal qual.
  - La classe té múltiples responsabilitats que no estan relacionades entre si.
  - El mètode doAction té un nom ambigu que no reflecteix la seva funcionalitat real (processar un pagament).
  - No hi ha cap mecanisme per prevenir que una mateixa petició de pagament es processi múltiples vegades si l'usuari, per error, la reenvia.
  - La verificació de l'estat (getStatus() == 'paid') i l'actualització posterior no estan dins d'una transacció atòmica Dos processos paral·lels que arribin simultàniament podrien superar la comprovació i cobrar la factura dues vegades (condició de carrera / race condition).

NO es pot tenir més de:
- Obvis: 2.5 punts
- Normal: 1 punt
- Alta: 0.5 punts

Si trobeu més d'alguna categoria, es comptarà només fins al màxim de punts d'aquesta categoria.

Molts de vosaltres heu posat coses que no es troben en aquesta llista orientativa, les he contat la gran majoria bé i classificat segons el meu criteri.

# Exercici 2

- **Per resoldre els conflictes entre equips, proposa una divisió lògica de l'antic monòlit en microserveis. Descriu breument quina seria la responsabilitat única i principal de cada microservei.? (1 punt)**

S'espera que l'alumne faci una divisió similar a la següent, encara que qualsevol variant serà correcta:

- **Microservei de Catàleg:** Responsable de tota la informació dels productes (descripcions, preus, imatges, estoc). La seva responsabilitat única és ser la font de la veritat sobre els productes.
- **Microservei de Vendes:** Responsable de gestionar el cicle de vida de les comandes, els carretons de la compra i l'aplicació de promocions.
- **Microservei de Clients:** Responsable de la gestió d'usuaris, perfils, autenticació i adreces.

Si l'alumne dona algun altre raonament, es considerarà vàlid.

Puntuació:
- 0.5p => Si la divisió és correcta.
- 0.5p => S'explica bé la responsabilitat de cada microservei.

- **A més, volen també aprofitar la divisió per oferir una API pública, com ho heu fet a la pràctica? (0.5 punts)**

S'espera que l'alumne  digui que a la pràctica s'ha fet una llibreria que conté els DTO; així ens protegim si el backend afegeix un paràmetre perquè l'API el necessita, ja que no afectarà el frontend perquè el DTO no el té.

Es donarà per correcte si ha mencionat que s'ha fet servir un ACL (Anti-Corruption Layer) o una estratègia similar per a protegir el frontend de canvis al backend.

- **El CEO, preocupat pels costos, suggereix que tots els microserveis continuïn utilitzant la mateixa base de dades centralitzada per estalviar en infraestructura i simplificar la generació d'informes. És una bona idea? (0.5 punts)**

Dos riscos greus són:

- **Acoblament Fort (Tight Coupling):** Si tots els serveis depenen del mateix esquema de base de dades, es perd l'autonomia. Un canvi a l'esquema requerit per un servei pot trencar tots els altres serveis. Això elimina el principal avantatge dels microserveis, que és poder desplegar i modificar equips de forma independent.
- **Escalabilitat Reduïda:** La base de dades es converteix en un coll d'ampolla centralitzat. Cada microservei pot tenir necessitats de dades diferents (uns més de lectura, altres d'escriptura). Una única base de dades no es pot optimitzar per a tots aquests patrons d'ús, limitant l'escalabilitat independent.

Si l'alumne dona algun altre raonament (base de dades master-slave amb suficient escalabilitat, etc.), es considerarà vàlid.

0.3p => Per mencionar els riscos d'acoblament fort i escalabilitat reduïda.
0.2p => Per donar una solució alternativa (base de dades per servei, estratègia de replicació, etc.).

# Exercici 3

- **Per a la llista del "Top 10 dels Streamers" (idèntica per a tothom), com ho arreglaries? Es pot forçar manualment el recàlcul dels Top 10, per exemple si hi ha alguna campanya de la plataforma. S'ha de fer algun canvi en aquest servei? (0,5 punts)**

El millor seria afegir memòria caché. S'espera que l'alumne s'adoni que tampoc pot ser de 2 nivells perquè, en recalcular-ho manualment, no s'actualitzaria. En recalcular, s'ha d'invalidar la memòria cau.

0.4p => Per mencionar la memòria caché.
0.1p => Per indicar que seria Redis. NO pot ser a nivell d'instància, perquè si es recalcula manualment no s'actualitzaria.

Si heu mencionat Apcu, només he comptat 0,25 Ja que NO es pot invalidar des de diferents instàncies, i això és un requisit per a aquest cas d'ús.

- **La empresa ha llençat un videojoc propi "Chasing the AI on exams", que ha sigut un gran èxit. Entre els treballadors hi ha una preocupació sobre si algú pot copiar el videojoc i vendre'l com a seu. El CEO els hi ha dit que no es preocupin, que segons els drets de copyleft, que venen implícits amb qualsevol obra, això no ho poden fer. És correcta aquesta informació? (0.5 punt)**

No, no és correcta. El copyleft és un tipus de llicència que permet a altres persones utilitzar, modificar i distribuir una obra, però amb la condició que qualsevol obra derivada també ha de ser distribuïda sota la mateixa llicència. El que és implícit en qualsevol obra és el **copyright**.

0,25p => explicar que el copyleft no ve implícit en qualsevol obra
0,25p => explicar que copyleft no és el que protegeix contra la còpia, sinó el copyright.

# Exercici 4

- **Hi ha alguna part del procés de compra que canviaries? Explica-ho i què faries servir (1 punt)**

Els candidats ideals per a un procés asíncron són:

- **Reserva del seient a l'API de l'aerolínia:** És una crida a un sistema extern, que pot ser lenta i propensa a errors. No és imprescindible per confirmar al client que el seu pagament s'ha rebut. Si l'alumne argumenta que pot fallar i llavors no es pot confirmar la reserva, es donarà per vàlid.
- **Generació del PDF i enviament del correu electrònic:** Aquesta és la tasca més lenta i menys crítica per a la confirmació immediata. Pot fallar per molts motius (servidor de correu caigut, etc.) i no hauria de bloquejar mai la resposta a l'usuari.

Si l'alumne ha dit que faria una cua de missatges (com RabbitMQ, Redis, etc.) per a aquestes tasques, es donarà per correcte. Si ha dit que faria servir un sistema de treballadors (com Symfony Messenger, Laravel Queues, etc.) també es donarà per correcte. Amb una cua ja és correcte, no cal fer dues cues separades.

0.5p => Per identificar correctament les parts que es poden fer asíncrones.
0.5p => Per proposar una solució asíncrona adequada (cua de missatges, sistema de treballadors, etc.).
  
- **Que és un chargeback? Com pot afectar a "ViatgesMon" i què mesures es poden prendre per minimitzar el risc de chargebacks? (0.5 punts)**

Un chargeback és una disputa iniciada per un titular de targeta que demana al seu banc, per exemple, pot alegar que no va autoritzar una compra o que no va rebre el producte. El banc pot revertir la transacció i retirar els fons del compte de l'empresa, cosa que pot causar pèrdues si el client ja ha rebut el servei (com un bitllet d'avió). 

Quan hi ha un chargeback, "ViatgesMon" ha de proporcionar proves al banc que la transacció va ser legítima (com registres de pagament, comunicacions amb el client, etc.). A més, pot implementar mecanismes com el pagament amb 3D Secure, verificació addicional d'identitat, i monitorització de transaccions sospitoses per minimitzar el risc de chargebacks.

Un chargeback NO és una devolució voluntària del client, sinó una disputa formal que pot resultar en la pèrdua de fons per a l'empresa, fins i tot després d'haver proporcionat el servei.

0.3p => Per explicar què és un chargeback i com pot afectar a l'empresa.
0.2p => Per mencionar mesures per minimitzar el risc de chargebacks.

- **Quina diferència hi ha entre fer servir Redsys i Stripe com a passarel·la de pagament? (0.5 punts)**

Stripe fa alhora de banc i passarel·la de pagament, mentre que Redsys és només una passarel·la que connecta amb diversos bancs. 

Per a treballar amb Stripe, només cal crear un compte i configurar-lo, mentre que amb Redsys cal tenir un acord amb un banc que ofereixi Redsys com a passarel·la.

- **El banc us ha ofert una comissió de pagament per tarjeta del 0.5% (excepte per tarjetes Amercian Express que és del 1.5%). Stripe cobra un 1.4% + 0.25€ per transacció. Quina és la feina de Visa, Mastercard o American Express dins de l'ecosistema de pagaments? Com guanyen diners? (0.5 punts)**

0.3p => Visa, Mastercard i American Express fan d'intermediari entre el banc del client i el banc de l'empresa. Proporcionen la infraestructura per a comunicar-se entre els diferents bancs.

0.2p => Guanyen diners cobrant una comissió per cada transacció que passa a través de la seva xarxa. Aquesta comissió pot variar segons el tipus de targeta (per exemple, les American Express solen tenir comissions més altes perquè ofereixen més avantatges als titulars de les seves targetes i tenen un model de negoci diferent).

- **La sessió de l'usuari es gestiona amb un JWT. Un desenvolupador suggereix posar totes les dades del perfil de l'usuari (nom, email, adreça) dins del payload del JWT i passar-ho a la url. Així s'aconsegueix estalviar un 80% de consultes a la base de dades i reduir un 30% els costos en servidors. Què opines? (0.5 punts)**

S'espera que l'alumne mencioni que no és una bona idea; ha de mencionar almenys:

- **Risc de Privacitat:** El contingut (payload) d'un JWT no està xifrat, només codificat en Base64. Qualsevol persona que intercepti el token pot llegir fàcilment tota la informació personal que conté, com el nom, l'email o l'adreça. És una fuita de dades personals innecessària.
- **Informació Mínima Indispensable:** El token només hauria de contenir l'identificador de l'usuari (userId o sub), la data de caducitat (exp) i potser els rols (roles). Qualsevol altra dada del perfil s'ha de consultar a la base de dades a través d'un endpoint segur, un cop l'usuari ha estat autenticat amb l'ID del token.