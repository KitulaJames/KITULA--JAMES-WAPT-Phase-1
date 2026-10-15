# PortSwigger Lab: SQL Injection Vulnerability in WHERE Clause Allowing Retrieval of Hidden Data

**Platform:** PortSwigger Web Security Academy (Apprentice level)
**Lab:** https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data
**Date completed:** 15 October 2026

## 1. Lab Name
SQL injection vulnerability in WHERE clause allowing retrieval of hidden data

## 2. Objective
Perform a SQL injection attack on the product category filter to make the application display unreleased products that should not normally be shown.

## 3. Vulnerability / Concept
The application builds a SQL query by directly concatenating the user-supplied `category` parameter into the WHERE clause, with no sanitisation or parameterisation:

    SELECT * FROM products WHERE category = 'Gifts' AND released = 1

Because the category value comes straight from the URL and is inserted as a raw string, an attacker can break out of the intended string literal and rewrite the logic of the query itself.

## 4. Testing Approach
1. Visited the shop and noted the normal request: `GET /filter?category=Gifts`, which only returns released products in that category.
2. Confirmed the parameter was reflected directly into a query by testing a broken payload, which produced errors/unexpected behaviour.
3. Built a payload that closes the string early and neutralises the rest of the WHERE clause:
   `category=Gifts' OR 1=1-- `
   - The closing `'` ends the original string literal.
   - `OR 1=1` makes the WHERE condition always true for every row.
   - `--` (with a trailing space) comments out the rest of the original query, including `AND released = 1`.
4. Sent the modified request and confirmed the response returned every product in the table, including unreleased ones.
5. The lab's status indicator flipped from "Not solved" to "Solved".

## 5. Evidence
Screenshots in the `ScreenShots` folder: normal request, the broken/error attempt, the final working payload and response, and the Solved confirmation.

## 6. Impact
Rating: **High**

An attacker can read data from the database that the application was specifically designed to hide (here, unreleased products). In a real application, the same technique against a different table or WHERE clause could expose other users' private data, bypass authentication logic, or be extended with UNION-based injection to read entirely unrelated tables (as seen in the Burp Suite Repeater room's `/about/ID` exercise).

## 7. Remediation
- Use parameterised queries / prepared statements so user input is always treated as data, never as part of the SQL syntax.
- Apply least-privilege database accounts so even a successful injection has limited reach.
- Validate and whitelist expected input (e.g. category must match a known list of values).
- Never build SQL queries via string concatenation of user input.

## 8. What I Learned
This lab reinforced the same root cause I found manually in the Burp Suite Repeater room: directly concatenating user input into a SQL query lets an attacker change the query's logic, not just its data. The `' OR 1=1--` pattern is the simplest form of this attack — it does not need to know the table or column structure, unlike the UNION-based extraction I performed earlier, which makes it a good first test for any input that touches a database.
