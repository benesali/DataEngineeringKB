# Data Modeling

Data modeling is the practice of defining *how data is structured* — which tables exist, what columns they have, how they relate, and how they handle change over time.

## In this section

| Page | What you'll learn |
|---|---|
| [Star Schema](star-schema.md) | The standard analytical model — facts surrounded by dimensions |
| [Snowflake Schema](snowflake-schema.md) | A normalized variant of the star schema |
| [Slowly Changing Dimensions](scd.md) | How to track changes to dimension attributes over time |
| [Fact Table Types](fact-types.md) | Transaction, periodic snapshot, and accumulating snapshot facts |
| [Keys — Natural, Surrogate, Business](keys.md) | The different types of identifiers and when to use each |

## Why modeling matters

The same data can be modeled dozens of ways. The model you choose determines:
- How fast queries run
- How easy it is for analysts to understand the data
- How much rework is needed when business requirements change
- Whether you can join across different subject areas correctly
