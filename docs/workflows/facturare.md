bounded context: Facturare
workflow: "Emite factura"
    triggered by: evenimentul "OrderPlaced" (primit ca OrderPlacedDto prin `POST /invoices` de la Comenzi)
    primary input: UnvalidatedInvoice
    other input: cota de TVA, numărul următor din serie, termenul de plată, ora curentă
    output events: "InvoiceIssued" OR "IssueInvoiceFailed"
    side-effects: factura salvată în baza Facturare,
                seria incrementată în aceeași tranzacție,
                InvoiceIssuedDto întors către Comenzi



// ---------- obiecte-valoare ----------
data OrderNumber = string de forma ORD-000000   // tip propriu, nu cel din Comenzi
data InvoiceNumber = string de forma INV-000000
data Money = zecimal ≥ 0, cu 2 zecimale
data VatRate = zecimal între 0 și 1
data PaymentTerm = număr de zile > 0
data BillingCustomer = ???                      // persoană fizică sau firmă cu CUI?



// ---------- intrări ----------
data UnvalidatedInvoice = OrderNumber: string 
                      AND CustomerEmail: string
                      AND UnvalidatedAddress 
                      AND DeclaredTotal: string
                      AND list of UnvalidatedInvoiceLine
data UnvalidatedInvoiceLine = ProductCode: string 
                          AND Quantity: string 
                          AND UnitPrice: string


// ---------- stări ----------
data ValidatedInvoice = OrderNumber 
                    AND CustomerEmail 
                    AND Address
                    AND list of ValidatedInvoiceLine
data ValidatedInvoiceLine = ProductCode 
                        AND Quantity 
                        AND UnitPrice: Money
data CalculatedInvoice = ValidatedInvoice 
                     AND list of CalculatedInvoiceLine
                     AND TotalNet: Money 
                     AND TotalVat: Money 
                     AND TotalGross: Money
data CalculatedInvoiceLine = ValidatedInvoiceLine 
                         AND Net 
                         AND Vat 
                         AND Gross
data IssuedInvoice = InvoiceNumber 
                 AND CalculatedInvoice 
                 AND IssuedAt 
                 AND DueDate
data Invoice = UnvalidatedInvoice 
            OR ValidatedInvoice 
            OR CalculatedInvoice 
            OR IssuedInvoice


// ---------- ieșiri ----------
data InvoiceIssued = InvoiceNumber 
                 AND OrderNumber
                 AND TotalNet 
                 AND TotalVat 
                 AND TotalGross 
                 AND IssuedAt 
                 AND DueDate
data IssueInvoiceFailed = ValidationFailed 
                       OR TotalMismatch
data ValidationFailed = list of ValidationError
data ValidationError = InvalidOrderNumber 
                    OR InvalidEmail 
                    OR InvalidAddress
                    OR InvalidQuantity 
                    OR InvalidPrice 
                    OR NoLines
data TotalMismatch = Expected: Money 
                 AND Received: Money



// ---------- pași ----------
substep "Validează datele facturii" =
    input: UnvalidatedInvoice
    output: ValidatedInvoice OR IssueInvoiceFailed
    dependencies: —

    parsează fiecare câmp în obiecte-valoare; dacă lista de linii e goală -> NoLines
    combină toate rezultatele (Result.Combine); dacă există erori -> ValidationFailed
    calculează suma UnitPrice × Quantity pe linii
    dacă suma ≠ DeclaredTotal -> TotalMismatch, altfel -> ValidatedInvoice


// nu poate eșua
substep "Calculează TVA" =
    input: ValidatedInvoice
    output: CalculatedInvoice
    dependencies: vatRate

    pentru fiecare linie: Net = UnitPrice × Quantity
                        Vat = Net × vatRate, rotunjit la 2 zecimale
                        Gross = Net + Vat
    TotalNet, TotalVat, TotalGross = sumele pe linii


// nu poate eșua
substep "Emite factura" =
    input: CalculatedInvoice
    output: IssuedInvoice
    dependencies: nextInvoiceNumber, paymentTerm, now

    atașează InvoiceNumber = nextInvoiceNumber, IssuedAt = now
    DueDate = now + paymentTerm zile