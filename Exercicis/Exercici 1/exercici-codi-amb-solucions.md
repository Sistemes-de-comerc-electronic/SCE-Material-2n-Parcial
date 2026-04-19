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

//ERROR: La classe fa massa coses, no es pot separar en una per cada cosa? BookingCreationService, BookingCancellationService, BookingRetrievalService?
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

        //ERROR: Que és 0? Que és 1? S'han de fer servir constants
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
        $booking->setUserEmail($userEmail); //ERROR: que passa si ens posa "tonto el que lo lea"? S'hauria de validar l'email
        $booking->setPrice($price);
        $booking->setStatus('confirmed'); //ERROR: Constant?
        $booking->setCreatedAt(new \DateTime());

        $this->em->persist($booking);
        $this->em->flush();

        //ERROR: No s'ha de fer servir un fitxer per guardar logs, s'ha de fer servir el logger. Els fitxers s'esborren quan un contenidor es reinicia!
        file_put_contents('/tmp/bookings.log', $userEmail . ' booked room ' . $roomId . PHP_EOL, FILE_APPEND);

        return true;
    }

    public function cancelBooking(int $bookingId): bool
    {
        $booking = $this->bookingRepository->find($bookingId);

        if ($booking->getStatus() === 'cancelled') {
            //ERROR: Constant?
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
        //ERROR: No es podria paginar?
         //ERROR: Falta caché

        $result = [];

        foreach ($bookings as $booking) {
            if ($booking->getHotelId() !== $hotelId) {
                //ERROR: S'hauria de filtrar a la consulta
                continue;
            }

            //ERROR: CONSULTA DINS D'UN BUCLE
            $room = $this->roomRepository->find($booking->getRoomId());

            //ERROR: Que passa si no hi ha room? o és null? S'hauria de gestionar l'error

            $result[] = [
                'room' => $room->getName(),
                'email' => $booking->getUserEmail(),
                'status' => $booking->getStatus(),
                'price' => $booking->getPrice() * 7.5, //ERROR
            ];
        }

        return $result;
    }
}
```

---

## Avaluació

Es valorarà que l'alumne hagi estat capaç de detectar els següents elements al codi segons el nivell de complexitat.

Si trobeu algun altre que no està en aquesta llista, es considerarà vàlid i el professor el tindrà en compte segons la categoria que cregui més adequada (obvi, normal o alta).

>**Veus algun que falta?** No dubtis a posar-lo aquí, fer una branca amb la teva solució i fer un pull request perquè el professor el pugui revisar i afegir a la llista d'errors. Envieu l'enllaç del pull request al professor.


### Obvis (sense trobar aquests no es pot aprovar):

Trobar-los tots són **2.5 punts**

- S'utilitzen valors màgics (0, 1) i cadenes literals ("confirmed", "cancelled") directament a la lògica sense constants ni enumeracions, fent el codi fràgil i propens a errors.
- La consulta `findAll()` no està cachejada.
- Es fa una consulta a la base de dades dins d’un bucle (`roomRepository->find()`), generant el problema de **N+1 queries**.
- La crida a `find()` no comprova si el resultat és `null`, cosa que pot provocar errors fatals si l'entitat no existeix.
- Es fa servir `findAll()` per recuperar totes les dades sense paginació, cosa que pot provocar problemes greus de rendiment i memòria.
- El filtrat de reserves per `hotelId` es fa en PHP en lloc de fer-se a la base de dades, desaprofitant l’eficiència del motor SQL.

### Normal

Trobar-los tots és **1 punt** 

- El mètode `createBooking` utilitza un condicional `if/elseif` per definir els preus segons el tipus, violant el principi **OCP (Open/Closed Principle)**.
- No es valida l’entrada de dades (`userEmail`, `type`), cosa que pot provocar inconsistències o errors.


### Alta

Trobar-los tots és **0.5 punt**

- La classe viola el principi **SRP (Single Responsibility Principle)**, ja que gestiona creació, cancel·lació i consulta de reserves en una sola classe.
- Hi ha lògica de negoci amagada amb números màgics (`price * 7.5`). És difícil saber per què es multiplica per 7.5 i què representa aquest valor.
- No és recomanable treballar sobre el EntityManager, als laboratoris vam veure el métode save().
- Es fa servir accés directe al sistema de fitxers (`file_put_contents`) en lloc del sistema de logging de l’aplicació, fent el codi no portable i poc mantenible.