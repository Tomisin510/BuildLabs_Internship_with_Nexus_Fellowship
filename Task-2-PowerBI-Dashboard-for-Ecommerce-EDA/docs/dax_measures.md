# DAX measures and model documentation

**Project:** UK E-Commerce Sales & Customer Performance  
**Author:** Tomisin Obijole  
**Programme:** BuildLabs / Nexus Fellowship - Task 2  
**Documentation date:** 7 October 2026

## Status and evidence

This document records the DAX formulas established in the project conversation and the screenshot-supported results. The completed `.pbix` was not supplied for inspection. Compare these definitions with the formulas in the final file before describing them as an audited extraction. They are complete implementation documentation for the agreed measures.

## Data scope

- Source: [UK Online Retail dataset](https://www.kaggle.com/datasets/carrie1/ecommerce-data).
- Actual coverage: 1 December 2010 to 9 December 2011. December 2011 is partial.
- Grain: one invoice line, not one complete order.
- `FactSales`: `positive_sales.csv`, 524,878 retained positive invoice lines.
- `DimProduct`: `product_summary.csv`, reduced to descriptive fields and full-period classification flags. Aggregated sales, units and invoice counts are not used as fact measures.
- Stock codes were normalised to uppercase in Python before reaggregation to resolve observed case variants such as `35809A` / `35809a`. Both fact and product exports must be refreshed together.
- Positive quantity and unit price are required. Exact raw-row duplicates were removed before the subsequent transformations. No further fact-row deduplication is implied by stock-code normalisation.
- Non-merchandise classification covers codes `DOT`, `POST`, `BANK CHARGES`, `AMAZONFEE`, `B`, and `M`, and documented matching descriptions. Classification is analytical judgment, not proof that every other code is ordinary physical merchandise; retained vouchers/samples may need separate analysis for stock planning.
- Missing CustomerIDs remain in overall values but not in identified-customer metrics.
- Negative entries are absent from this fact table: none of these measures calculates net sales or profit.

## Relationships and types

| One side | Many side | Settings |
|---|---|---|
| `DimDate[Date]` | `FactSales[OrderDate]` | Active, 1:*, single direction from dimension to fact |
| `DimProduct[StockCode]` | `FactSales[StockCode]` | Active, 1:*, single direction from dimension to fact |

`InvoiceNo` and `StockCode` must be Text. Preserve `A563185`; numeric conversion causes an error and is not a reason to delete the row. `CustomerID` can be Whole Number with blanks and must not be summed. `OrderDate` is Date, `InvoiceDate` is Date/Time, `Quantity` is Whole Number, `LineSales` and `UnitPrice` are Decimal Number, and `IsNonMerchandise` is True/False. Format money at display time; do not round each line before summation.

The product dimension contains named-merchandise codes, so unmatched non-merchandise fact codes can create a blank product member. Hide the blank in the portfolio slicer only; do not delete fact records to remove it.

## Measure behaviour

Create each formula separately with **New measure**. All measures respond to active relationship filters and directly applied fact filters, including date, country, product and review status. `KEEPFILTERS` intersects the metric's scope with existing filters. `DIVIDE` returns blank for a zero denominator by default.

Distinct counts are non-additive: invoice/customer/product counts across groups may overlap and should not be summed. Currency totals can differ by a penny when separately rounded subgroup values are added. Validation should use unrounded source aggregates with a small floating-point tolerance.

## Recorded Positive Value

**Purpose:** Recorded value of all retained positive invoice lines. Includes separately classified charges and accounting entries.

```dax
Recorded Positive Value =
SUM ( FactSales[LineSales] )
```

**Format:** GBP, 2 decimals.  
**Validation:** £10,642,110.80.

## Merchandise Sales

**Purpose:** Recorded value restricted to named merchandise. Excludes classified non-merchandise entries and missing descriptions.

```dax
Merchandise Sales =
CALCULATE (
    [Recorded Positive Value],
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** GBP, 2 decimals.  
**Validation:** £10,255,019.18.

## Positive Invoices

**Purpose:** Distinct invoice identifiers represented by retained positive lines; an order proxy, not a count of rows.

```dax
Positive Invoices =
DISTINCTCOUNT ( FactSales[InvoiceNo] )
```

**Format:** Whole number.  
**Validation:** 19,960.

## Value per Invoice

**Purpose:** Recorded positive value divided by distinct positive invoices within the same filter context. Not the average invoice-line value.

```dax
Value per Invoice =
DIVIDE ( [Recorded Positive Value], [Positive Invoices] )
```

**Format:** GBP, 2 decimals.  
**Validation:** £533.17.

## Invoice Lines

**Purpose:** Number of retained invoice-line records.

```dax
Invoice Lines =
COUNTROWS ( FactSales )
```

**Format:** Whole number.  
**Validation:** 524,878.

## Identified Customers

**Purpose:** Distinct nonblank CustomerIDs represented in the selected positive-value records.

```dax
Identified Customers =
CALCULATE (
    DISTINCTCOUNT ( FactSales[CustomerID] ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** Whole number.  
**Validation:** 4,338.

## Identified Customer Value

**Purpose:** Recorded positive value associated with a nonblank CustomerID. Includes identified non-merchandise entries.

```dax
Identified Customer Value =
CALCULATE (
    [Recorded Positive Value],
    KEEPFILTERS ( NOT ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** GBP, 2 decimals.  
**Validation:** £8,887,208.89.

## Unidentified Customer Value

**Purpose:** Recorded positive value for rows without CustomerID. It is unassigned value, not one anonymous customer.

```dax
Unidentified Customer Value =
CALCULATE (
    [Recorded Positive Value],
    KEEPFILTERS ( ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** GBP, 2 decimals.  
**Validation:** Approximately £1.75m; chat reconciliation gives £1,754,901.92.

## Unidentified Value Share

**Purpose:** Unidentified-customer value divided by all recorded positive value within the current selection.

```dax
Unidentified Value Share =
DIVIDE ( [Unidentified Customer Value], [Recorded Positive Value] )
```

**Format:** Percentage, 1 decimal.  
**Validation:** Approximately 16.5%.

## Merchandise Units

**Purpose:** Units on named-merchandise positive lines. Retains the investigated large positive lines unless the review filter excludes them.

```dax
Merchandise Units =
CALCULATE (
    SUM ( FactSales[Quantity] ),
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** Whole number.  
**Validation:** 5,561,564.

## Merchandise Invoices

**Purpose:** Distinct invoices containing retained named-merchandise lines. An invoice with several products is counted once in a total.

```dax
Merchandise Invoices =
CALCULATE (
    [Positive Invoices],
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** Whole number.  
**Validation:** Screenshot is rounded to 20K; exact count not verified.

## Active Merchandise Products

**Purpose:** Distinct merchandise codes present in the fact records selected by filters. Count from FactSales so dates and countries affect the result.

```dax
Active Merchandise Products =
CALCULATE (
    DISTINCTCOUNT ( FactSales[StockCode] ),
    KEEPFILTERS ( FactSales[IsNonMerchandise] = FALSE () ),
    KEEPFILTERS ( NOT ISBLANK ( FactSales[Description_clean] ) )
)
```

**Format:** Whole number.  
**Validation:** Screenshot is rounded to 4K; use refreshed post-normalisation notebook count, not the earlier 3,915.

## Identified Invoices

**Purpose:** Distinct invoices represented by identified-customer lines. Numerator and denominator for identified invoice value use the same subset.

```dax
Identified Invoices =
CALCULATE (
    [Positive Invoices],
    KEEPFILTERS ( NOT ISBLANK ( FactSales[CustomerID] ) )
)
```

**Format:** Whole number.  
**Validation:** Netherlands: 94 with only Netherlands selected, full period.

## Identified Value per Invoice

**Purpose:** Identified-customer value divided by identified invoices. Use this for the international comparison, not Value per Invoice.

```dax
Identified Value per Invoice =
DIVIDE ( [Identified Customer Value], [Identified Invoices] )
```

**Format:** GBP, 2 decimals.  
**Validation:** Netherlands: £3,036.66 with Netherlands selected, full period.

## Top 5 Customer Value Share

**Purpose:** Share of selected identified-customer value attributable to the five highest-value customers in that selection. CustomerID is a deterministic tie-breaker.

```dax
Top 5 Customer Value Share =
VAR CustomerValues =
    FILTER (
        ADDCOLUMNS (
            VALUES ( FactSales[CustomerID] ),
            "@Value", [Identified Customer Value]
        ),
        NOT ISBLANK ( FactSales[CustomerID] )
    )
VAR TopCustomers =
    TOPN (
        5,
        CustomerValues,
        [@Value], DESC,
        FactSales[CustomerID], ASC
    )
RETURN
    DIVIDE (
        SUMX ( TopCustomers, [@Value] ),
        [Identified Customer Value]
    )
```

**Format:** Percentage, 1 decimal.  
**Validation:** 11.8%.

### Important top-five denominator rule

Use `Top 5 Customer Value Share` on a summary card. It ranks within the current context, including any customer selection. On a visual row containing one CustomerID it can show 100%, because the ranking population has narrowed to that customer. It is not intended as each customer's share of a wider population. Do not use the 11.8% full-period result as fixed text beside a filtered view.

## Calculated columns

### ReversalReviewStatus - FactSales

```dax
ReversalReviewStatus =
IF (
    FactSales[InvoiceNo] IN { "581483", "541431" },
    "Known matching negative entry",
    "Other positive lines"
)
```

This static review flag identifies only the two investigated positive invoices. Matching source records are `C581484` (-80,995 units at £2.08, 12 minutes later) and `C541433` (-74,215 units at £1.04, 16 minutes later). It is not an automated return-matching algorithm or formal source linkage. Default slicer selection is All. Selecting Other positive lines is a sensitivity view, not net sales.

### CustomerLabel - FactSales

```dax
CustomerLabel =
IF (
    ISBLANK ( FactSales[CustomerID] ),
    BLANK (),
    FORMAT ( FactSales[CustomerID], "0" )
)
```

Use this text label on customer bar-chart axes to prevent a continuous numeric axis. Exclude blank labels and set Top 10 by Identified Customer Value.

## Calculated date table

```dax
DimDate =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2010, 1, 1 ), DATE ( 2011, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "MonthNumber", MONTH ( [Date] ),
    "MonthName", FORMAT ( [Date], "MMM" ),
    "YearMonth", FORMAT ( [Date], "yyyy-MM" ),
    "YearMonthSort", YEAR ( [Date] ) * 100 + MONTH ( [Date] )
)
```

Mark DimDate as a date table using Date. Sort YearMonth by YearMonthSort and MonthName by MonthNumber. Calendar coverage is wider than observed data; default slicers should use 1 December 2010 to 9 December 2011. Filter monthly visuals to nonblank Recorded Positive Value to avoid empty pre-study months. No month-over-month measure is documented here because none was established for the completed report; do not compare partial December with full November as an ordinary decline.

## Power Query display column

`DimProduct[ProductLabel]` is a Power Query custom column, not DAX:

```powerquery
[StockCode] & " | " & [Description]
```

`RelativePosition`, `Rare_5_Invoices`, `Low_Sales_100`, and `Low_Units_50` originate from full-period Python aggregation. Their classifications remain fixed under report filters, although the associated fact measures recalculate. Refreshing source data requires rerunning the classifications.

## Validation checklist

Clear date/country/product/review selections and cross-highlights, then check:

| Check | Expected |
|---|---:|
| Invoice Lines | 524,878 |
| Recorded Positive Value | £10,642,110.80 |
| Merchandise Sales | £10,255,019.18 |
| Positive Invoices | 19,960 |
| Value per Invoice | £533.17 |
| Identified Customers | 4,338 |
| Identified Customer Value | £8,887,208.89 |
| Merchandise Units | 5,561,564 |
| Top 5 Customer Value Share | 11.8% |

Exact counts and amounts above use the notebook/chat reconciliation; screenshots round several cards. The revised active-product count and merchandise-invoice count need the final refreshed source or PBIX, not rounded 4K/20K cards.

With **Other positive lines** selected, all countries and full actual coverage:

| Check | Expected |
|---|---:|
| Recorded Positive Value | £10,396,457.60 |
| Merchandise Sales | £10,009,365.98 |
| Identified Customer Value | Approximately £8,641,555.69 |

This subtracts £245,653.20; it does not incorporate all returns. In a Netherlands-only full-period selection, verify 94 identified invoices, nine identified customers and £3,036.66 identified value per invoice. Test date and portfolio filters, then restore the baseline before saving screenshots/PDFs.

## Report usage

- Overview: overall recorded value, merchandise value, invoice activity and geography.
- Products: merchandise measures with DimProduct labels; classifications are full-period.
- Customers: nonblank CustomerIDs; international comparison uses identified numerator and denominator and at least ten identified invoices.
- Definitions: coverage, assumptions and the two investigated matching negative entries.
- A PDF is static. Use the `.pbix` for slicers and drill interaction.

## Reproduction and refresh

Run the Task 1 notebook top to bottom with its source CSV. Re-export `positive_sales.csv` and `product_summary.csv` into the project data folder. In Power BI, update Source locations using Power Query/data-source settings and refresh both queries. Confirm types, unique product keys, active relationships and baseline values. Save the PBIX with default filters restored. GitHub hosts the file for download; it does not render it as a live Power BI report.

## References

- [Dataset](https://www.kaggle.com/datasets/carrie1/ecommerce-data)
- [Power BI star-schema guidance](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema)
- [Date tables](https://learn.microsoft.com/en-us/power-bi/transform-model/desktop-date-tables)
- [Power Query data types](https://learn.microsoft.com/en-us/power-query/data-types)
- [Slicers](https://learn.microsoft.com/en-us/power-bi/visuals/power-bi-visualization-slicers)
