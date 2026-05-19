SET NOCOUNT ON;
GO

----------------------------------------------------------------------
-- Tunable parameters
----------------------------------------------------------------------
/*
Tunable parameters notes

@TopPlans
The number of currently cached execution plans to include in the check, starting from the most expensive

@MaxFindings
The total number of results to report

NOTE - Only one of either @TopPlans and @MaxFindings should be set to a high number. Setting both high can increase runtime exponentially. No issue with checking your entire DB though, if you don't mind the runtime
 - low @TopPlans with high @MaxFindings = interest in finding worst-affected expensive queries
 - high @TopPlans with low @MaxFindings = more general interest in finding worst-affected statistics, but which still affect queries in the workload

@MinRowCount
Included because only sufficiently large data is at risk

@MaxRowCount
Included to initially limit runtime of this script on databases with very large tables, to try to provide initial findings more quickly. Feel free to add zeros if you have very large tables and lots of time to wait

@MinDistinct1Pct
Sufficiently high cardinality is required for the issue to surface. No point wasting runtime on stats which already reasonably represent a relatively low number of distinct values

@MinDistinctGrowth
Sufficiently high growth in total distinct values between 1% and 2% samples is required for the issue to be relevant (open to interpretation)

NOTES ON @MinDistinctGrowth (look up 'Poisson Distribution' for further insight)
 - growth of below 1.25 (25%) between 1% and 2% samples is relatively statistically irrelevant
 - growth of 1.9+ is highly relevant, potentially indicating that a histogram's range estimates are unrealistically represented

See result column estimation_quality for an indication of how relevant a result is:
            WHEN growth_factor >= 1.95 THEN 'UNSTABLE_HIGH' -- its even possible for 2% to find MORE than double the distinct values which 1% found, in which case we don't bother with estimated_distinct_full
            WHEN growth_factor >= 1.5  THEN 'GOOD'
            WHEN growth_factor >= 1.25 THEN 'MODERATE'
            ELSE 'LOW'

@StatPrefix - prefix used for test statistic names to ensure no overlap with existing statistic names. Increase certainty by adding your own initials, the date, a hand smash onto the keyboard... anything to ensure uniqueness
*/

DECLARE @TopPlans int = 10;
DECLARE @MaxFindings int = 100;
DECLARE @MinRowCount bigint = 3000000;
DECLARE @MaxRowCount bigint = 100000000;
DECLARE @MinDistinct1Pct bigint = 1000;
DECLARE @MinDistinctGrowth float = 1.25;
DECLARE @StatPrefix sysname = N'__diag_sample_';

----------------------------------------------------------------------
-- Temp tables
----------------------------------------------------------------------

IF OBJECT_ID('tempdb..#TopPlans') IS NOT NULL DROP TABLE #TopPlans;
IF OBJECT_ID('tempdb..#PlanColumns') IS NOT NULL DROP TABLE #PlanColumns;
IF OBJECT_ID('tempdb..#CandidateStats') IS NOT NULL DROP TABLE #CandidateStats;
IF OBJECT_ID('tempdb..#StatCache') IS NOT NULL DROP TABLE #StatCache;
IF OBJECT_ID('tempdb..#Histogram') IS NOT NULL DROP TABLE #Histogram;
IF OBJECT_ID('tempdb..#Results') IS NOT NULL DROP TABLE #Results;

CREATE TABLE #TopPlans
(
    plan_handle varbinary(64),
    total_worker_time bigint,
    plan_rank int
);

CREATE TABLE #PlanColumns
(
    schema_name sysname,
    table_name  sysname,
    column_name sysname,
    plan_handle varbinary(64),
    plan_rank int
);

CREATE TABLE #CandidateStats
(
    object_id int,
    stats_id int,
    schema_name sysname,
    table_name sysname,
    column_name sysname,
    stats_name sysname,
    row_count bigint,
    plan_handle varbinary(64),
    plan_rank int
);

CREATE TABLE #StatCache
(
    object_id int,
    stats_id int,
    distinct_1pct bigint,
    distinct_2pct bigint,
    growth_factor decimal(10,2),
    coverage decimal(10,2),
    inaccuracy decimal(10,2),
    PRIMARY KEY (object_id, stats_id)
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
    schema_name sysname,
    table_name sysname,
    column_name sysname,
    stats_name sysname,
    distinct_1pct bigint,
    distinct_2pct bigint,
    growth_factor decimal(10,2),
    coverage decimal(10,2),
    inaccuracy decimal(10,2),
    estimated_distinct_full bigint,
    estimation_quality varchar(20),
    plan_handle varbinary(64),
    plan_rank int,
    query_plan xml
);

