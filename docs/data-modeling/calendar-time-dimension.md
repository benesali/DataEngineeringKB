# Calendar & Time Dimension Modeling

> *Primary source: Claudia Imhoff, Nicholas Galemmo, Jonathan Geiger — "Mastering Data Warehouse Design: Relational and Dimensional Techniques" (Wiley, 2003), Chapter 6*

## Why the calendar is the most important dimension

The calendar is the one dimension that every other dimension links to. It is the central axis of historical analysis — the data warehouse's entire value proposition is "data over time," and the calendar is what makes "over time" queryable.

A common mistake is to rely on database date functions (`MONTH()`, `WEEK()`, `DATEPART()`) to interpret dates at query time. This fails for three reasons:
- Different users and reports apply different definitions of "fiscal month" or "business day"
- Cross-platform or international deployments have different calendar conventions
- Performance degrades when calendar logic is computed per-row in queries rather than pre-joined

A dedicated, pre-populated calendar dimension encodes these rules once and answers all queries without recomputation.

## Calendar types used in business

| Calendar type | Purpose | Example |
|---|---|---|
| **Gregorian** | Standard calendar — the foundation for all others | January 1 – December 31 |
| **Fiscal** | Financial reporting and accounting; doesn't align with Gregorian | July 1 – June 30 fiscal year for a retailer |
| **4-5-4** | Retail fiscal: 13 weeks per quarter, each quarter = 4+5+4 weeks | Ends on a Friday; facilitates week-to-week comparisons |
| **Billing cycle** | Events tied to billing date (e.g., utility meter read, payment due) | 30 days after billing date = interest start |
| **Factory / workday** | Manufacturing scheduling; counts only production days | Day 183 of factory calendar = April 18 |
| **Seasonal** | Business-defined time periods of significance | "Holiday Season" = day after Thanksgiving through Dec 24 |

## The 4-5-4 fiscal calendar

Retail companies use the 4-5-4 calendar to make week-to-week comparisons possible when rolling up to months. Each quarter has 13 weeks distributed 4+5+4 across 3 months. This ensures every quarter has the same number of days and weeks, making month-over-month and year-over-year comparisons meaningful.

**Why not just use calendar months?** In Gregorian months, February has 28 days and March has 31 — a 10% difference. Comparing "sales this February" to "sales last March" is noise, not signal. The 4-5-4 calendar eliminates this by keeping all periods week-aligned.

**The price:** every third month is 25% longer (5 weeks vs 4 weeks), so month-to-month comparisons within the 4-5-4 calendar must account for this. Quarter-to-quarter comparisons are clean.

## The denormalized calendar table

The calendar entity is unique in that its small size makes full denormalization worthwhile. The normalized business model (Year → Month → Day, FiscalYear → FiscalQuarter → FiscalMonth → FiscalWeek → Date) is kept for updates, but a single denormalized `Calendar` table is generated from it and used as the source for all data mart delivery.

**Why both?** Updates to the normalized model are simple (e.g., changing one day from workday to non-workday touches one row). Re-deriving the denormalized table from scratch after any change keeps derived columns consistent. Delivering the denormalized table to data marts is then a direct copy, no calculation needed at delivery time.

### Key attributes of the denormalized calendar table

| Column | Purpose |
|---|---|
| `Date` | Primary key — one row per calendar day |
| `DayName`, `DayShortName` | Day of week |
| `WorkdayIndicator` | True if this day is a workday (considers day-of-week + holidays) |
| `MonthName`, `MonthShortName` | Gregorian month |
| `YearNumber` | Gregorian year |
| `FiscalWeekStartDate`, `FiscalWeekEndDate` | Boundaries of the fiscal week this date belongs to |
| `FiscalMonthName`, `FiscalMonthSequenceNumber` | Fiscal month |
| `FiscalQuarterName`, `FiscalQuarterSequenceNumber` | Fiscal quarter |
| `FiscalYearStartDate`, `FiscalYearEndDate` | Fiscal year boundaries |
| `WorkdayOfWeek` | Ordinal position of this workday in its week (1=first workday) |
| `WorkdayOfMonth` | Ordinal position of this workday in its Gregorian month |
| `WorkdayOfFiscalMonth` | Ordinal position of this workday in its fiscal month |
| `WorkdayOfYear` | Running count of workdays in the Gregorian year |
| `WorkdayOfFiscalYear` | Running count of workdays in the fiscal year |
| `WorkdayCount` | Absolute running count of workdays since the start of the calendar |
| `WorkdaysInMonth` | Total workdays in this Gregorian month |
| `WorkdaysInFiscalMonth` | Total workdays in this fiscal month |
| `WorkdaysInFiscalYear` | Total workdays in this fiscal year |
| `LastDayOfMonthIndicator` | True if this is the last day of the Gregorian month |
| `LastDayOfFiscalMonthIndicator` | True if this is the last day of the fiscal month |
| `LastDayOfFiscalYearIndicator` | True if this is the last day of the fiscal year |
| `SameDayLastFiscalMonthDate` (FK) | Self-referential FK to the equivalent day in the previous fiscal month |
| `SameDayLastFiscalYearDate` (FK) | Self-referential FK to the equivalent day in the previous fiscal year |

