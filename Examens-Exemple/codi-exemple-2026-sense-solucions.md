Treballes en una empresa que ofereix una plataforma SaaS de **gestió de botigues en linia**. Ha arribat un company nou (que no ha cursat Sistemes de Comerç Electrònic) i l'heu posat a fer codi ja des del primer dia.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.


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