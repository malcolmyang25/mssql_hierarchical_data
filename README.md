# Hierarchical Data (SQL Server)
---
[Click to view code](https://github.com/malcolmyang25/mssql_hierarchy_data/blob/main/Hierarchy%20Structure%20Data%20Explore.sql)\
Hierarchical data is defined as a set of data items that are related to each other by hierarchical relationships. Hierarchical relationships exist where one item of data is the parent of another item.

In this documents, I list four ways to query hierarchical data.

To illustrate this hierarchical data process methods, the sample is about sum up the invoice of the two marketing team. George is the head of marketing, Lee and Mary are two directors, each of them leads a sales team.

The source table lists all invoice for both team. Each invoice needs to sign by higher level manager, i.e. if manager pay this invoice, then senior manager need to sign before the manager pay. Only the head can sign the invoice by themselft.

All invoices are listed by transaction. The business requirement is **listing the total invoice amount by the head**.

---
Source Table: 
| trans\_id | pay\_person | sign\_person | invoice\_amount |
| :--- | :--- | :--- | :--- |
| 1 | Anna | Nick | 200 |
| 2 | Paul | Jason | 150 |
| 3 | Ryan | Jason | 130 |
| 4 | Nathan | Jack | 200 |
| 5 | Julie | Jack | 120 |
| 6 | Naomi | Emma | 130 |
| 7 | Nick | Mary | 500 |
| 8 | Jason | Mary | 620 |
| 9 | Jack | Lee | 720 |
| 10 | Emma | Lee | 660 |
| 11 | Mary | George | 1150 |
| 12 | Lee | George | 1320 |
| 13 | George | George | 3000 |

---
Destination Table:
| team | Team Invoice |
| :--- | :--- |
| Lee  | 3150 |
| Mary | 2750 |
|      | 5900 |


----
Solution 1: sub-query+join
---
### Comment: This method only available on known structure scenario.
```
SELECT a.pay_person,b.pay_person,c.pay_person,a.invoice_amount,
       CASE WHEN (a.pay_person='Mary' OR b.pay_person='Mary' OR c.pay_person='Mary') THEN 'Mary'
            WHEN (a.pay_person='Lee' OR b.pay_person='Lee' OR c.pay_person='Lee') THEN 'Lee'
       ELSE 'None' END AS [Team]
FROM invoice a
LEFT JOIN  invoice b
ON a.sign_person=b.pay_person
LEFT JOIN invoice c
ON b.sign_person=c.pay_person;
```

---
Solution 2: T-SQL Loop
---
### comment: this method is to pick all the member in each team by loop, then sum up all invoice under those member.

```
DECLARE @team table(
  member_name nvarchar(10),
  team_name nvarchar(10)
                  );

INSERT @team VALUES ('Mary','Mary');
INSERT @team VALUES ('Lee','Lee');

DECLARE @new_team table(
  member_name nvarchar(10),
  team_name nvarchar(10)
                  );

DECLARE @teamnum int;
DECLARE @new_teamnum int;

SET @teamnum= (SELECT COUNT(DISTINCT member_name) FROM @team);
SET @new_teamnum= (SELECT COUNT(DISTINCT member_name) FROM @new_team);

WHILE @teamnum>@new_teamnum
    BEGIN
       WITH t AS
        (SELECT pay_person AS [member_name],'Mary' AS [team_name] FROM invoice
         WHERE sign_person IN (SELECT member_name FROM @team WHERE team_name='Mary')
        UNION ALL
        SELECT pay_person AS [member_name],'Lee' AS [team_name] FROM invoice
         WHERE sign_person IN (SELECT member_name FROM @team WHERE team_name='Lee')
        UNION ALL
        SELECT pay_person AS [member_name],'Mary' AS [team_name] FROM invoice
         WHERE pay_person IN (SELECT member_name FROM @team WHERE team_name='Mary')
        UNION ALL
        SELECT pay_person AS [member_name],'Lee' AS [team_name] FROM invoice
         WHERE pay_person IN (SELECT member_name FROM @team WHERE team_name='Lee'))
         INSERT INTO @new_team (member_name, team_name)
        SELECT DISTINCT member_name,team_name FROM t;
       WITH t AS
        (SELECT pay_person AS [member_name],'Mary' AS [team_name] FROM invoice
         WHERE sign_person IN (SELECT member_name FROM @new_team WHERE team_name='Mary')
        UNION ALL
        SELECT pay_person AS [member_name],'Lee' AS [team_name] FROM invoice
         WHERE sign_person IN (SELECT member_name FROM @new_team WHERE team_name='Lee')
        UNION ALL
        SELECT pay_person AS [member_name],'Mary' AS [team_name] FROM invoice
         WHERE pay_person IN (SELECT member_name FROM @new_team WHERE team_name='Mary')
        UNION ALL
        SELECT pay_person AS [member_name],'Lee' AS [team_name] FROM invoice
         WHERE pay_person IN (SELECT member_name FROM @new_team WHERE team_name='Lee'))
         INSERT INTO @team (member_name, team_name)
        SELECT DISTINCT member_name,team_name FROM t;
        SET @teamnum= (SELECT COUNT(DISTINCT member_name) FROM @team);
        SET @new_teamnum= (SELECT COUNT(DISTINCT member_name) FROM @new_team);
    END;

WITH t AS
    (SELECT team_name AS [Team],invoice_amount FROM invoice INNER JOIN
        (SELECT DISTINCT * FROM @team) team
        ON pay_person=member_name)
SELECT t.Team,SUM(invoice_amount) AS [Team Invoice] FROM t
GROUP BY t.Team WITH ROLLUP;
```

---
Solution 3: CTE(common table expression) with Parent ID and Child ID
---
### comment: For the source table, the parent identifier need to be NULL in root node.
```
DECLARE @invoice_temp TABLE (
    trans_id int,
    pay_person varchar(20),
    sign_person varchar(20)
);

INSERT INTO @invoice_temp (trans_id, pay_person, sign_person)
SELECT trans_id,pay_person,
       CASE WHEN pay_person=sign_person THEN NULL
            ELSE sign_person END AS [sign_person]
FROM invoice;
SELECT * FROM @invoice_temp;

WITH cte AS (
    SELECT *, pay_person AS [final_person]
    FROM @invoice_temp
    UNION ALL
    SELECT a.*,b.final_person
    FROM @invoice_temp a,
         cte b
    WHERE a.sign_person=b.pay_person
) SELECT final_person AS team,SUM(invoice_amount) AS [Team Invoice] FROM cte
INNER JOIN invoice c
ON cte.pay_person=c.pay_person
WHERE final_person IN ('Mary','Lee')
GROUP BY final_person WITH ROLLUP;
```

---
Solution 4: HierarchyID
---
### Comment:The hierarchyid data type is a variable length, system data type.Use hierarchyid to represent position in a hierarchy. To calculation, the parent-child table need to generate the hierarychyid column.

```
DECLARE @invoice_ch TABLE (
    pay_person varchar(20),
    sign_person varchar(20),
    invoice_amount int,
    level int,
    node hierarchyid,
);

WITH Children AS (
        SELECT
          pay_person
         ,sign_person
         ,CAST(ROW_NUMBER()
               OVER (PARTITION BY sign_person ORDER BY pay_person) AS VARCHAR) + '/' AS Child
        FROM invoice
        WHERE pay_person<>invoice.sign_person
    ),
    NoChildren AS (
        -- Anchor member    --
         SELECT  0 AS [Level]
                ,P.pay_person
                ,P.sign_person
                , hierarchyid::GetRoot() AS Node
         FROM invoice AS P
         WHERE  pay_person=sign_person
        UNION ALL
        -- Recursive member --
         SELECT  NC.[Level] + 1 AS [Level]
                 ,C.pay_person
                 ,C.sign_person
                 , CAST(NC.Node.ToString() + C.Child AS hierarchyid)  AS Node
         FROM invoice AS P
         JOIN NoChildren NC ON P.sign_person = NC.pay_person
         JOIN Children C ON P.pay_person = C.pay_person
    )
    INSERT INTO @invoice_ch (pay_person, sign_person, invoice_amount, level, node)
    SELECT p.pay_person,p.sign_person,invoice_amount,Level,Node
    FROM invoice P
     JOIN NoChildren NC ON P.pay_person = NC.pay_person;

DECLARE @marynode hierarchyid;
DECLARE @leenode hierarchyid;
SET @marynode = (SELECT Node FROM @invoice_ch WHERE pay_person = 'mary');
SET @leenode = (SELECT Node FROM @invoice_ch WHERE pay_person = 'lee');

WITH tb AS (
    SELECT 'Mary' AS [Team],invoice_amount FROM @invoice_ch
    WHERE Node.IsDescendantOf(@marynode)=1
    UNION ALL
    SELECT 'Lee' AS [Team],invoice_amount FROM @invoice_ch
    WHERE Node.IsDescendantOf(@leenode)=1
) SELECT team,SUM(invoice_amount) AS [Team Invoice] FROM tb
    GROUP BY team WITH ROLLUP;
```