### Why `WorkdayOfFiscalMonth` matters

Month-to-date comparisons require aligning on workday, not calendar day. If today is the 15th workday of this fiscal month, the correct year-ago comparison is data through the 15th workday of the same fiscal month last year — not through the same calendar date. This column enables that filter without any calculation in the query.

## Date vs. time — keep them separate

A critical design rule: **never store date and time as the same attribute** in a data model.

A `Date` attribute serves as a foreign key to the calendar dimension. A combined `DateTime` attribute cannot serve this role — it represents a point in time, not a date, and every sale at a different time on the same day would have a different FK value.

Separate them explicitly:
- `SaleDate` (FK to Calendar) — links to the date dimension
- `SaleTime` (integer: minutes since midnight, or HH:MM text) — stored separately for intra-day analysis

**UTC vs local time:** store both when needed. A retail chain cares about local time (customer's perspective); a telecom carrier cares about both local (customer patterns) and UTC (network traffic analysis). Because timezone offsets change (DST, political decisions, ~37 active zones), calculating one from the other retrospectively is unreliable. Capture both at the source.

## Date key choice

The date is one of the few natural keys that is:
- Unique by definition (no two days have the same date)
- Persistent (dates don't change)
- Universally understood
- Not subject to reuse or reassignment

A surrogate key for the calendar is optional. The natural date (in native database format) makes a perfectly good primary key. The tradeoff: if you use natural date keys, you must handle incoming dates that are in non-standard formats. Add alternate key columns (e.g., `DateCYMD VARCHAR(8)` in YYYYMMDD format, `DateJulian INTEGER`) to support direct lookup from various source feed formats without format conversion.

## Alternate date formats as alternate keys

Source systems deliver dates in different formats: CCYYMMDD character strings, Julian integers, YYMMDD numbers. Each format variation should have its own column in the Date table so that the ETL process can look up the date row directly using the source format, without needing to convert first. An invalid source date (e.g., "20260231") fails cleanly at lookup time rather than crashing with a datetime conversion error.

## Location-specific calendars

When stores have different operating schedules (hours, closed days), or when comparing store performance requires knowing how many hours each store was open, the standard corporate calendar must be extended with a location-specific layer.

Design:
```
Calendar (one row per date — corporate calendar)
    ↕ join on Date
Location_Calendar (one row per location per date — location-specific)
    Location_ID
    Date (FK → Calendar)
    OpenTime, CloseTime
    StoreClosed_Indicator
    ...all calendar attributes repeated or inherited
```

The Location_Calendar is generated from Schedule → Location_Schedule (which schedule this location follows, bounded by effective/expiration dates). When a store changes schedules, a new Location_Schedule row is inserted; the Location_Calendar is regenerated.

**Important:** the Location_Calendar needs a surrogate primary key if it will be delivered to a data mart as a dimension. Natural keys (Location + Date compound) translate poorly to star schemas where a single-column FK in the fact table is expected.

## Multilingual calendars

Adding a language code to the key of each text table enables unlimited language support without schema changes:

```
DayText (DayID, LanguageCode, DayName, DayShortName)
MonthText (MonthID, LanguageCode, MonthName, MonthShortName)
FiscalMonthText (FiscalMonthID, LanguageCode, FiscalMonthName, ...)
```

When a user logs in, their language preference selects which rows are returned. This approach handles unlimited languages; adding column-per-language (`DayName_EN`, `DayName_FR`) is a dead end that requires schema changes for every new language.

## Seasonal calendars

Seasons are user-defined time periods of significance. The "holiday season" (day after Thanksgiving through December 24) varies by year but is semantically consistent. Pre-storing a `HolidaySeasonIndicator` in the Date table lets analysts filter or compare across years without looking up specific dates each time.

The full pattern: any period of significance gets a flag or a FK in the Date table. This gives analysts control over analysis context without requiring them to know the calendar math.

## Modern implementation note

In a Fabric/Databricks lakehouse, the same pattern applies:
- Store the calendar as a Gold dimension table (e.g., `dim_date`)
- Pre-populate 5–10 years back + 2–3 years forward (planning horizon)
- Include all derived columns at build time
- Update annually for new holiday schedules and fiscal year definitions
- Join all fact tables to `dim_date` on the date key — avoid `MONTH()` and `YEAR()` in query logic

```sql
-- Pattern: filter by fiscal period without date arithmetic
SELECT d.FiscalYearName, SUM(f.SalesAmount)
FROM fact_sales f
JOIN dim_date d ON f.SaleDateKey = d.DateKey
WHERE d.FiscalYearName = '2026'
  AND d.WorkdayOfFiscalMonth <= 15  -- month-to-date through 15th workday
GROUP BY d.FiscalYearName
```
