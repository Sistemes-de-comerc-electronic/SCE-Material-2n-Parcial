```
<?php

namespace App\Service;

use App\Entity\Invoice;
use App\Repository\InvoiceRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;
use Symfony\Contracts\HttpClient\HttpClientInterface;

class PaymentProcessor
{
    private $entityManager;
    private $client; // HTTP Client
    private $logger;
    private $invoiceRepository;

    public function __construct(
        EntityManagerInterface $em,
        HttpClientInterface $httpClient,
        LoggerInterface $logger,
        InvoiceRepository $invoiceRepository
    ) {
        $this->entityManager = $em;
        $this->client = $httpClient;
        $this->logger = $logger;
        $this->invoiceRepository = $invoiceRepository;
    }

    /**
     * Processes the payment for an invoice and updates its status.
     * This method is called by a controller action after the user clicks "Pay".
     */
    public function doAction(int $invoiceId, string $paymentToken): bool
    {
        $invoice = $this->invoiceRepository->find($invoiceId);
        if ($invoice->getStatus() == 'paid') {
            $this->logger->warning('Attempt to pay an invoice that is already paid.', ['invoice_id' => $invoiceId]);
            return true;
        }

        $tax = 0.21;
        $finalAmount = $invoice->getAmount() * (1 + $tax);

        try {
            $apiKey = 'sk_test_12345ABCDE_hardcoded_key';
            
            $response = $this->client->request(
                'POST',
                'https://api.stripe.com/v1/charges',
                [
                    'headers' => ['Authorization' => 'Bearer ' . $apiKey],
                    'json' => [
                        'amount' => $finalAmount * 100, // Convert to cents
                        'currency' => 'eur', 
                        'source' => $paymentToken,
                        'description' => 'Payment for invoice #' . $invoiceId,
                        'metadata' => ['invoice_id' => $invoiceId]
                    ],
                ]
            );

            if ($response->getStatusCode() === 200) {
                // Update invoice status
                $invoice->setStatus('paid');
                $this->entityManager->flush();

                $this->sendEmail($invoice); 

                return true;
            }
        } catch (\Exception $e) {
            return false;
        }

        return false;
    }
    
    private function validateInvoice(Invoice $invoice) {
        if ($invoice->getAmount() <= 0) {
            $this->logger->error("Invoice with invalid amount.", ['invoice_id' => $invoice->getId()]);
            return false;
        }
        return true;
    }

    /**
     * Method to generate a report.
     * This method is called by a different controller action.
     *
     */
    public function generateInvoicesReport(string $format = 'csv'): string
    {
        $data = $this->invoiceRepository->findAll();
        if ($format == 'csv') {
            $csv = "ID,Concept,Amount,Status\n";
            foreach ($data as $invoice) {
                $csv .= "{$invoice->getId()},{$invoice->getConcept()},{$invoice->getAmount()},{$invoice->getStatus()}\n";
            }
            
            $vat = $invoice->getAmount() * 0.21;
            $vat += 0.5;
            $invoice->setVAT($vat);
            $this->invoiceRepository->save($invoice);

            return $csv;
        }
        // [...]
        return '';
    }

    private function sendEmail(Invoice $invoice)
    {
        $this->logger->info('Preparing to send email...');
        sleep(3); // Simulates a slow email sending process
        $this->logger->info("Confirmation email sent for invoice #{$invoice->getId()}.");
    }
}
```