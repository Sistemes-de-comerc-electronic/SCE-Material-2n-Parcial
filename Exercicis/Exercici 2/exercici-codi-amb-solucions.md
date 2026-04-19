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

## Avaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts** (0,5 punts per cada un).

- S'utilitzen cadenes literals (`homepage`, `newsletter`, `ads`, `published`, `draft`) i pressupostos màgics (50, 100, 300) directament a la lògica en lloc de constants, enums o configuració.
- La crida a `find()` no comprova si el resultat és `null`, cosa que provocarà errors fatals si la promoció no existeix.
- A `getDashboard()`, es carrega tota la taula amb `findAll()` i es filtra en PHP per `shopId`, desaprofitant la base de dades i fent el codi ineficient.
- Dins del bucle de `getDashboard()`, es fan consultes addicionals a `shopRepository` i `productRepository` per cada promoció, generant problemes de **N+1 queries**.
- Es registren dades personals (`owner_email`) directament als logs, exposant informació que pot ser sensible i innecessària per al monitoratge tècnic.

### Normal

Trobar-los tots és **1 punt** (0,25 punts per cada un).

- El mètode `publishPromotion()` utilitza un `if/elseif` per decidir el pressupost segons el canal, violant el principi **OCP (Open/Closed Principle)**.
- Es fa servir la funció global `mail()` directament des del servei d'aplicació en lloc d'un servei d'infraestructura o d'un bus de missatges, fent el codi difícil de provar i mantenir.
- No es valida el format de `ownerEmail` abans de persistir-lo i utilitzar-lo en l'enviament de correu.
- A `getDashboard()`, el càlcul `* 1.21` amaga una regla de negoci o fiscal amb un número màgic sense context ni encapsulació.

### Alta

Trobar-los tots és **0.5 punt** (0,125 punts per cada un).

- La classe assumeix massa responsabilitats: publicar promocions, construir panells del backoffice i duplicar entitats. Això viola el principi **SRP (Single Responsibility Principle)**.
- `duplicatePromotion()` fa un `clone` directe d'una entitat Doctrine. Això és perillós perquè pot copiar estat no desitjat, relacions o identificadors i generar inconsistències amb el model de persistència.
- `publishPromotion()` persisteix l'estat de la promoció abans de confirmar que l'acció posterior d'enviament de correu hagi anat bé, deixant potencialment el sistema en un estat inconsistent.
- El panell del backoffice és una pantalla calenta i no hi ha ni paginació, ni caché, ni cap optimització específica per a una consulta d'alt tràfic.
