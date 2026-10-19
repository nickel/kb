# Feedback del CFO sobre los early seeds, explicado para un developer

## 1. Intereses: se calculan sobre el calendario, no sobre la realidad

- **Bug principal:** el interés se calcula sobre el saldo *programado*, como si el cliente hubiera pagado puntualmente. El saldo baja en la fecha del calendario aunque el dinero no haya llegado. En código: el interés depende del `schedule`, cuando debería depender del saldo real día a día.
  - Préstamos 5 y 6: uno pagó con 21 días de retraso y otro con 36, y los dos generan exactamente 9.936,52 de intereses. Para el cliente, el retraso solo costó la late fee de 60 SEK.
  - Préstamo 10: se cobra interés sobre unos 76,8k cuando debía 92.582. Faltan unos 628 SEK.
- **Interés de demora (dröjsmålsränta):** las cuentas 1711/1718 existen pero nunca se usan, aunque §13 lo prevé. Según una investigación anterior, el devengo está desactivado a propósito y pendiente de una decisión regulatoria/de producto en `docs/pending/`. No lo he vuelto a comprobar hoy.
- **Waterfall inconsistente:** en los préstamos 5 y 6 la fee de 60 SEK sigue sin cobrar desde diciembre de 2024, mientras 19 pagos posteriores fueron a intereses y principal. El préstamo 10 sí cobró primero la fee. Hay dos caminos de código que asignan pagos de forma distinta.

## 2. Provisiones (ECL): calculadas una vez y mal

- **Nunca se actualizan:** los préstamos 1, 7 y 9 tienen una única provisión, la del día del desembolso, y ninguna en unos dos años. Falta la remeasurement periódica, o no se ejecuta para ellos.
- **Imposibles:**
  - Préstamo 9: provisión de 11.686 sobre un principal pendiente de 8.973. No se puede esperar perder más de lo que se debe.
  - Préstamo 1: la provisión es el 92% del saldo.
  - Los dos son Stage 1, grado A y con 0 DPD. Debería ser un invariante: `provisión ≤ EAD × LGD`.
- **Inconsistentes:**
  - Los préstamos 5 y 6, con el mismo saldo que el 1, tienen 78 y 63.
  - El préstamo 8 está en Stage 2 y tiene la cobertura más baja de todos (1,4%). Stage 2 es ECL de toda la vida, así que debería tener **más** cobertura que Stage 1.
- **Día uno:** con PD 3% y LGD 55%, la cuenta es `100k × 3% × 55% ≈ 1.650`.
  - El préstamo 2 provisiona 1.553: razonable.
  - El préstamo 4 provisiona 7.762 con los mismos PD y LGD, unas 5 veces más. Eso apunta a un multiplicador mal aplicado (horizonte, PD acumulada, etc.).
- **Dos fuentes de PD:** el modelo usa PDs del 19–31% en grado A, mientras que la tabla de stages dice 3%. O se elige una sola fuente, o, si de verdad la PD es tan alta, el test de SICR debería pasar esos préstamos a Stage 2.
- **Signo invertido (préstamo 3):** el cliente no devuelve el dinero en el plazo tras ejercer el desistimiento, y la provisión **baja** 5.624 al día siguiente. El riesgo ha subido, así que la provisión debería subir.

## 3. Modificaciones: el ajuste contable se mete en el contrato del cliente

- **Raíz:** la pérdida por modificación se registra directamente contra 1220. Desde ese momento, 1220 ya no es lo que debe el cliente: se ha mezclado el saldo contractual con el ajuste contable. El ajuste debería ir a una cuenta separada, de modo que `carrying = 1220 + ajustes`.
- **Préstamo 9 (el tipo baja del 8,95% al 7,95%):**
  - La cuota que paga el cliente (4.537) es correcta.
  - El ledger devenga a 8,94% sobre el saldo reducido: unos 667/mes, cuando el interés real del cliente es unos 593.
  - Desde IFRS 9 tiene sentido: la pérdida se va deshaciendo al EIR original.
  - **El problema:** FIN-68 reconstruye desde esos asientos el interés que se reporta a Skatteverket, y declararía unos 840 SEK de más. Se confunden dos cosas: el *ingreso contable* y el *interés contractual que paga el cliente*. Hacen falta dos fuentes distintas.