----------------------------------------------------------------------
-- Top plans
----------------------------------------------------------------------

INSERT #TopPlans
SELECT TOP (@TopPlans)
    qs.plan_handle,
    qs.total_worker_time,
    ROW_NUMBER() OVER (ORDER BY qs.total_worker_time DESC)
FROM sys.dm_exec_query_stats qs
WHERE qs.execution_count > 0
ORDER BY qs.total_worker_time DESC;

----------------------------------------------------------------------
-- Extract predicate columns
----------------------------------------------------------------------

WITH XMLNAMESPACES (DEFAULT 'http://schemas.microsoft.com/sqlserver/2004/07/showplan')

INSERT #PlanColumns
SELECT DISTINCT
    REPLACE(REPLACE(col.value('@Schema','sysname'),'[',''),']',''),
    REPLACE(REPLACE(col.value('@Table','sysname'),'[',''),']',''),
    REPLACE(REPLACE(col.value('@Column','sysname'),'[',''),']',''),
    tp.plan_handle,
    tp.plan_rank
FROM #TopPlans tp
CROSS APPLY sys.dm_exec_query_plan(tp.plan_handle) qp
CROSS APPLY qp.query_plan.nodes('//RelOp//ColumnReference') AS c(col)
WHERE
    col.value('@Column','sysname') IS NOT NULL
    AND col.value('@Schema','sysname') IS NOT NULL
    AND col.value('@Table','sysname') IS NOT NULL
    AND col.value('@Column','sysname') NOT LIKE 'Expr%'
    AND col.value('@Column','sysname') NOT LIKE 'ConstExpr%';

----------------------------------------------------------------------
-- Map to stats
----------------------------------------------------------------------

INSERT #CandidateStats
SELECT DISTINCT
    s.object_id,
    s.stats_id,
    sch.name,
    t.name,
    c.name,
    s.name,
    SUM(p.rows),
    pc.plan_handle,
    pc.plan_rank
FROM #PlanColumns pc
JOIN sys.schemas sch ON sch.name = pc.schema_name
JOIN sys.tables t ON t.name = pc.table_name AND t.schema_id = sch.schema_id
JOIN sys.columns c ON c.object_id = t.object_id AND c.name = pc.column_name
JOIN sys.stats_columns sc ON sc.object_id = c.object_id AND sc.column_id = c.column_id
	AND sc.stats_column_id = 1 -- a previous iteration was finding multiple columns for the same statistics object, but histograms only represent the first column
JOIN sys.stats s ON s.object_id = sc.object_id AND s.stats_id = sc.stats_id
JOIN sys.partitions p ON p.object_id = t.object_id AND p.index_id IN (0,1)
GROUP BY
    s.object_id,
    s.stats_id,
    sch.name,
    t.name,
    c.name,
    s.name,
    pc.plan_handle,
    pc.plan_rank
HAVING SUM(p.rows) BETWEEN @MinRowCount AND @MaxRowCount;

----------------------------------------------------------------------
-- Main loop
----------------------------------------------------------------------

DECLARE
    @object_id int,
    @stats_id int,
    @schema sysname,
    @table sysname,
    @column sysname,
    @stat sysname,
    @plan_handle varbinary(64),
    @plan_rank int,
    @stat1 sysname,
    @stat2 sysname,
    @sql nvarchar(max),
    @d1 bigint,
    @d2 bigint,
    @query_plan xml;

DECLARE cur CURSOR LOCAL FAST_FORWARD FOR
SELECT object_id, stats_id, schema_name, table_name, column_name, stats_name, plan_handle, plan_rank
FROM #CandidateStats
ORDER BY plan_rank;

OPEN cur;

FETCH NEXT FROM cur INTO
    @object_id, @stats_id, @schema, @table, @column, @stat, @plan_handle, @plan_rank;

