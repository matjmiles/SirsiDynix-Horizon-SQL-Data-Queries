# 049 Tag Cleanup: Resolving Collection Mismatches

## The Problem
In library bibliographic records, the `049` (Local Holdings) tag should represent the unique collection codes assigned to the physical items. A data integrity issue occurs when a Bib record has items in **multiple distinct collections** (e.g., `GEN` and `REF`), but the `049` tags are **duplicates** of only one of those collections.

### The "Cartesian Product" Challenge
In most ILS schemas (like Horizon), `item` records and `bib` tags are both children of the `bib#` but have no direct link to one another. Joining them creates a **Many-to-Many** relationship. If a record has 2 items and 2 tags, a standard join produces 4 rows ($2 \times 2$), making it difficult to isolate which tag needs to change.



---

## The Solution
This solution identifies records where the count of `049` tags matches the count of unique collections, but the actual values are redundant. It uses `MAX(tagord)` to surgically target the second tag for update while preserving the first.

### Step 1: The Audit (Verification)
Run this query first to identify the records that will be modified. It shows the current "incorrect" text alongside the "new" text including the necessary MARC control characters (`CHAR(31)` for the subfield delimiter and `CHAR(30)` for the field terminator).

```sql
SELECT 
    b.bib# AS [Bib Number],
    b.tagord AS [Target TagOrd],
    b.text AS [Current Raw Text],
    -- Human readable version of the fix:
    '|a' + missing_code.collection AS [Readable New Value],
    -- The actual string including MARC control characters:
    CHAR(31) + 'a' + missing_code.collection + CHAR(30) AS [New Raw Text]
FROM bib b
INNER JOIN (
    -- Isolate Bibs with exactly 2 identical 049 tags
    SELECT bib#, MAX(tagord) as duplicate_tagord
    FROM bib
    WHERE tag = '049'
    GROUP BY bib#
    HAVING COUNT(*) = 2 AND MIN(text) = MAX(text)
) as duplicates 
    ON b.bib# = duplicates.bib# 
    AND b.tagord = duplicates.duplicate_tagord
INNER JOIN (
    -- Find the collection code present on an item but missing from the 049 tags
    SELECT DISTINCT i.bib#, i.collection
    FROM item i
    WHERE NOT EXISTS (
        SELECT 1 FROM bib b2 
        WHERE b2.bib# = i.bib# AND b2.tag = '049' 
        AND b2.text LIKE '%' + i.collection + '%'
    )
) as missing_code 
    ON b.bib# = missing_code.bib#
WHERE b.tag = '049'
ORDER BY b.bib#;