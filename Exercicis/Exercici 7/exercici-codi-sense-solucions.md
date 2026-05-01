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