- **Préstamo 7 (el plazo pasa de 24 a 30 meses, mismo tipo):**
  - Se registra una pérdida de 1.985 más 1.035 de broker fee.
  - Como la pérdida redujo 1220 y la cuota nueva se calcula desde 1220, **el cliente acaba perdonado de unos 1.985**. La contabilidad se ha filtrado al contrato.
  - Señal de alarma: el APR es 7,62%, por debajo del nominal 8,95%. En un préstamo sin fees eso solo es posible si se ha condonado deuda.
  - El TCC (coste total del crédito) sigue en 9.947, el de originación; con los flujos nuevos sale 10.244. No se recalcula al modificar.
  - Además, el signo está mal: más plazo al 8,95%, descontado al 4,11%, da valor presente **mayor**. Debería ser una **ganancia**, no una pérdida.
- **Préstamos 8 y 9:** una "transaction cost modification gain" aumenta el broker fee diferido. Resultado: el gasto total en 4415 es 5.079 y 4.888, cuando a Lendo se le deben 4.850. Invariante: `Σ gasto 4415 == fee realmente pagado`.
- **Préstamo 7, clasificación:**
  - Una extensión de plazo debería marcarlo como forborne.
  - La vida esperada sigue en 18 meses y la fecha de fin del fee sigue en 2026-03-27, aunque el fee se agotó el 2025-09-06.
  - Son campos derivados que no se recalculan al modificar.

## 4. Stage 3 y write-off

- **Intereses en Stage 3:** se calculan sobre el neto, así que hay que recortar el % de cobertura del interés diario.
  - Préstamo 10: recorta un 26% con una cobertura del 57%, así que el ingreso queda inflado (FIN-73).
  - Préstamo 3: recorta un 59%, que sí cuadra con su cobertura.
  - Otra vez, el mismo cálculo da resultados distintos según el préstamo.
- **Charge-off vs write-off:**
  - Préstamo 3: "charged off" a 180 DPD, pero los 105.321 siguen en balance; solo ha cambiado el status.
  - Préstamo 10: "written off" y eliminado.
  - Dos conceptos para lo mismo con dos comportamientos. Hace falta una sola definición.
- **DPD del préstamo 10:** el Terms sheet muestra 90 (el umbral), pero el write-off fue a unos 185. Se muestra el valor equivocado.
- **Asiento de write-off:**
  - Correcto: `Dr Provisión / Cr Préstamo`, y solo el exceso sobre la provisión va a P&L (3142).
  - Actual: se libera la provisión contra P&L (ingreso) y se carga el saldo entero a 3142 (gasto). El P&L neto da lo mismo, pero las líneas brutas salen infladas.
  - La conciliación de provisiones de IFRS 7 exige una línea "importes dados de baja" y aquí aparecería como "liberación".

## Patrones de fondo

1. **Se mezcla estado contractual y estado contable:** 1220 con ajustes de modificación, el reporte a Skatteverket sacado del ledger IFRS, y el interés calculado sobre el calendario.
2. **Valores derivados que se calculan una vez y no se recalculan:** ECL, TCC, APR, vida esperada y fin del fee.
3. **El mismo concepto implementado en varios caminos:** waterfall, recorte de Stage 3, charge-off frente a write-off, y la fuente de PD.
4. **Faltan invariantes que convertirían esto en tests:**
   - `provisión ≤ EAD`
   - `Σ 4415 == fee pagado`
   - `APR ≥ nominal` si no hay fees
   - `1220 == principal contractual`
   - `cobertura Stage 2 ≥ Stage 1`
   - `recorte de Stage 3 == cobertura`

Encajarían bien en el golden tier de los early-book loans.

--

## Coverage per finding

