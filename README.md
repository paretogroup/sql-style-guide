
# Pareto's SQL Style Guide

Inspired by [Mazur's SQL Style Guide](https://github.com/mattm/sql-style-guide).

## Example

Here's a non-trivial query to give you an idea of what this style guide looks like in the practice:

```sql

WITH hubspot_interest AS (
    SELECT
        email,
        TIMESTAMP_MILLIS(property_beacon_interest) AS expressed_interest_at
    FROM hubspot.contact
    WHERE property_beacon_interest IS NOT NULL
), 

support_interest AS (
    SELECT 
        conversation.email,
        conversation.created_at AS expressed_interest_at
    FROM helpscout.conversation
    INNER JOIN helpscout.conversation_tag ON conversation.id = conversation_tag.conversation_id
    WHERE conversation_tag.tag = 'beacon-interest'
), 

combined_interest AS (
    SELECT * FROM hubspot_interest
    UNION ALL
    SELECT * FROM support_interest
),

final AS (
    SELECT 
        email,
        MIN(expressed_interest_at) AS expressed_interest_at
    FROM combined_interest
    GROUP BY email
)

SELECT * FROM final
```
## Guidelines

### Use lowerCASE SQL

It's easy to read even when you don't have good syntax highlighting.

```sql
-- Good
SELECT * FROM users

-- Bad
select * from users

-- Bad
Select * From users
```

### Single line vs multiple line queries

The only time you should place all of your SQL on a single line is when you're selecting one thing and there's no additional complexity in the query:

```sql
-- Good
SELECT * FROM users

-- Good
SELECT id FROM users

-- Good
SELECT COUNT(*) FROM users
```

Once you start adding more columns or more complexity, the query becomes easier to read if it's spread out on multiple lines:

```sql
-- Good
SELECT
    id,
    email,
    created_at
FROM users

-- Good
SELECT *
FROM users
WHERE email = 'example@domain.com'

-- Good
SELECT
    user_id,
    COUNT(*) AS total_charges
FROM charges
GROUP BY user_id

-- Bad
SELECT id, email, created_at
FROM users

-- Bad
SELECT id,
    email
FROM users
```

### Left align SQL keywords

Some IDEs have the ability to automatically format SQL so that the spaces after the SQL keywords are vertically aligned. This is cumbersome to do by hand (and in my opinion harder to read anyway) so I recommend just left aligning all of the keywords:

```sql
-- Good
SELECT 
    id, 
    email
FROM users
WHERE email LIKE '%@gmail.com'

-- Bad
SELECT id, email
  FROM users
 WHERE email LIKE '%@gmail.com'
```

### Use single quotes

Some SQL dialects like BigQuery support using double quotes, but for most dialects double quotes will wind up referring to column names. For that reason, single quotes are preferable:

```sql
-- Good
SELECT *
FROM users
WHERE email = 'example@domain.com'

-- Bad
SELECT *
FROM users
WHERE email = "example@domain.com"
```

### Use `!=` over `<>`

Simply because `!=` reads like "not equal" which is closer to how we'd say it out loud.

```sql
-- Good
SELECT COUNT(*) AS paying_users_count
FROM users
WHERE plan_name != 'free'
```

### Commas should be at the the END of lines

```sql
-- Good
SELECT
    id,
    email
FROM users

-- Bad
SELECT
    id
    , email
FROM users
```

### Indenting WHERE conditions

When there's only one WHERE condition, leave it on the same line as `WHERE`:

```sql
SELECT email
FROM users
WHERE id = 1234
```

When there are multiple, indent each one one level deeper than the `WHERE`. Put logical operators at the BEGINNING of the previous condition:

```sql
SELECT 
    id, 
    email
FROM users
WHERE 
    created_at >= '2019-03-01' 
    AND vertical = 'work'
```

### Avoid spaces inside of parenthesis

```sql
-- Good
SELECT *
FROM users
WHERE id IN (1, 2)

-- Bad
SELECT *
FROM users
WHERE id IN ( 1, 2 )
```

### Break long lists of `IN` values into multiple indented lines

```sql
-- Good
SELECT *
FROM users
WHERE email IN (
    'user-1@example.com',
    'user-2@example.com',
    'user-3@example.com',
    'user-4@example.com'
)
```

### Table names should be a plural snake case of the noun

```sql
-- Good
SELECT * FROM users
SELECT * FROM visit_logs

-- Bad
SELECT * FROM user
SELECT * FROM visitLog
```

### Column names should be snake_case

```sql
-- Good
SELECT
    id,
    email,
    TIMESTAMP_TRUNC(created_at, month) AS signup_month
FROM users

-- Bad
SELECT
    id,
    email,
    TIMESTAMP_TRUNC(created_at, month) AS SignupMonth
FROM users
```

### Column name conventions

* Boolean fields should be prefixed with `is_`, `has_`, or `does_`. For example, `is_customer`, `has_unsubscribed`, etc.
* Date-only fields should be suffixed with `_date`. For example, `report_date`.
* Date+time fields should be suffixed with `_at`. For example, `created_at`, `posted_at`, etc.

### Column order conventions

Put the primary key first, followed by foreign keys, then by all other columns. If the table has any system columns (`created_at`, `updated_at`, `is_deleted`, etc.), put those last.

```sql
-- Good
SELECT
    id,
    name,
    created_at
FROM users

-- Bad
SELECT
    created_at,
    name,
    id,
FROM users
```

### Include `INNER` for INNER JOINs

Better to be explicit so that the JOIN type is crystal clear:

```sql
-- Good
SELECT
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
INNER JOIN charges ON users.id = charges.user_id

-- Bad
SELECT
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
JOIN charges ON users.id = charges.user_id
```

### For JOIN conditions, put the table that was referenced first immediately after the `on`

By doing it this way it makes it easier to determine if your JOIN is going to cause the results to fan out:

```sql
-- Good
SELECT
    ...
FROM users
LEFT JOIN charges ON users.id = charges.user_id
-- primary_key = foreign_key --> one-to-many --> fanout
  
SELECT
    ...
FROM charges
LEFT JOIN users ON charges.user_id = users.id
-- foreign_key = primary_key --> many-to-one --> no fanout

-- Bad
SELECT
    ...
FROM users
LEFT JOIN charges ON charges.user_id = users.id
```

### Single JOIN conditions should be on the same line as the JOIN

```sql
-- Good
select
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
INNER JOIN charges ON users.id = charges.user_id
GROUP BY email

-- Bad
SELECT
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
INNER JOIN charges
ON users.id = charges.user_id
GROUP BY email
```

When you have mutliple JOIN conditions, place each one on their own indented line:

```sql
-- Good
SELECT
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
INNER JOIN charges ON 
    users.id = charges.user_id 
    AND refunded = false
GROUP BY email
```

### Avoid aliasing table names most of the time

It can be tempting to abbreviate table names like `users` to `u` and `charges` to `c`, but it winds up making the SQL less readable:

```sql
-- Good
SELECT
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
INNER JOIN charges ON users.id = charges.user_id

-- Bad
SELECT
    u.email,
    SUM(c.amount) AS total_revenue
FROM users u
INNER JOIN charges c ON u.id = c.user_id
```

Most of the time you'll want to type out the full table name.

There are two exceptions:

If you you need to join to a table more than once in the same query and need to distinguish each version of it, aliases are necessary.

Also, if you're working with long or ambiguous table names, it can be useful to alias them (but still use meaningful names):

```sql
-- Good: Meaningful table aliases
SELECT
  companies.com_name,
  beacons.created_at
FROM stg_mysql_helpscout__helpscout_companies companies
INNER JOIN stg_mysql_helpscout__helpscout_beacons_v2 beacons ON companies.com_id = beacons.com_id

-- OK: No table aliases
SELECT
  stg_mysql_helpscout__helpscout_companies.com_name,
  stg_mysql_helpscout__helpscout_beacons_v2.created_at
FROM stg_mysql_helpscout__helpscout_companies
INNER JOIN stg_mysql_helpscout__helpscout_beacons_v2 ON stg_mysql_helpscout__helpscout_companies.com_id = stg_mysql_helpscout__helpscout_beacons_v2.com_id

-- Bad: Unclear table aliases
SELECT
  c.com_name,
  b.created_at
FROM stg_mysql_helpscout__helpscout_companies c
INNER JOIN stg_mysql_helpscout__helpscout_beacons_v2 b ON c.com_id = b.com_id
```

### Include the table WHEN there is a JOIN, but omit it otherwise

When there are no JOIN involved, there's no ambiguity around which table the columns came FROM so you can leave the table name out:

```sql
-- Good
SELECT
    id,
    name
FROM companies

-- Bad
SELECT
    companies.id,
    companies.name
FROM companies
```

But WHEN there are JOINs involved, it's better to be explicit so it's clear WHERE the columns originated:

```sql
-- Good
SELECT
    users.email,
    SUM(charges.amount) AS total_revenue
FROM users
INNER JOIN charges ON users.id = charges.user_id

-- Bad
SELECT
    email,
    SUM(amount) AS total_revenue
FROM users
INNER JOIN charges ON users.id = charges.user_id

```

### Always rename aggregates and function-wrapped arguments

```sql
-- Good
SELECT COUNT(*) AS total_users
FROM users

-- Bad
select COUNT(*)
FROM users

-- Good
SELECT TIMESTAMP_MILLIS(property_beacon_interest) AS expressed_interest_at
FROM hubspot.contact
WHERE property_beacon_interest IS NOT NULL

-- Bad
SELECT TIMESTAMP_MILLIS(property_beacon_interest)
FROM hubspot.contact
WHERE property_beacon_interest IS NOT NULL
```

### Be explicit in boolean conditions

```sql
-- Good
SELECT * FROM customers WHERE is_cancelled = true
SELECT * FROM customers WHERE is_cancelled = false

-- Bad
SELECT * FROM customers WHERE is_cancelled
SELECT * FROM customers WHERE NOT is_cancelled
```

### Use `as` to alias column names

```sql
-- Good
SELECT
    id,
    email,
    TIMESTAMP_TRUNC(created_at, month) AS signup_month
FROM users

-- Bad
SELECT
    id,
    email,
    TIMESTAMP_TRUNC(created_at, month) signup_month
FROM users
```

### Group using column names or numbers, but not both

I prefer grouping by name, but grouping by numbers is [also fine](https://blog.getdbt.com/write-better-sql-a-defense-of-group-by-1/).

```sql
-- Good
SELECT user_id, COUNT(*) AS total_charges
FROM charges
GROUP BY user_id

-- Good
SELECT user_id, COUNT(*) AS total_charges
FROM charges
GROUP BY 1

-- Bad
SELECT
    TIMESTAMP_TRUNC(created_at, month) AS signup_month,
    vertical,
    COUNT(*) AS users_count
FROM users
GROUP BY 1, vertical
```

### Take advantage of lateral column aliasing when grouping by name

```sql
-- Good
SELECT
  TIMESTAMP_TRUNC(com_created_at, year) AS signup_year,
  COUNT(*) AS total_companies
FROM companies
GROUP BY signup_year

-- Bad
SELECT
  TIMESTAMP_TRUNC(com_created_at, year) AS signup_year,
  COUNT(*) AS total_companies
FROM companies
GROUP BY TIMESTAMP_TRUNC(com_created_at, year)
```

### Grouping columns should go first

```sql
-- Good
SELECT
  TIMESTAMP_TRUNC(com_created_at, year) AS signup_year,
  COUNT(*) AS total_companies
FROM companies
GROUP BY signup_year

-- Bad
SELECT
  COUNT(*) AS total_companies,
  TIMESTAMP_TRUNC(com_created_at, year) AS signup_year
FROM mysql_helpscout.helpscout_companies
GROUP BY signup_year
```

### Aligning CASE/WHEN statements

Each `WHEN` should be ON its own line (nothing on the `CASE` line) AND should be indented one level deeper than the `CASE` line. The `THEN` can be on the same line or on its own line below it, just aim to be consistent.

```sql
-- Good
SELECT
    CASE
        WHEN event_name = 'viewed_homepage' THEN 'Homepage'
        WHEN event_name = 'viewed_editor' THEN 'Editor'
        ELSE 'Other'
    END AS page_name
FROM events

-- Good too
SELECT
    CASE
        WHEN event_name = 'viewed_homepage'
            THEN 'Homepage'
        WHEN event_name = 'viewed_editor'
            THEN 'Editor'
        ELSE 'Other'            
    END AS page_name
FROM events

-- Bad 
SELECT
    CASE WHEN event_name = 'viewed_homepage' THEN 'Homepage'
        WHEN event_name = 'viewed_editor' THEN 'Editor'
        ELSE 'Other'        
    END AS page_name
FROM events
```

### Use CTEs, not subqueries

Avoid subqueries; CTEs will make your queries easier to read AND reason about.

When using CTEs, pad the query with new lines. 

If you use any CTEs, always have a CTE named `final` AND `SELECT * FROM final` at the END. That way you can quickly inspect the output of other CTEs used in the query to debug the results.

Closing CTE parentheses should use the same indentation level AS `WITH` and the CTE names.

```sql
-- Good
WITH ordered_details AS (
    SELECT
        user_id,
        name,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date_updated DESC) AS details_rank
    FROM billingdaddy.billing_stored_details
),

final AS (
    SELECT user_id, name
    FROM ordered_details
    WHERE details_rank = 1
)

SELECT * FROM final

-- Bad
SELECT user_id, name
FROM (
    SELECT
        user_id,
        name,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date_updated DESC) AS details_rank
    FROM billingdaddy.billing_stored_details
) ranked
WHERE details_rank = 1
```

### Use meaningful CTE names

```sql
-- Good
WITH ordered_details AS (

-- Bad
WITH d1 AS (
```

### Window functions

You can leave it all on its own line or break it up into multiple depending on its length:

```sql
-- Good
SELECT
    user_id,
    name,
    ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date_updated DESC) AS details_rank
FROM billingdaddy.billing_stored_details

-- Good
SELECT
    user_id,
    name,
    ROW_NUMBER() OVER (
        PARTITION BY user_id
        ORDER BY date_updated DESC
    ) AS details_rank
FROM billingdaddy.billing_stored_details
```

## Credits

This style guide was inspired in part by:

* [Fishtown Analytics' dbt Style Guide](https://github.com/fishtown-analytics/corp/blob/master/dbt_coding_conventions.md#sql-style-guide)
* [KickStarter's SQL Style Guide](https://gist.github.com/fredbenenson/7bb92718e19138c20591)
* [GitLab's SQL Style Guide](https://about.gitlab.com/handbook/business-ops/data-team/sql-style-guide/)

Hat-tip to Peter Butler, Dan Wyman, Simon Ouderkirk, Alex Cano, Adam Stone, Brian Kim, and Claire Carroll for providing feedback on this guide.
