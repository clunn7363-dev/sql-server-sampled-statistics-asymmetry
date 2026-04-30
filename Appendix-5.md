```sql
/*
Appendix 5 – Identifying tables vulnerable to distinct under-representation
Representative, bounded diagnostic (SQL Server 2012+ compatible)
*/

SET NOCOUNT ON;
GO

----------------------------------------------------------------------
-- Tunable parameters
----------------------------------------------------------------------

DECLARE @MinRowCount        bigint = 3000000;   -- minimum table size
DECLARE @MaxRowCount        bigint = 100000000;  -- maximum table size (for performance reasons)
DECLARE @MinDistinct1Pct   bigint = 1000;       -- high-cardinality relevance
DECLARE @MinDistinctGrowth float  = 1.25;       -- 25% growth threshold
DECLARE @MaxFindings       int    = 10;          -- max tables to report
DECLARE @StatPrefix        sysname = N'__diag_sample_';

----------------------------------------------------------------------
-- Temp tables
----------------------------------------------------------------------

IF OBJECT_ID('tempdb..#CandidateTables') IS NOT NULL DROP TABLE #CandidateTables;
IF OBJECT_ID('tempdb..#CandidateStats')  IS NOT NULL DROP TABLE #CandidateStats;
IF OBJECT_ID('tempdb..#Histogram')       IS NOT NULL DROP TABLE #Histogram;
IF OBJECT_ID('tempdb..#Results')         IS NOT NULL DROP TABLE #Results;

CREATE TABLE #CandidateTables
(
    object_id   int PRIMARY KEY,
    schema_name sysname,
    table_name  sysname,
    row_count   bigint
);

CREATE TABLE #CandidateStats
(
    object_id   int,
    schema_name sysname,
    table_name  sysname,
    stats_name  sysname,
    column_name sysname
);

CREATE TABLE #Histogram
(
    RANGE_HI_KEY sql_variant,
    RANGE_ROWS float,
    EQ_ROWS float,
    DISTINCT_RANGE_ROWS float,
    AVG_RANGE_ROWS float
);

CREATE TABLE #Results
(
    schema_name                 sysname,
    table_name                  sysname,
    column_name                 sysname,
    base_statistics_name        sysname,
    distinct_1pct               bigint,
    distinct_2pct               bigint,
    growth_factor               float,
    estimated_min_coverage_1pct float,
    estimated_min_inaccuracy    float
);

----------------------------------------------------------------------
-- Identify candidate tables (partition-safe, bounded range)
----------------------------------------------------------------------

INSERT #CandidateTables (object_id, schema_name, table_name, row_count)
SELECT
    t.object_id,
    sch.name,
    t.name,
    SUM(p.rows) AS row_count
FROM sys.tables t
JOIN sys.schemas sch
    ON sch.schema_id = t.schema_id
JOIN sys.partitions p
    ON p.object_id = t.object_id
   AND p.index_id IN (0,1)
GROUP BY
    t.object_id,
    sch.name,
    t.name
HAVING
    SUM(p.rows) >= @MinRowCount
    AND SUM(p.rows) <= @MaxRowCount
ORDER BY
    SUM(p.rows) DESC;

----------------------------------------------------------------------
-- Main loop: one representative column per table
----------------------------------------------------------------------

DECLARE
    @object_id int,
    @schema sysname,
    @table  sysname,
    @column sysname,
    @base_stat sysname,
    @stat1 sysname,
    @stat2 sysname,
    @sql nvarchar(max),
    @d1 bigint,
    @d2 bigint,
    @tables_checked int = 0;

DECLARE table_cursor CURSOR LOCAL FAST_FORWARD FOR
SELECT object_id, schema_name, table_name
FROM #CandidateTables
ORDER BY row_count DESC;

OPEN table_cursor;
FETCH NEXT FROM table_cursor INTO @object_id, @schema, @table;

WHILE @@FETCH_STATUS = 0
  AND (SELECT COUNT(*) FROM #Results) < @MaxFindings
BEGIN
    SET @tables_checked += 1;

    PRINT CONCAT(
        'Scanning table ',
        @schema, '.', @table,
        ' (table #', @tables_checked, ')'
    );

    ------------------------------------------------------------------
    -- Collect eligible statistics for this table
    ------------------------------------------------------------------

    DELETE FROM #CandidateStats;

    INSERT #CandidateStats
    SELECT
        s.object_id,
        @schema,
        @table,
        s.name,
        c.name
    FROM sys.stats s
    JOIN sys.stats_columns sc
        ON s.object_id = sc.object_id
       AND s.stats_id  = sc.stats_id
    JOIN sys.columns c
        ON c.object_id = sc.object_id
       AND c.column_id = sc.column_id
    LEFT JOIN sys.index_columns ic
        ON ic.object_id = c.object_id
       AND ic.column_id = c.column_id
    LEFT JOIN sys.indexes i
        ON i.object_id = ic.object_id
       AND i.index_id  = ic.index_id
    WHERE
        s.object_id = @object_id
        AND s.auto_created = 1
        AND (i.index_id IS NULL OR i.type <> 1); -- exclude clustered index columns

    DECLARE stat_cursor CURSOR LOCAL FAST_FORWARD FOR
    SELECT stats_name, column_name
    FROM #CandidateStats;

    OPEN stat_cursor;
    FETCH NEXT FROM stat_cursor INTO @base_stat, @column;

    WHILE @@FETCH_STATUS = 0
    BEGIN
        SET @stat1 = @StatPrefix + @base_stat + '_1pct';
        SET @stat2 = @StatPrefix + @base_stat + '_2pct';

        ------------------------------------------------------------------
        -- Create sampled statistics
        ------------------------------------------------------------------

        SET @sql = N'
            CREATE STATISTICS ' + QUOTENAME(@stat1) + '
            ON ' + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '
            (' + QUOTENAME(@column) + ') WITH SAMPLE 1 PERCENT;

            CREATE STATISTICS ' + QUOTENAME(@stat2) + '
            ON ' + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '
            (' + QUOTENAME(@column) + ') WITH SAMPLE 2 PERCENT;
        ';
        EXEC (@sql);

        ------------------------------------------------------------------
        -- 1% histogram
        ------------------------------------------------------------------

        TRUNCATE TABLE #Histogram;

        SET @sql = N'
            DBCC SHOW_STATISTICS (
                ''' + @schema + '.' + @table + ''',
                ' + QUOTENAME(@stat1) + '
            ) WITH HISTOGRAM;
        ';
        INSERT #Histogram EXEC (@sql);
        SELECT @d1 = SUM(DISTINCT_RANGE_ROWS) FROM #Histogram;

        ------------------------------------------------------------------
        -- 2% histogram
        ------------------------------------------------------------------

        TRUNCATE TABLE #Histogram;

        SET @sql = N'
            DBCC SHOW_STATISTICS (
                ''' + @schema + '.' + @table + ''',
                ' + QUOTENAME(@stat2) + '
            ) WITH HISTOGRAM;
        ';
        INSERT #Histogram EXEC (@sql);
        SELECT @d2 = SUM(DISTINCT_RANGE_ROWS) FROM #Histogram;

        ------------------------------------------------------------------
        -- Evaluate significance
        ------------------------------------------------------------------

        IF @d1 >= @MinDistinct1Pct
           AND @d2 >= CAST(@d1 * @MinDistinctGrowth AS bigint)
        BEGIN
            INSERT #Results
            SELECT
                @schema,
                @table,
                @column,
                @base_stat,
                @d1,
                @d2,
                CAST(@d2 AS float) / @d1,
                CAST(@d1 AS float) / @d2,
                (CAST(@d2 AS float) / @d1) - 1.0;

            PRINT '  -> Representative high-cardinality case found, skipping remaining columns';

			-- Cleanup
			SET @sql = N'DROP STATISTICS '
			         + QUOTENAME(@schema) + N'.'
			         + QUOTENAME(@table)  + N'.'
			         + QUOTENAME(@stat1);
			EXEC (@sql);
			
			SET @sql = N'DROP STATISTICS '
			         + QUOTENAME(@schema) + N'.'
			         + QUOTENAME(@table)  + N'.'
			         + QUOTENAME(@stat2);
			EXEC (@sql);

            BREAK; -- move to next table
        END

        -- Cleanup and continue
		SET @sql = N'DROP STATISTICS '
		         + QUOTENAME(@schema) + N'.'
		         + QUOTENAME(@table)  + N'.'
		         + QUOTENAME(@stat1);
		EXEC (@sql);
		
		SET @sql = N'DROP STATISTICS '
		         + QUOTENAME(@schema) + N'.'
		         + QUOTENAME(@table)  + N'.'
		         + QUOTENAME(@stat2);
		EXEC (@sql)

        FETCH NEXT FROM stat_cursor INTO @base_stat, @column;
    END

    CLOSE stat_cursor;
    DEALLOCATE stat_cursor;

    FETCH NEXT FROM table_cursor INTO @object_id, @schema, @table;
END

CLOSE table_cursor;
DEALLOCATE table_cursor;

----------------------------------------------------------------------
-- Final output
----------------------------------------------------------------------

SELECT *
FROM #Results
ORDER BY estimated_min_inaccuracy DESC;
GO

