Treballes en una empresa que ofereix una plataforma SaaS de **gestió de devolucions per a botigues online**. Un company ha afegit ràpidament una funció per carregar les devolucions pendents des de la base de dades perquè l'equip de suport les pugui revisar des del backoffice.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
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

## Avaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts** (0,5 punts per cada un).

- La funció construeix SQL concatenant directament valors (`$shopId`, `$status`, `customer_id`, `coupon_code`) en lloc d'utilitzar consultes parametritzades o prepared statements, exposant el sistema a **SQL injection**.
- Es fa servir `SELECT *`, recuperant més dades de les necessàries i acoblant el codi a l'estructura física de la taula.
- Dins del bucle principal es llancen consultes addicionals a `customers` i `coupons`, generant un problema de **N+1 queries**.
- El resultat de `query()` i `fetch()` no es comprova mai. Si una consulta falla o no retorna files, el codi pot acabar amb warnings o errors fatals.
- La funció carrega totes les devolucions d'una botiga sense cap límit ni paginació, cosa que pot penalitzar molt el rendiment del backoffice.

### Normal

Trobar-los tots és **1 punt** (0,25 punts per cada un).

- La mateixa funció fa massa coses: accés a dades, enriquiment amb client, lectura de cupons i càlcul de regles de negoci. Viola el principi **SRP (Single Responsibility Principle)**.
- Hi ha números màgics (`1.21` i `5000`) que amaguen regles fiscals o de negoci sense cap context ni encapsulació.
- El paràmetre booleà `$includeCustomerData` canvia substancialment el comportament de la funció, fet que és un indici de disseny poc clar (flag argument).
- Es retorna una estructura d'arrays directament lligada a l'esquema SQL, en lloc de treballar amb DTOs, objectes de domini o una capa de lectura més explícita.

### Alta

Trobar-los tots és **0.5 punt** (0,125 punts per cada un).

- Posar SQL inline dins la funció d'aplicació incrementa l'acoblament entre negoci i persistència i dificulta la reutilització, les proves i l'evolució del model.
- Si `coupon_code` o `customer_id` no tenen el format esperat, el codi assumeix que tot existirà i amaga errors de consistència de dades que després es manifestaran en producció.
- La funció pot exposar dades personals del client encara que la pantalla només necessiti una vista resumida, vulnerant el principi de minimització de dades.
