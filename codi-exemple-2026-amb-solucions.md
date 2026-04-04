Treballes en una empresa que ofereix una plataforma SaaS de **gestió de botigues en linia**. Ha arribat un company nou (que no ha cursat Sistemes de Comerç Electrònic) i l'heu posat a fer codi ja des del primer dia.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

Aquest exercici val **4 punts**.

```
<?php

namespace App\Service;

use App\Entity\Subscription;
use App\Repository\SubscriptionRepository;
use App\Repository\UserRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;
use Symfony\Contracts\HttpClient\HttpClientInterface;

class SubscriptionService
{
    private $entityManager;
    private $client; // HTTP Client
    private $logger;
    private $subscriptionRepository;
    private $userRepository;

    public function __construct(
        EntityManagerInterface $em,
        HttpClientInterface $httpClient,
        LoggerInterface $logger,
        SubscriptionRepository $subscriptionRepository,
        UserRepository $userRepository
    ) {
        $this->entityManager = $em;
        $this->client = $httpClient;
        $this->logger = $logger;
        $this->subscriptionRepository = $subscriptionRepository;
        $this->userRepository = $userRepository;
    }

    /**
     * Activa una subscripció per a un usuari i en configura els límits i la data de venciment.
     * Aquest mètode és cridat des del controlador un cop el pagament ha estat confirmat.
     */
    public function activate(int $subscriptionId, string $planType, string $userToken): bool
    {
        $subscription = $this->subscriptionRepository->find($subscriptionId);
        $user = $this->userRepository->find($subscription->getUserId());

        $this->logger->info('Activant subscripció per a l\'usuari.', [
            'subscription_id' => $subscriptionId,
            'user_token'      => $userToken,
        ]);

        if ($subscription->getStatus() === 'active') {
            $this->logger->warning('Intent d\'activar una subscripció ja activa.', ['id' => $subscriptionId]);
            return true;
        }

        if ($planType === 'basic') {
            $productLimit = 10;
        } elseif ($planType === 'professional') {
            $productLimit = 100;
        } elseif ($planType === 'enterprise') {
            $productLimit = 1000;
        } else {
            return false;
        }

        $expiresAt = new \DateTime('+30 days');

        $subscription->setProductLimit($productLimit);
        $subscription->setExpiresAt($expiresAt);
        $subscription->setStatus('active');

        $notifyUrl = 'https://notifications.internal.company.com/api/v2/subscription-activated';

        try {
            $response = $this->client->request('POST', $notifyUrl, [
                'json' => [
                    'user_id'         => $user->getId(),
                    'plan'            => $planType,
                    'product_limit'   => $productLimit,
                    'expires_at'      => $expiresAt->format('Y-m-d'),
                ],
            ]);

            if ($response->getStatusCode() === 200) {
                $this->entityManager->flush();
                return true;
            }
        } catch (\Exception $e) {
            $this->logger->info('Hi ha hagut un problema en activar la subscripció.', [
                'exception' => $e->getMessage(),
            ]);
        }

        return false;
    }

    public function getSubscription(int $subscriptionId): Subscription
    {
        $subscription = $this->subscriptionRepository->find($subscriptionId);

        if ($subscription->getExpiresAt() < new \DateTime()) {
            $subscription->setExpiresAt(new \DateTime('+30 days'));
            $subscription->setStatus('active');
            $this->entityManager->flush();
        }

        return $subscription;
    }

    /**
     * Genera un informe de les subscripcions actives amb les dades dels usuaris associats.
     */
    public function buildActiveReport(): array
    {
        $all = $this->subscriptionRepository->findAll();

        $report = [];
        foreach ($all as $subscription) {
            if ($subscription->getStatus() === 'active') {
                $user = $this->userRepository->find($subscription->getUserId());

                $report[] = [
                    'subscription_id' => $subscription->getId(),
                    'user_email'      => $user->getEmail(),
                    'plan'            => $subscription->getPlanType(),
                    'expires_at'      => $subscription->getExpiresAt()->format('Y-m-d'),
                    'product_limit'   => $subscription->getProductLimit(),
                ];
            }
        }

        return $report;

        $this->logger->info('Informe de subscripcions generat.', ['total' => count($report)]);
        return [];
    }
}

```

## Evaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts** (0,5 punts per cada un).

- S'utilitzen cadenes de text literals ("basic", "professional", "enterprise", "active") directament a la lògica sense ser constants ni enumeracions, fent el codi fràgil i propens a errors tipogràfics indetectables.
- El càlcul de la data de venciment utilitza +30 days en lloc d'un interval mensual real (+1 month o P1M). Per mesos de menys de 30 dies (febrer) o de 31, la data resultant serà incorrecta.
- La URL d'un servei intern de notificació està escrita directament al codi en lloc d'estar en un fitxer de configuració o variable d'entorn (.env), impedint canviar-la entre entorns (dev/staging/prod) sense tocar el codi.
- Una excepció crítica es capturada i registrada al nivell info en lloc d'error. Això fa que si el codi falla per això no es vegi.
- Dins el bucle de buildActiveReport, es llança una consulta independent a la base de dades per cada subscripció activa per obtenir l'usuari associat. Amb N subscripcions actives, això genera N+1 consultes (problema N+1 Queries). 

### Normal

Trobar-los tots és **1 punt** (0,2 punts per cada un).

- A buildActiveReport, tot el contingut de la taula es carrega en memòria (findAll()) per filtrar-lo posteriorment en PHP. En un sistema amb milers de subscripcions, això pot esgotar la memòria del servidor. La query hauria de filtrar directament per status='active'.
- El mètode activate conté un condicional if/elseif que enumera tots els tipus de plans possibles. Afegir un nou tipus de pla requereix modificar directament aquest mètode, violant el principi OCP (Open/Closed Principle). La solució seria usar una configuració externa, un mapa de plans o el patró Strategy.
- Es registren dades sensibles de l'usuari (un token d'autenticació) directament als logs del sistema. Qualsevol persona amb accés als logs pot obtenir tokens vàlids i suplantar la identitat d'usuaris.
- A buildActiveReport, hi ha codi mort (instruccions escrites després d'un return) que mai s'executarà. Indica que el codi no ha estat revisat ni testat adequadament i confon qualsevol desenvolupador que el llegeixi.

### Alta

Trobar-los tots és **0.5 punt** (0,125 punts per cada un).

- El mètode getSubscription() porta un nom que indica una consulta, però internament modifica l'estat de l'entitat i executa un flush() a la base de dades. Viola el principi CQS (Command Query Separation): els mètodes que retornen informació no han de tenir efectes secundaris. Cridar aquest mètode de manera innocent des d'un altre lloc pot provocar renovacions no desitjades i escriptures no esperades.
- La classe té dues responsabilitats clarament diferenciades: la gestió del cicle de vida de les subscripcions (activate, getSubscription) i la generació d'informes (buildActiveReport). Viola el principi SRP (Single Responsibility Principle), dificultant el manteniment i les proves unitaries independents.
- Si l'API de notificació falla després del flush, la subscripció queda marcada com a activa a la BBDD però el servei extern no ho sap, deixant l'estat inconsistent entre sistemes.
- La crida a find() no comprova si el resultat és null, cosa que saltarà un error si la subscripció no existeix.