WHILE @@FETCH_STATUS = 0
  AND (SELECT COUNT(*) FROM #Results) < @MaxFindings
BEGIN

    -- If already cached, skip heavy work
    IF EXISTS (
        SELECT 1 FROM #StatCache
        WHERE object_id = @object_id AND stats_id = @stats_id
    )
    BEGIN
        GOTO InsertResults;
    END

    SET @stat1 = @StatPrefix + CAST(@@SPID AS varchar(10)) + '_' + @stat + '_1pct';
    SET @stat2 = @StatPrefix + CAST(@@SPID AS varchar(10)) + '_' + @stat + '_2pct';

    BEGIN TRY

        -- Pre-clean
        SET @sql = N'DROP STATISTICS ' 
                 + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '.' + QUOTENAME(@stat1);
        EXEC (@sql);

        SET @sql = N'DROP STATISTICS ' 
                 + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '.' + QUOTENAME(@stat2);
        EXEC (@sql);

    END TRY
    BEGIN CATCH END CATCH;

    BEGIN TRY

        -- CREATE stats
        SET @sql = N'
            CREATE STATISTICS ' + QUOTENAME(@stat1) + '
            ON ' + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '
            (' + QUOTENAME(@column) + ') WITH SAMPLE 1 PERCENT;

            CREATE STATISTICS ' + QUOTENAME(@stat2) + '
            ON ' + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '
            (' + QUOTENAME(@column) + ') WITH SAMPLE 2 PERCENT;';
        EXEC (@sql);

        -- 1%
        TRUNCATE TABLE #Histogram;
        SET @sql = N'DBCC SHOW_STATISTICS (''' + @schema + '.' + @table + ''', ' + QUOTENAME(@stat1) + ') WITH HISTOGRAM;';
        INSERT #Histogram EXEC (@sql);
        SELECT @d1 = SUM(DISTINCT_RANGE_ROWS) FROM #Histogram;

        -- 2%
        TRUNCATE TABLE #Histogram;
        SET @sql = N'DBCC SHOW_STATISTICS (''' + @schema + '.' + @table + ''', ' + QUOTENAME(@stat2) + ') WITH HISTOGRAM;';
        INSERT #Histogram EXEC (@sql);
        SELECT @d2 = SUM(DISTINCT_RANGE_ROWS) FROM #Histogram;

        INSERT #StatCache
        SELECT
            @object_id,
            @stats_id,
            @d1,
            @d2,
            CAST(@d2 * 1.0 / NULLIF(@d1,0) AS decimal(10,2)),
            CAST(@d1 * 1.0 / NULLIF(@d2,0) AS decimal(10,2)),
            CAST(@d2 * 1.0 / NULLIF(@d1,0) - 1 AS decimal(10,2));

    END TRY
    BEGIN CATCH
        PRINT 'Error processing stat ' + @stat;
    END CATCH

    -- Always cleanup
    BEGIN TRY
        SET @sql = N'DROP STATISTICS ' + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '.' + QUOTENAME(@stat1);
        EXEC (@sql);
        SET @sql = N'DROP STATISTICS ' + QUOTENAME(@schema) + '.' + QUOTENAME(@table) + '.' + QUOTENAME(@stat2);
        EXEC (@sql);
    END TRY
    BEGIN CATCH END CATCH;

InsertResults:

    SET @query_plan = (SELECT query_plan FROM sys.dm_exec_query_plan(@plan_handle));

    INSERT #Results
    SELECT
        @schema,
        @table,
        @column,
        @stat,
        distinct_1pct,
        distinct_2pct,
        growth_factor,
        coverage,
        inaccuracy,

		CASE 
			WHEN distinct_1pct = 0 THEN NULL
			ELSE
				CAST(
					CASE 
						WHEN distinct_2pct * 1.0 / distinct_1pct >= 1.98
							THEN distinct_2pct * 50
						ELSE
							distinct_1pct * 1.0 /
							(2.0 - (distinct_2pct * 1.0 / distinct_1pct))
					END
				AS bigint)
		END,

        CASE
            WHEN growth_factor >= 1.95 THEN 'UNSTABLE_HIGH'
            WHEN growth_factor >= 1.5  THEN 'GOOD'
            WHEN growth_factor >= 1.25 THEN 'MODERATE'
            ELSE 'LOW'
        END,

        @plan_handle,
        @plan_rank,
        @query_plan
    FROM #StatCache
    WHERE object_id = @object_id
      AND stats_id = @stats_id
      AND distinct_1pct >= @MinDistinct1Pct
      AND distinct_2pct >= distinct_1pct * @MinDistinctGrowth;

    FETCH NEXT FROM cur INTO
        @object_id, @stats_id, @schema, @table, @column, @stat, @plan_handle, @plan_rank;
END

CLOSE cur;
DEALLOCATE cur;

----------------------------------------------------------------------
-- Final output
----------------------------------------------------------------------

SELECT *
FROM #Results
ORDER BY plan_rank, inaccuracy DESC;
GO