Treballes en una empresa que ofereix una plataforma SaaS de **sistema de gestió de reserves per hotels**. Un desenvolupador ha creat un nou servei per cancelar reserves i te l'ha fet arribar.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.


```
<?php

namespace App\Service\BookingsManager\BookingCancellator;

use App\Entity\Booking\Booking;
use App\Entity\Booking\BookingERPStatus;
use App\Entity\Booking\Payment;
use App\Entity\Booking\Status;
use App\Service\BookingsManager\ERPSender\ERPSenderService;
use App\Service\Integrations\Provider\ProviderServiceFactory;
use App\Service\MailingManager\GuestMailingManager\GuestMailingRequest;
use App\Service\MailingManager\GuestMailingManager\GuestMailingService;
use App\Service\MailingManager\InternalMailingManager\InternalMailingRequest;
use App\Service\MailingManager\InternalMailingManager\InternalMailingService;
use App\Service\PaymentManager\RefundManager\PaymentRefundRequest;
use App\Service\PaymentManager\RefundManager\PaymentRefundService;
use Doctrine\Persistence\ManagerRegistry;

class BookingCancellatorService
{

    private ManagerRegistry $doctrine;

    private PaymentRefundService $paymentRefundService;

    private ERPSenderService $erpSenderService;

    private GuestMailingService $guestMailingService;
    private ProviderServiceFactory $providerServiceFactory;
    private InternalMailingService $internalMailingService;

    public function __construct(
        ManagerRegistry $doctrine,
        PaymentRefundService $paymentRefundService,
        ERPSenderService $erpSenderService,
        GuestMailingService $guestMailingService,
        ProviderServiceFactory $providerServiceFactory,
        InternalMailingService $internalMailingService
    )
    {
        $this->doctrine = $doctrine;
        $this->paymentRefundService = $paymentRefundService;
        $this->erpSenderService = $erpSenderService;
        $this->guestMailingService = $guestMailingService;
        $this->providerServiceFactory = $providerServiceFactory;
        $this->internalMailingService = $internalMailingService;
    }
    public function execute(BookingCancellatorRequest $request)
    {
        $allowRefund = $request->isAllowRefund();

        $cancelledByProvider = $request->isCancelledByProvider();

        $booking = $request->getBooking();
        // Cancel the booking
        $statusCancelled = $this->doctrine
            ->getRepository(Status::class)
            ->findOneBy(['id' => Status::STATUS_CANCELLED]);

        $booking->setStatus($statusCancelled);

        // We check the cancellation deadline
        if (!$booking->isNotRefundable() && $allowRefund) {
            // Refund the payment
            $payments = $booking->getPayments()->filter(function (Payment $payment) {
                return $payment->getStatus() === Payment::STATUS_PAID;
            });
            foreach ($payments as $payment) {
                // Refund the payment
                $paymentRefundRequest = new PaymentRefundRequest();
                $paymentRefundRequest->setPayment($payment);
                $paymentRefundRequest->setAmount($payment->getAmount());

                $this->paymentRefundService->execute($paymentRefundRequest);
            }

            $booking->setTotalPrice(0);

            // we get all costs
            $costs = $booking->getProviderCosts();

            //we set all refundable costs to 0
            foreach ($costs as $cost) {
                /* @var $cost \App\Entity\Booking\BookingProviderCost */
                if ($cost->getProviderCost()->isRefundedOnCancellation()) {
                    $cost->setCostValue(0);
                }
            }

            $booking->recalculatePayout();
        }

        // Send ERP to provider if not cancelled by provider
        if (!$cancelledByProvider) {
            $provider = $booking->getProvider();
            $integrationInterface = $provider->getIntegration();

            $providerCancellationService = $this->providerServiceFactory->createProviderCancellationService($integrationInterface);

            if ($providerCancellationService) {
                try {
                    $providerCancellationService->sendBookingCancellation($booking);

                    $internalMailingRequest = new InternalMailingRequest();
                    $internalMailingRequest->setSubject('Booking Cancellation');
                    $internalMailingRequest->setMessage('Reserva con ID ' . $booking->getId() . ' ha sido cancelada en ' . $provider->getName(). ' en el admin');
                    $this->internalMailingService->execute($internalMailingRequest);
                } catch (\Exception $e) {
                    $internalMailingRequest = new InternalMailingRequest();
                    $internalMailingRequest->setSubject('Booking Cancellation Error');
                    $internalMailingRequest->setMessage('Error sending booking cancellation to provider ' . $provider->getName() . ' for booking ID ' . $booking->getId() . ': ' . $e->getMessage());

                    $this->internalMailingService->execute($internalMailingRequest);
                }
            } else{
                $internalMailingRequest = new InternalMailingRequest();
                $internalMailingRequest->setSubject('Booking Cancellation');
                $internalMailingRequest->setMessage('Reserva con ID ' . $booking->getId() . ' ha sido cancelada, cancela la reserva en ' . $booking->getProvider()->getName());

                $this->internalMailingService->execute($internalMailingRequest);
            }
        }

        $this->doctrine->getManager()->flush();

        if($request->isSendERPToProvider()){
            $this->erpSenderService->sendErp($booking, BookingERPStatus::ACTION_CANCEL);
        }

        if(!$request->isSendMailtoGuest()) {
            //send email to customer
            $mailRequest = new GuestMailingRequest();
            $mailRequest->setBooking($booking);
            $mailRequest->setType(GuestMailingRequest::TYPE_BOOKING_CANCELLATION);

            $this->guestMailingService->execute($mailRequest);
        }
    }
}

class BookingCancellatorRequest
{
    private bool $allowRefund = true;
    private bool $cancelledByProvider = false;
    private bool $cancelledByAdmin = false;
    private bool $cancelledByPropertyManager = false;
    private bool $cancelledByGuest = false;
    private bool $cancelledBySystem = false;
    private bool $sendERPToProvider = true;
    private bool $sendMailtoGuest = true;

    private ?Booking $booking = null;

    [...] //setters i getters per a tots els camps

}
```

