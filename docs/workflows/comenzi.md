bounded context: Comenzi
workflow: "Plasează comanda"
    triggered by: comanda "PlaceOrder"
    primary input: UnvalidatedOrder
    other input: catalogul de produse, 
                 stocul, 
                 valoarea minimă a comenzii, 
                 numărul următor de comandă, 
                 ora curentă
    output events: "OrderPlaced" 
                OR "PlaceOrderFailed"
    side-effects: comanda salvată în tabela Comenzi cu status Placed, 
                  stocul scăzut în aceeași tranzacție, 
                  Orders.Api apelează apoi Facturarea și Expedierea



// ---------- obiecte-valoare ----------
data CustomerEmail = string în format de e-mail
data ProductCode = string de forma PRD-0000
data Quantity = întreg între 1 și maximul pe linie
data Money = zecimal ≥ 0, cu 2 zecimale
data Currency = ???                                                 // doar RON?
data Address = County 
           AND City 
           AND Street 
           AND PostalCode (6 cifre)
data OrderNumber = string de forma ORD-000000



// ---------- intrări ----------
data UnvalidatedOrder = CustomerEmail: string
                    AND UnvalidatedAddress
                    AND list of UnvalidatedOrderLine
data UnvalidatedOrderLine = ProductCode: string 
                        AND Quantity: string
data UnvalidatedAddress = County: string 
                      AND City: string 
                      AND Street: string 
                      AND PostalCode: string


// ---------- stări ----------
data ValidatedOrder = CustomerEmail 
                  AND Address 
                  AND list of ValidatedOrderLine
data ValidatedOrderLine = ProductCode 
                      AND Quantity
data StockConfirmedOrder = ValidatedOrder   // aceleași date, dar stocul e garantat
data PricedOrder = StockConfirmedOrder 
               AND list of PricedOrderLine 
               AND Subtotal: Money 
               AND Total: Money
data PricedOrderLine = ProductCode 
                   AND Quantity 
                   AND UnitPrice: Money 
                   AND LineTotal: Money
data PlacedOrder = OrderNumber 
               AND PricedOrder 
               AND PlacedAt
data Order = UnvalidatedOrder 
          OR ValidatedOrder 
          OR StockConfirmedOrder 
          OR PricedOrder 
          OR PlacedOrder


// ---------- ieșiri ----------
data OrderPlaced = OrderNumber 
               AND CustomerEmail 
               AND Address 
               AND list of PricedOrderLine 
               AND Subtotal 
               AND Total 
               AND PlacedAt
data PlaceOrderFailed = ValidationFailed 
                     OR OutOfStock 
                     OR BelowMinimumValue
data ValidationFailed = list of ValidationError
data ValidationError = InvalidEmail 
                    OR InvalidProductCode 
                    OR InvalidQuantity 
                    OR InvalidAddress 
                    OR UnknownProduct
data OutOfStock = list of (ProductCode AND Requested AND Available)
data BelowMinimumValue = Total 
                     AND Minimum



// ---------- pași ----------
substep "Validează comanda" =
    input: UnvalidatedOrder
    output: ValidatedOrder OR ValidationFailed
    dependencies: productExists, now

    parsează e-mailul, adresa și fiecare linie în obiecte-valoare
    pentru fiecare linie: dacă produsul nu există în catalog -> UnknownProduct
    combină toate rezultatele (Result.Combine), ca să raportăm toate erorile odată
    dacă există vreo eroare -> ValidationFailed, altfel -> ValidatedOrder


substep "Verifică stocul" =
    input: ValidatedOrder
    output: StockConfirmedOrder OR OutOfStock
    dependencies: stockOf

    pentru fiecare linie: dacă Quantity > stockOf(ProductCode) -> reține produsul
    dacă lista e goală -> StockConfirmedOrder, altfel -> OutOfStock cu toate produsele


substep "Calculează prețul" =
    input: StockConfirmedOrder
    output: PricedOrder OR BelowMinimumValue
    dependencies: priceOf, minimumOrderValue

    pentru fiecare linie: LineTotal = priceOf(ProductCode) × Quantity
    Total = suma LineTotal
    dacă Total < minimumOrderValue -> BelowMinimumValue, altfel -> PricedOrder


// nu poate eșua
substep "Plasează" =
    input: PricedOrder
    output: PlacedOrder
    dependencies: nextOrderNumber, now

    atașează OrderNumber = nextOrderNumber și PlacedAt = now