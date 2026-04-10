Treballes en una empresa que ofereix una plataforma SaaS de **gestió de reserves per a hotels**. Un nou desenvolupador ha implementat la lògica de reserves sense gaire experiència en bones pràctiques.

Realitza una revisió exhaustiva del fragment de codi proporcionat a continuació. Per cada error que trobis:
1.	**Descriu el problema:** Identifica la mala pràctica o l'error conceptual.
2.	**Explica perquè és un problema:** Justifica la teva observació fent referència als principis, patrons o conceptes vistos a classe (p. ex., SOLID, DDD, seguretat, rendiment, gestió d'errors, etc.) i explica les seves conseqüències negatives.
3.	**Proposa una solució:** Descriu clarament com refactoritzaries o corregiries el codi per solucionar el problema.

Si no trobes cap error pots posar **"LGTM" (Looks Good To Me)**.

```
<?php

namespace App\Service;

use App\Entity\Booking;
use App\Repository\BookingRepository;
use App\Repository\RoomRepository;
use Doctrine\ORM\EntityManagerInterface;
use Psr\Log\LoggerInterface;

class BookingService
{
    private $em;
    private $bookingRepository;
    private $roomRepository;
    private $logger;

    public function __construct(
        EntityManagerInterface $em,
        BookingRepository $bookingRepository,
        RoomRepository $roomRepository,
        LoggerInterface $logger
    ) {
        $this->em = $em;
        $this->bookingRepository = $bookingRepository;
        $this->roomRepository = $roomRepository;
        $this->logger = $logger;
    }

    public function createBooking(int $roomId, string $userEmail, int $type): bool
    {
        $room = $this->roomRepository->find($roomId);

        if ($type === 0) {
            $price = 100;
        } elseif ($type === 1) {
            $price = 200;
        } else {
            return false;
        }

        if ($room->isAvailable() === false) {
            return false;
        }

        $booking = new Booking();
        $booking->setRoomId($roomId);
        $booking->setUserEmail($userEmail);
        $booking->setPrice($price);
        $booking->setStatus('confirmed');
        $booking->setCreatedAt(new \DateTime());

        $this->em->persist($booking);
        $this->em->flush();

        file_put_contents('/tmp/bookings.log', $userEmail . ' booked room ' . $roomId . PHP_EOL, FILE_APPEND);

        return true;
    }

    public function cancelBooking(int $bookingId): bool
    {
        $booking = $this->bookingRepository->find($bookingId);

        if ($booking->getStatus() === 'cancelled') {
            return true;
        }

        $booking->setStatus('cancelled');
        $this->em->flush();

        return true;
    }

    /**
    Aquest mètode es fa servir des de la pantalla que un hotel pot veure totes les seves reserves. Hi ha molt de tràfic en aquesta pantalla, per tant és important que sigui eficient.
    */
    public function getAllBookings(int $hotelId): array
    {
        $bookings = $this->bookingRepository->findAll();

        $result = [];

        foreach ($bookings as $booking) {
            if ($booking->getHotelId() !== $hotelId) {
                continue;
            }

            $room = $this->roomRepository->find($booking->getRoomId());

            $result[] = [
                'room' => $room->getName(),
                'email' => $booking->getUserEmail(),
                'status' => $booking->getStatus(),
                'price' => $booking->getPrice() * 7.5,
            ];
        }

        return $result;
    }
}
```