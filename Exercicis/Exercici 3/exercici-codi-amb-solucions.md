Treballes en una empresa que ofereix una plataforma SaaS de **catàleg i inventari per a botigues online**. Un company ha intentat optimitzar la lectura del catàleg públic amb caché, però el resultat final és difícil de mantenir i està provocant incidències estranyes.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

```
<?php

namespace App\Service;

use App\Repository\ProductRepository;
use App\Repository\StockRepository;
use Doctrine\ORM\EntityManagerInterface;
use Predis\Client;
use Psr\Log\LoggerInterface;

class CatalogCacheService
{
    private $productRepository;
    private $stockRepository;
    private $redis;
    private $em;
    private $logger;
    private $apcuCacheManager;

    public function __construct(
        ProductRepository $productRepository,
        StockRepository $stockRepository,
        Client $redis,
        EntityManagerInterface $em,
        LoggerInterface $logger,
        ApcuCacheManager $apcuCacheManager
    ) {
        $this->productRepository = $productRepository;
        $this->stockRepository = $stockRepository;
        $this->redis = $redis;
        $this->em = $em;
        $this->logger = $logger;
        $this->apcuCacheManager = $apcuCacheManager;
    }

    public function getCatalog(int $shopId): array
    {
        $cacheKey = 'catalog_' . $shopId;

        $localCache = $this->apcuCacheManager->get($cacheKey);
        if ($localCache) {
            return $localCache;
        }

        $redisCache = $this->redis->get($cacheKey);
        if ($redisCache) {
            $decoded = json_decode($redisCache, true);
            $this->apcuCacheManager->set($cacheKey, $decoded, 3600);

            return $decoded;
        }

        $products = $this->productRepository->findAll();
        $result = [];

        foreach ($products as $product) {
            if ($product->getShopId() !== $shopId) {
                continue;
            }

            $x = 0;

            foreach ($product->getTags() as $tag) {
                $x = $x + 1;
            }

            $x = 0;

            foreach ($product->getImages() as $image) {
                $x = $x + 1;
            }

            foreach ($product->getVariants() as $variant) {
                $variantName = $variant->getName();
            }

            $stock = $this->stockRepository->findAvailableByProductId($product->getId());

            $result[] = [
                'id' => $product->getId(),
                'name' => $product->getName(),
                'price' => $product->getPrice(),
                'stock' => $stock,
                'last_variant' => $variantName ?? null,
            ];
        }

        $this->apcuCacheManager->set($cacheKey, $result, 3600);
        $this->redis->setex($cacheKey, 86400, json_encode($result));

        return $result;
    }

    public function updatePrice(int $productId, float $price): bool
    {
        $product = $this->productRepository->find($productId);

        $product->setPrice($price);
        $this->em->flush();

        $cacheKey = 'catalog_' . $product->getShopId();
        $this->redis->del([$cacheKey]);

        $this->logger->info('Price updated', [
            'product_id' => $productId,
            'new_price' => $price,
        ]);

        return true;
    }
}
```

## Avaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts** (0,5 punts per cada un).

- El servei fa servir **dues capes de caché diferents (APCu i Redis)** però, quan es modifica el preu d'un producte, només invalida Redis. APCu pot continuar retornant dades antigues, deixant el sistema amb resultats inconsistents segons quin servidor o procés atengui la petició.
- A `getCatalog()`, es fa un `findAll()` i després es filtra per `shopId` en PHP, fent treball innecessari i malgastant memòria.
- Dins del bucle principal es fa una consulta a `stockRepository` per cada producte, generant un problema de **N+1 queries**.
- Hi ha diversos `foreach` inútils (`tags`, `images`) que no aporten cap valor i, a més, es calcula una variable que se sobreescriu.
- La crida a `find()` de `updatePrice()` no comprova si el producte existeix (`null`), cosa que pot acabar en error fatal.

### Normal

Trobar-los tots és **1 punt** (0,25 punts per cada un).

- `$this->apcuCacheManager->get($cacheKey)` es comprova amb un `if ($localCache)` en lloc de distingir correctament entre "cache miss" i valors cachejats buits. Si el catàleg és un array buit, es considerarà fals i es recalcularà cada vegada.
- Redis i APCu utilitzen TTL diferents (`3600` i `86400`) per a la mateixa dada sense justificació clara, cosa que dificulta entendre el comportament del sistema.

### Alta

Trobar-los tots és **0.5 punt** (0,125 punts per cada un).

- El servei barreja massa responsabilitats: lectura del catàleg, estratègia de caché, escriptura de preus i escalfat massiu de catàlegs. Viola el principi **SRP (Single Responsibility Principle)**.
- La variable `$variantName` només conserva l'última variant recorreguda, fet que amaga una regla de negoci implícita i potencialment incorrecta.
