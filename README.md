/*
=========================================================
PBI 595486 - Plan of Work Table Removal
STEP 1 of 3: RENAME the PoW tables (reversible)

What it does
  Renames every PlanOfWork% table, plus SourceOfFundingAllocation, by
  adding the prefix ZZ_DROP_ (example: PlanOfWorkX -> ZZ_DROP_PlanOfWorkX).
  Nothing is dropped and no data is touched. If the app still needs a
  table, it breaks loudly and 02_Undo_Rename puts the name back.

How to run (do this every time, in this order)
  1. Connect to the right server and confirm the database in SSMS.
  2. Set @ExpectedServer below to that server name (recommended).
  3. Run once with @Execute = 0. This is a DRY RUN: it only reports.
  4. Read result sets 1-3. If result set 1 says STOP, fix it first.
  5. Set @Execute = 1 and run again.
  6. Save all result sets (they are the record for documentation).

Safety built in
  - Stops if the table count is not @ExpectedCount (wrong DB, or
    tables already renamed).
  - Stops if the target ZZ_DROP_ name already exists.
  - Stops if a schema-bound view/function would block the rename.
  - All renames run in ONE transaction: any error rolls back all of them.
  - Does not rename constraint or index names. That is intentional;
    they disappear with the table at drop time.

Result sets
  1. STOP list (only appears if something is wrong)
  2. Code that mentions these tables (expected to break once renamed
     if it is truly dead code; use this list to ask devs)
  3. The rename plan with row counts (and result when executed)
  4. Verification (only when @Execute = 1)
=========================================================
*/
SET NOCOUNT ON;
SET XACT_ABORT ON;

-------------------------------------------------------------
-- SETTINGS
-------------------------------------------------------------
DECLARE @Execute        BIT     = 0;      -- 0 = dry run, 1 = do it
DECLARE @ExpectedCount  INT     = 11;     -- number of tables in scope
DECLARE @ExpectedServer SYSNAME = N'';    -- e.g. N'YOUR-DEV-SERVER'; blank = skip check
DECLARE @Prefix         SYSNAME = N'ZZ_DROP_';
DECLARE @Pattern        SYSNAME = N'PlanOfWork%';

-------------------------------------------------------------
-- SCOPE: PlanOfWork% plus the extra table(s) below
-------------------------------------------------------------
IF OBJECT_ID('tempdb..#Extra')    IS NOT NULL DROP TABLE #Extra;
IF OBJECT_ID('tempdb..#Scope')    IS NOT NULL DROP TABLE #Scope;
IF OBJECT_ID('tempdb..#Problems') IS NOT NULL DROP TABLE #Problems;
IF OBJECT_ID('tempdb..#Plan')     IS NOT NULL DROP TABLE #Plan;

CREATE TABLE #Extra (TableName SYSNAME);
INSERT #Extra (TableName) VALUES (N'SourceOfFundingAllocation');

CREATE TABLE #Problems (Problem NVARCHAR(500));

SELECT t.object_id,
       SCHEMA_NAME(t.schema_id) AS SchemaName,
       t.name                   AS TableName
INTO #Scope
FROM sys.tables t
WHERE t.name LIKE @Pattern
   OR t.name IN (SELECT TableName FROM #Extra);

PRINT 'Server: ' + @@SERVERNAME + '   Database: ' + DB_NAME()
    + '   Mode: ' + CASE WHEN @Execute = 1 THEN 'EXECUTE' ELSE 'DRY RUN' END;

-------------------------------------------------------------
-- SAFETY CHECKS
-------------------------------------------------------------
IF @ExpectedServer <> N'' AND @@SERVERNAME <> @ExpectedServer
    INSERT #Problems VALUES (N'Wrong server: connected to ' + @@SERVERNAME
                           + N', expected ' + @ExpectedServer);

DECLARE @Count INT = (SELECT COUNT(*) FROM #Scope);
IF @Count <> @ExpectedCount
    INSERT #Problems VALUES (N'Scope has ' + CAST(@Count AS NVARCHAR(10))
        + N' tables, expected ' + CAST(@ExpectedCount AS NVARCHAR(10))
        + N'. Wrong database, or some tables already renamed?');

