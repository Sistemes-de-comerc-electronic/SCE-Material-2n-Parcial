Treballes en una empresa que ofereix una plataforma SaaS de **gestió de promocions per a marketplaces**. Un desenvolupador ha preparat una primera versió del servei que publica campanyes i construeix el panell de seguiment per a cada botiga.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

Aquest exercici val **4 punts**.

```
<?php

namespace App\Service;

use App\Entity\Promotion;
use App\Repository\ProductRepository;
use App\Repository\PromotionRepository;
use App\Repository\ShopRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;

class PromotionService
{
    private $em;
    private $promotionRepository;
    private $productRepository;
    private $shopRepository;
    private $logger;

    public function __construct(
        EntityManagerInterface $em,
        PromotionRepository $promotionRepository,
        ProductRepository $productRepository,
        ShopRepository $shopRepository,
        LoggerInterface $logger
    ) {
        $this->em = $em;
        $this->promotionRepository = $promotionRepository;
        $this->productRepository = $productRepository;
        $this->shopRepository = $shopRepository;
        $this->logger = $logger;
    }

    public function publishPromotion(int $promotionId, string $channel, string $ownerEmail): bool
    {
        $promotion = $this->promotionRepository->find($promotionId);

        if ($channel === 'homepage') {
            $budget = 50;
        } elseif ($channel === 'newsletter') {
            $budget = 100;
        } elseif ($channel === 'ads') {
            $budget = 300;
        } else {
            return false;
        }

        $promotion->setChannel($channel);
        $promotion->setBudget($budget);
        $promotion->setStatus('published');
        $promotion->setOwnerEmail($ownerEmail);
        $promotion->setPublishedAt(new \DateTime());

        $this->logger->info('Publicant promocio', [
            'promotion_id' => $promotionId,
            'owner_email' => $ownerEmail,
        ]);

        $this->em->flush();

        mail(
            'marketing@company.com',
            'Promotion published',
            $ownerEmail . ' has published ' . $promotion->getName()
        );

        return true;
    }

    /**
     * Aquest mètode s'utilitza a la pantalla principal del backoffice d'una botiga.
     * Els responsables de màrqueting l'obren constantment durant tot el dia.
     */
    public function getDashboard(int $shopId): array
    {
        $promotions = $this->promotionRepository->findAll();

        $result = [];

        foreach ($promotions as $promotion) {
            if ($promotion->getShopId() !== $shopId) {
                continue;
            }

            $shop = $this->shopRepository->find($promotion->getShopId());
            $products = $this->productRepository->findBy(['promotionId' => $promotion->getId()]);

            $totalRevenue = 0;
            foreach ($products as $product) {
                $totalRevenue += $product->getPrice() * $product->getSoldUnits();
            }

            $result[] = [
                'promotion' => $promotion->getName(),
                'shop' => $shop->getName(),
                'status' => $promotion->getStatus(),
                'products' => count($products),
                'revenue' => $totalRevenue * 1.21,
            ];
        }

        return $result;
    }

    public function duplicatePromotion(int $promotionId): Promotion
    {
        $promotion = $this->promotionRepository->find($promotionId);

        $copy = clone $promotion;
        $copy->setStatus('draft');
        $copy->setCreatedAt(new \DateTime());

        $this->em->persist($copy);
        $this->em->flush();

        return $copy;
    }
}
```