```
1a/1c  Interest earned on the scheduled balance; late payers earn nothing extra
  Coverage: PARTIAL
  Refs:     finance-signoff/010 (CFO question 2), reg-20260907135057
  Missing:  Default interest is the only answer to lateness. No item asks
            whether ordinary interest should run on the actual balance.

1b     No default interest; accounts 1711/1718 never used
  Coverage: MOSTLY COVERED
  Refs:     reg-20261001171814 (blocked-on-com-38); decisions COM-9, COM-38,
            COM-43, BUS-63/66/67/69, FIN-49
  Missing:  Built but switched off (accrue_late_interest_operation.rb:8,
            ENABLED = false). The CFO cites §13 "under FIN-43", but FIN-43 is
            about withdrawal repayment order; the items cite the terms as
            "AV §11".

1d     Loan 10 earns interest on 76.8k instead of 92.6k
  Coverage: PARTIAL
  Refs:     010 question 2, FIN-65 (acceleration only)
  Missing:  Nothing covers interest on the actual overdue balance of a
            defaulted loan that has not been terminated.

1e     Late fee never collected
  Coverage: GAP
  Refs:     BUS-66 (Finance said OK on 2026-10-15), payments/20261013075447
  Missing:  BundledPaymentSplitCalculator pays the instalment first. It takes
            a fee only from leftover cash, and only when the leftover clears
            the fees exactly, so a normal instalment payment never touches the
            fee. BUS-66's only build hook is the default-interest work, which
            is switched off. Loan 10's "fee first" happened only because it
            paid exactly instalment plus fee.

2a     Provision never updated
  Coverage: SEED ARTEFACT + GAP
  Refs:     arch-20261016113817, arch-20261016114750 (close gate),
            arch-20261004174946 question 4
  Missing:  The seed has no nightly provisioning tick for early-book or clean
            loans; production re-measures every night
            (daily_arrears_batch_job.rb:49-71). The fallback-rate item says
            the seed ticks "about monthly", which is wrong: clean loans get no
            tick. Nothing alerts on a loan whose allowance has been held
            unchanged for a long time.

2b     Provision larger than remaining principal
  Coverage: GAP
  Refs:     same as 2a, plus finance-signoff/014, FIN-86
  Missing:  No cap or check that the allowance stays at or below the exposure.
            Follows from 2a: principal amortised while the allowance stayed
            frozen.

2c     Forborne Stage 2 loan has the lowest coverage
  Coverage: GAP
  Refs:     FIN-3 (finance-signoff/003), arch-20261004174946 question 6
  Missing:  No floor or benchmark comparing model Stage 2 coverage with
            Stage 1 or the stage table (0.35 x 0.55 = ~19%).

2d     Identical 100k loans start at 1,553 to 11,686
  Coverage: PARTIAL
  Refs:     finance-signoff/20261016135513, FIN-86, arch-20260903105837
  Missing:  The screen shows the stage table's PD/LGD (3%/55%) next to the
            model's figure; the existing item covers regulatory returns only.
            The simulated bureau PD is a hash of the consumer id (0.1-25%,
            unrelated to grade), which drives the spread. No item.

2e     19-31% PDs on grade A, Stage 1
  Coverage: PARTIAL
  Refs:     FIN-3, arch-20261001131508, arch-20260903105837
  Missing:  The significant-increase test is relative (3x origination PD), so
            a high PD alone moves nothing; that is by design, but nobody has
            explained it to the CFO. No item checks that grade and PD agree.

2f     Loan 3's provision drops after the missed repay-by date
  Coverage: GAP
  Refs:     arch-20261007082003, FIN-78
  Missing:  A withdrawn loan leaves the model's scope (ecl_measured_scope.rb:18)
            and falls back to the stage table at Stage 1, so the allowance
            drops. A missed repay-by date resets days past due and changes no
            stage. No item.

3a     Modification loss booked directly against 1220
  Coverage: GAP
  Refs:     near: FIN-31, FIN-32, reg-20260806075059
  Missing:  No item asks for a separate adjustment account so 1220 keeps
            showing what the customer owes.

3b     Rate cut: interest booked at the old rate; FIN-68 rebuild to
       Skatteverket overstated
  Coverage: PARTIAL
  Refs:     arch-20260914160159, FIN-68 item, arch-20260825110435 (KU
            reporting not built)
  Missing:  Nothing says KU and customer statements must use contractual
            interest, not journal interest that includes unwinding the
            modification loss. Latent until KU is built.

3c     Term extension forgives ~1,985; APR below the interest rate;
       wrong-looking sign
  Coverage: PARTIAL
  Refs:     REG-112, arch-20261015164548 (SV-04: Product says the term cannot
            change, yet extension exists), arch-009, reg-20260806070703
  Missing:  The code makes extensions cost-free on purpose (TermExtender
            refuses to add cost), so the loss sign and the APR below 8.95%
            follow from that design. Nobody has asked Finance or Product
            whether an extension should keep charging the contract rate. The
            TCC gap (9,947 vs 10,244) is untracked; the agent could not
            reproduce 10,244 from code.

3d     Broker-fee "modification gain" lifts 4415 above the 4,850 owed
  Coverage: GAP
  Refs:     near: FIN-72, FIN-45
  Missing:  This was a decided design (ADR-027 Decision 5, ADR-028
            Decision 3). Reopening it means amending ADR-027.

3e     Loan 7 not forborne; expected life and fee end date frozen
  Coverage: PARTIAL
  Refs:     arch-20260914125701 (FIN-27, revising expected life on running
            loans), reg-024, arch-20261007085922
  Missing:  No item asks whether an extension given to a borrower in
            difficulty should set forborne. The fee running out early
            (2025-09-06 vs 2026-03-27) is untracked; the cause needs the
            loan's data.

4a     Stage 3 adjustment 26% vs 57% coverage
  Coverage: PARTIAL
  Refs:     FIN-73 (finance-signoff/20261001055214), arch-20261016073303, 010
  Missing:  FIN-73 frames the issue as under 0.1%, but the CFO's gap is about
            31 points. Likely cause: the adjustment runs before the night's
            re-measurement (daily_arrears_batch_job.rb:64-69). It reuses the
            previous allowance, still the Stage 2 one on the first Stage 3
            night, for every catch-up day. It also divides by carrying amount
            where the model uses principal. Inferred from code, figures not
            reproduced.

4b     "Charged off" keeps the loan on the books, "written off" removes it
  Coverage: GAP
  Refs:     from the side only: FIN-24, FIN-66
  Missing:  Charge-off only changes status; the asset leaves only when a human
            approves the write-off. No deadline or alert for charged-off loans
            waiting for write-off, and the seed leaving loan 3 charged_off is
            not tracked as a defect.

4c     Terms sheet shows 90 days past due, write-off at ~185
  Coverage: GAP
  Refs:     none
  Missing:  days_past_due_calculator.rb:9-16 returns the figure frozen at
            default for written-off loans.

4d     Write-off releases the provision through P&L instead of using it
  Coverage: GAP
  Refs:     finance-signoff/012 assumes the current posting is correct
  Missing:  approve_write_off_operation.rb:155-234 reverses the provision, then
            charges the full balance to 3142. Net P&L is right, but both gross
            lines are inflated, and the IFRS 7 reconciliation (35H) has no
            write-off line. Nothing provides the 35L figure either. Fixing it
            changes the premise of item 012 and FIN-24.
```

## Summary

- **Already covered, waiting on decisions:** 1b (default interest), most of 2e, part of 3b and 3e (expected life, FIN-27), and 4a (FIN-73).
- **No item at all:** 1e (fees first on today's payments), 2b, 2f, 3a, 3b's Skatteverket point, 3d, 4b, 4c and 4d.
- **Seed artefacts first:** 2a and 2b come largely from the seed lacking a nightly provisioning tick, and 2d and 2e from the random simulated PD. Fix those before Finance reads provision figures from the seed again, or they will keep showing up as model defects.
- **The CFO is challenging a deliberate design in two places:** 3c (extensions are cost-free) and 3d (ADR-027 Decision 5). These need decision-register rows, not defect items.
- **The CFO's §13 citation needs correcting:** the feedback places it under FIN-43, which is about withdrawal repayment order. The items cite the terms as AV §11.
