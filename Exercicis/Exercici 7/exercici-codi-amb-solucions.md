Treballes en una empresa que ofereix una plataforma SaaS de **gestió d'enviaments per a botigues online**. Un desenvolupador ha preparat un servei per generar etiquetes d'enviament i construir el panell de logística del backoffice.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

Aquest exercici val **4 punts**.

```
<?php

namespace App\Service;

use App\Repository\OrderItemRepository;
use App\Repository\OrderRepository;
use App\Repository\ShopRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;

class ShipmentService
{
    private $em;
    private $orderRepository;
    private $orderItemRepository;
    private $shopRepository;
    private $logger;

    public function __construct(
        EntityManagerInterface $em,
        OrderRepository $orderRepository,
        OrderItemRepository $orderItemRepository,
        ShopRepository $shopRepository,
        LoggerInterface $logger
    ) {
        $this->em = $em;
        $this->orderRepository = $orderRepository;
        $this->orderItemRepository = $orderItemRepository;
        $this->shopRepository = $shopRepository;
        $this->logger = $logger;
    }

    public function shipOrder(int $orderId, string $carrier, string $trackingEmail): bool
    {
        $order = $this->orderRepository->find($orderId);

        if ($carrier === 'standard') {
            $shippingCost = 4.95;
        } elseif ($carrier === 'express') {
            $shippingCost = 9.95;
        } elseif ($carrier === 'pickup') {
            $shippingCost = 0;
        } else {
            return false;
        }

        $labelUrl = 'https://carrier.internal/create-label?carrier=' . $carrier
            . '&order=' . $order->getId()
            . '&email=' . $trackingEmail
            . '&address=' . $order->getShippingAddress();

        $label = file_get_contents($labelUrl);

        $order->setCarrier($carrier);
        $order->setShippingCost($shippingCost);
        $order->setShipmentStatus('shipped');
        $order->setTrackingEmail($trackingEmail);
        $order->setShippedAt(new \DateTimeImmutable());

        $this->em->flush();

        $this->logger->info('Shipment created', [
            'order_id' => $orderId,
            'tracking_email' => $trackingEmail,
            'shipping_address' => $order->getShippingAddress(),
            'label' => $label,
        ]);

        mail(
            $trackingEmail,
            'Your order is on the way',
            'Tracking label: ' . $label
        );

        return true;
    }

    /**
     * Aquest mètode es mostra a la pantalla principal del mòdul de logística.
     * L'equip d'operacions l'obre constantment durant tot el dia.
     */
    public function getShippingDashboard(int $shopId): array
    {
        $orders = $this->orderRepository->findAll();
        $result = [];

        foreach ($orders as $order) {
            if ($order->getShopId() !== $shopId) {
                continue;
            }

            if ($order->getPaymentStatus() !== 'paid') {
                continue;
            }

            $shop = $this->shopRepository->find($order->getShopId());
            $items = $this->orderItemRepository->findBy(['orderId' => $order->getId()]);

            $weight = 0;
            foreach ($items as $item) {
                $weight += $item->getWeight() * $item->getUnits();
            }

            $result[] = [
                'order' => $order->getId(),
                'shop' => $shop->getName(),
                'email' => $order->getCustomerEmail(),
                'status' => $order->getShipmentStatus(),
                'weight' => $weight * 1000,
            ];
        }

        return $result;
    }

    public function retryFailedShipments(): void
    {
        $failedOrders = $this->orderRepository->findBy(['shipmentStatus' => 'label_error']);

        foreach ($failedOrders as $order) {
            $this->shipOrder($order->getId(), $order->getCarrier(), $order->getCustomerEmail());
        }
    }
}
```

## Avaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts** (0,5 punts per cada un).

- La crida a `find()` no comprova si la comanda existeix. Si retorna `null`, el codi fallarà quan intenti accedir a `getId()` o a l'adreça d'enviament.
- La integració amb el transportista es fa amb `file_get_contents()` concatenant directament dades dins de la URL. Això és fràgil, dificulta el tractament d'errors i pot exposar dades sensibles en logs, proxies o cachés intermèdies.
- Es registren dades personals i sensibles directament als logs (`tracking_email`, `shipping_address` i fins i tot el contingut de l'etiqueta), vulnerant el principi de minimització de dades.
- `getShippingDashboard()` carrega totes les comandes amb `findAll()` i les filtra en PHP, desaprofitant la base de dades i penalitzant una pantalla d'alt tràfic.
- Dins del bucle de `getShippingDashboard()` es fan consultes addicionals a `shopRepository` i `orderItemRepository` per cada comanda, generant un problema de **N+1 queries**.

### Normal

Trobar-los tots és **1 punt** (0,25 punts per cada un).

- S'utilitzen literals i números màgics (`standard`, `express`, `pickup`, `paid`, `shipped`, `label_error`, `4.95`, `9.95`, `1000`) directament a la lògica en lloc de constants, enums o configuració.
- Es fa servir `mail()` directament des del servei d'aplicació, acoblant la lògica de negoci a una implementació concreta d'infraestructura i dificultant les proves.
- No es valida el format de `trackingEmail` abans de persistir-lo, logar-lo i fer-lo servir per enviar notificacions.
- A `getShippingDashboard()`, el càlcul `weight * 1000` amaga una conversió o regla de negoci amb un número màgic sense cap context.

### Alta

Trobar-los tots és **0.5 punt** (0,125 punts per cada un).

- La classe barreja massa responsabilitats: creació d'enviaments, integració amb transportistes, notificacions, construcció del dashboard i reprocessament d'errors. Viola el principi **SRP (Single Responsibility Principle)**.
- El mètode `shipOrder()` modifica i persisteix l'estat de la comanda abans de saber si la notificació per correu s'ha completat correctament, deixant el sistema en un estat inconsistent si l'última part falla.
- `retryFailedShipments()` torna a executar el cas d'ús sense cap mecanisme d'idempotència, lock o protecció davant execucions simultànies, fet que pot generar etiquetes o notificacions duplicades.
- El dashboard retorna el correu del client encara que una vista logística sovint no necessita aquesta dada, exposant més informació personal de la necessària.
