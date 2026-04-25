Treballes en una empresa que ofereix una plataforma SaaS de **facturació recurrent per a comerços**. Un desenvolupador ha creat una comanda de Symfony per tancar subscripcions impagades i generar un fitxer de resum. La comanda fa moltes accions seguides i, si alguna falla, el sistema pot quedar a mig actualitzar.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica per què és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

Aquest exercici val **4 punts**.

```
<?php

namespace App\Command;

use App\Entity\Subscription;
use App\Repository\ShopRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;
use Symfony\Component\Console\Attribute\AsCommand;
use Symfony\Component\Console\Command\Command;
use Symfony\Component\Console\Input\InputArgument;
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

#[AsCommand(name: 'app:close-overdue-subscriptions')]
class CloseOverdueSubscriptionsCommand extends Command
{
    public function __construct(
        private EntityManagerInterface $em,
        private ShopRepository $shopRepository,
        private LoggerInterface $logger
    ) {
        parent::__construct();
    }

    protected function configure(): void
    {
        $this->addArgument('shopId', InputArgument::REQUIRED);
        $this->addArgument('reportPath', InputArgument::REQUIRED);
    }

    protected function execute(InputInterface $input, OutputInterface $output): int
    {
        $shop = $this->shopRepository->find($input->getArgument('shopId'));

        $rows = $this->em->getConnection()->fetchAllAssociative(
            "SELECT id, customer_email, amount_due FROM subscription WHERE shop_id = " . $shop->getId() . " AND status = 'overdue'"
        );

        $report = fopen($input->getArgument('reportPath'), 'w');
        fwrite($report, "subscription_id,customer_email,amount_due\n");

        foreach ($rows as $row) {
            $subscription = $this->em->find(Subscription::class, $row['id']);
            $subscription->setStatus('closed');
            $subscription->setClosedAt(new \DateTimeImmutable());

            $this->em->flush();

            mail(
                $row['customer_email'],
                'Subscription closed',
                'Your subscription has been closed because the payment is overdue.'
            );

            fwrite($report, $row['id'] . ',' . $row['customer_email'] . ',' . $row['amount_due'] . "\n");

            $this->logger->info('Closed subscription', $row);
        }

        fclose($report);

        return Command::SUCCESS;
    }
}
```

## Avaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts** (0,5 punts per cada un).

- La crida a `find()` no comprova si la botiga existeix. Si retorna `null`, el codi farà `getId()` sobre un valor nul i la comanda pot **crashejar** abans de començar.
- La consulta SQL es construeix concatenant directament valors dins la cadena, en lloc d'utilitzar consultes parametritzades.
- El resultat de `fopen()` no es valida. Si el fitxer no es pot obrir per permisos o per ruta incorrecta, la crida següent a `fwrite()` fallarà i la comanda caurà.
- La comanda persisteix canvis amb `flush()` dins del bucle i sense transacció. Si hi ha un error a mig procés, quedaran subscripcions tancades parcialment i altres no.
- Es registren directament dades personals (`customer_email`) als logs, exposant informació sensible innecessària.

### Normal

Trobar-los tots és **1 punt** (0,25 punts per cada un).

- Es fa servir `mail()` directament des de la comanda. Això acobla el cas d'ús a una infraestructura concreta i dificulta les proves i la gestió d'errors.
- La comanda carrega totes les files en memòria amb `fetchAllAssociative()`, cosa que pot ser problemàtica si la botiga té moltes subscripcions pendents.
- Es fan servir literals com `'overdue'` i `'closed'` directament al codi en lloc de constants o enums.
- No hi ha cap mecanisme de lock, idempotència o protecció contra execucions simultànies; dues execucions concorrents podrien trepitjar-se o duplicar treball.

### Alta

Trobar-los tots és **0.5 punt** (0,125 punts per cada un).

- La comanda barreja massa responsabilitats: lectura SQL, canvis d'estat, persistència, enviament de correus, generació d'informes i logging. Viola el principi **SRP (Single Responsibility Principle)**.
- L'ordre dels efectes laterals és arriscat: primer es persisteix l'estat de la subscripció i després es fan correu i escriptura a disc. Si una d'aquestes accions falla, el sistema queda en un estat inconsistent.
- No hi ha cap estratègia clara de rollback, recuperació o tancament segur del fitxer si es produeix una excepció durant el processament.
