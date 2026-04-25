Treballes en una empresa que ofereix una plataforma SaaS de **gestió de devolucions per a botigues online**. Un company ha afegit ràpidament una funció per carregar les devolucions pendents des de la base de dades perquè l'equip de suport les pugui revisar des del backoffice.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica per què és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

Aquest exercici val **4 punts**.

```
<?php

use PDO;

function loadPendingRefunds(PDO $connection, int $shopId, string $status, bool $includeCustomerData): array
{
    $sql = "SELECT * FROM refunds WHERE shop_id = $shopId AND status = '" . $status . "' ORDER BY created_at DESC";
    $refunds = $connection->query($sql)->fetchAll(PDO::FETCH_ASSOC);

    foreach ($refunds as $index => $refund) {
        if ($includeCustomerData) {
            $customerSql = "SELECT * FROM customers WHERE id = " . $refund['customer_id'];
            $refunds[$index]['customer'] = $connection->query($customerSql)->fetch(PDO::FETCH_ASSOC);
        }

        if ($refund['coupon_code']) {
            $couponSql = "SELECT discount_percent FROM coupons WHERE code = '" . $refund['coupon_code'] . "'";
            $coupon = $connection->query($couponSql)->fetch(PDO::FETCH_ASSOC);
            $refunds[$index]['coupon_discount'] = $coupon['discount_percent'];
        }

        $refunds[$index]['gross_amount'] = $refund['amount'] * 1.21;
        $refunds[$index]['can_be_approved'] = $refund['status'] === 'pending' && $refund['amount'] < 5000;
    }

    return $refunds;
}
```