INSERT #Problems
SELECT N'Target name already exists: ' + s.SchemaName + N'.' + @Prefix + s.TableName
FROM #Scope s
WHERE OBJECT_ID(QUOTENAME(s.SchemaName) + N'.' + QUOTENAME(@Prefix + s.TableName)) IS NOT NULL;

INSERT #Problems
SELECT DISTINCT N'Schema-bound object blocks rename: '
     + OBJECT_SCHEMA_NAME(d.referencing_id) + N'.' + OBJECT_NAME(d.referencing_id)
FROM sys.sql_expression_dependencies d
JOIN #Scope s ON s.object_id = d.referenced_id
WHERE d.is_schema_bound_reference = 1;

IF EXISTS (SELECT 1 FROM #Problems)
BEGIN
    SELECT N'STOP - fix these before running' AS Status, Problem FROM #Problems;
    RETURN;
END

-------------------------------------------------------------
-- CODE THAT MENTIONS THESE TABLES (for the dev conversation)
-------------------------------------------------------------
SELECT DISTINCT
       OBJECT_SCHEMA_NAME(m.object_id) + N'.' + OBJECT_NAME(m.object_id) AS ReferencingObject,
       o.type_desc                                                       AS ObjectType,
       s.TableName                                                       AS MentionsTable
FROM sys.sql_modules m
JOIN sys.objects o ON o.object_id = m.object_id
JOIN #Scope s      ON m.definition LIKE N'%' + s.TableName + N'%'
WHERE o.is_ms_shipped = 0
ORDER BY ReferencingObject, MentionsTable;

-------------------------------------------------------------
-- THE PLAN
-------------------------------------------------------------
CREATE TABLE #Plan (
    StepId     INT IDENTITY(1,1) PRIMARY KEY,
    SchemaName SYSNAME,
    OldName    SYSNAME,
    NewName    SYSNAME,
    RowsBefore BIGINT NULL,
    Result     NVARCHAR(500) NULL
);

INSERT #Plan (SchemaName, OldName, NewName, RowsBefore)
SELECT s.SchemaName,
       s.TableName,
       @Prefix + s.TableName,
       (SELECT SUM(ps.row_count)
          FROM sys.dm_db_partition_stats ps
         WHERE ps.object_id = s.object_id AND ps.index_id IN (0, 1))
FROM #Scope s
ORDER BY s.TableName;

-------------------------------------------------------------
-- EXECUTE (single transaction)
-------------------------------------------------------------
IF @Execute = 1
BEGIN
    DECLARE @i INT = 1;
    DECLARE @max INT = (SELECT MAX(StepId) FROM #Plan);
    DECLARE @old NVARCHAR(300);
    DECLARE @new SYSNAME;

    BEGIN TRY
        BEGIN TRAN;
        WHILE @i <= @max
        BEGIN
            SELECT @old = QUOTENAME(SchemaName) + N'.' + QUOTENAME(OldName),
                   @new = NewName
            FROM #Plan WHERE StepId = @i;

            EXEC sp_rename @objname = @old, @newname = @new, @objtype = 'OBJECT';

            UPDATE #Plan SET Result = N'Renamed' WHERE StepId = @i;
            SET @i += 1;
        END
        COMMIT;
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0 ROLLBACK;
        UPDATE #Plan SET Result = N'ROLLED BACK - ' + ERROR_MESSAGE();
        PRINT 'ERROR - everything was rolled back: ' + ERROR_MESSAGE();
    END CATCH
END

SELECT StepId, SchemaName, OldName, NewName, RowsBefore,
       CASE WHEN @Execute = 1 THEN Result ELSE N'DRY RUN - not executed' END AS Result
FROM #Plan
ORDER BY StepId;

IF @Execute = 1
    SELECT p.StepId, p.OldName, p.NewName,
           CASE WHEN OBJECT_ID(QUOTENAME(p.SchemaName) + N'.' + QUOTENAME(p.NewName)) IS NOT NULL
                 AND OBJECT_ID(QUOTENAME(p.SchemaName) + N'.' + QUOTENAME(p.OldName)) IS NULL
                THEN 'VERIFIED' ELSE 'CHECK' END AS Verification
    FROM #Plan p
    ORDER BY p.StepId;

DROP TABLE #Plan;
DROP TABLE #Problems;
DROP TABLE #Scope;
DROP TABLE #Extra;
