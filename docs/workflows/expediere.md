bounded context: Expediere
workflow: "Expediază comanda"
    triggered by: evenimentul "InvoiceIssued" (Comenzi trimite ShipmentRequestDto prin `POST /shipments` doar după ce factura a fost emisă)
    primary input: UnvalidatedShipment
    other input: județele deservite, 
                 greutatea maximă a unui colet, 
                 tarifele curierilor, AWB-ul următor, 
                 numărul de zile de livrare, 
                 ora curentă
    output events: "ShipmentInitiated" 
                OR "ShipOrderFailed"
    side-effects: expedierea salvată în baza Expediere,
                  ShipmentInitiatedDto întors către Comenzi



// ---------- obiecte-valoare ----------
data OrderNumber = string de forma ORD-000000   // tip propriu, nu cel din Comenzi
data InvoiceNumber = string de forma INV-000000 // tip propriu, nu cel din Facturare
data County = unul dintre cele 41 de județe sau București
data Address = County 
           AND City 
           AND Street 
           AND PostalCode
data Weight = zecimal > 0, în kilograme
data Parcel = ParcelNumber 
          AND Weight
data Courier = ???                               // cu ce curieri lucrăm?
data ShippingCost = Money ≥ 0
data Awb = string generat, format fix



// ---------- intrări ----------
data UnvalidatedShipment = OrderNumber: string 
                       AND InvoiceNumber: string
                       AND UnvalidatedAddress
                       AND list of UnvalidatedShipmentLine
data UnvalidatedShipmentLine = ProductCode: string 
                           AND Quantity: string 
                           AND UnitWeightKg: string


// ---------- stări ----------
data ValidatedShipment = OrderNumber 
                     AND InvoiceNumber 
                     AND Address
                     AND list of ValidatedShipmentLine
data ValidatedShipmentLine = ProductCode 
                         AND Quantity 
                         AND UnitWeight: Weight
data PackagedShipment = ValidatedShipment 
                    AND TotalWeight: Weight 
                    AND list of Parcel
data CourierAssignedShipment = PackagedShipment 
                           AND Courier 
                           AND ShippingCost
data ScheduledShipment = Awb 
                     AND CourierAssignedShipment 
                     AND CreatedAt 
                     AND EstimatedDelivery
data Shipment = UnvalidatedShipment 
             OR ValidatedShipment 
             OR PackagedShipment
             OR CourierAssignedShipment 
             OR ScheduledShipment


// ---------- ieșiri ----------
data ShipmentInitiated = Awb 
                     AND OrderNumber 
                     AND Courier 
                     AND list of Parcel
                     AND ShippingCost 
                     AND EstimatedDelivery
data ShipOrderFailed = ValidationFailed 
                    OR AddressNotServiced 
                    OR NoCourierAvailable
data ValidationFailed = list of ValidationError
data ValidationError = InvalidOrderNumber 
                    OR InvalidInvoiceNumber 
                    OR InvalidAddress
                    OR InvalidQuantity 
                    OR InvalidWeight 
                    OR NoLines
data AddressNotServiced = County
data NoCourierAvailable = TotalWeight 
                      AND County



// ---------- pași ----------
substep "Validează expedierea" =
    input: UnvalidatedShipment
    output: ValidatedShipment 
         OR ShipOrderFailed
    dependencies: servicedCounties

    parsează fiecare câmp în obiecte-valoare; dacă lista de linii e goală -> NoLines
    combină toate rezultatele (Result.Combine); dacă există erori -> ValidationFailed
    dacă County nu e în servicedCounties -> AddressNotServiced, altfel -> ValidatedShipment


// nu poate eșua (un produs mai greu decât maximul = ??? întrebăm expertul)
substep "Împachetează" =
    input: ValidatedShipment
    output: PackagedShipment
    dependencies: maxParcelWeight

    TotalWeight = suma UnitWeight × Quantity
    împarte produsele în colete, fiecare cu greutatea ≤ maxParcelWeight


substep "Alocă curierul" =
    input: PackagedShipment
    output: CourierAssignedShipment 
         OR NoCourierAvailable
    dependencies: courierRates

    păstrează tarifele care acceptă regiunea județului și greutatea coletelor
    dacă nu rămâne niciunul -> NoCourierAvailable
    altfel alege tariful cel mai ieftin; ShippingCost = preț × numărul de colete


// nu poate eșua
substep "Programează expedierea" =
    input: CourierAssignedShipment
    output: ScheduledShipment
    dependencies: nextAwb, deliveryDays, now

    atașează Awb = nextAwb, CreatedAt = now
    EstimatedDelivery = now + deliveryDays zile lucrătoare