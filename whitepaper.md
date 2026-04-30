Asymmetric Extrapolation in SQL Server Sampled Statistics Generation
How Unscaled DISTINCT_RANGE_ROWS Inflates AVG_RANGE_ROWS in Sampled Histograms and Destabilizes Range Cardinality Estimates, with suggested low- and ~zero-cost mitigations

________________________________________
1. Summary

Microsoft SQL Server applies deterministic extrapolation to point frequency estimates (EQ_ROWS) during sampled statistics generation but applies no corresponding adjustment to range frequency estimates (DISTINCT_RANGE_ROWS, and by extension AVG_RANGE_ROWS).

The extrapolation applied to EQ_ROWS only is easily reverse engineered, but mathematically is equally justifiable for both point and range frequencies. Sampled distinct counts are treated as complete observations even when sampling rates are low and cardinality is high.

As table cardinality increases and effective sampling rates fall – a typical consequence of many long lived systems – sampled distinct counts increasingly under represent true cardinality. Because AVG_RANGE_ROWS is derived directly from these unscaled distinct counts, average range frequency estimates inflate steadily over time, despite true per value frequency remaining constant in many cases. This eventually triggers abrupt and unexpected execution plan degradation.

This behaviour:

 - Is undocumented
 - Is independent of skew
 - Affects organically grown, real world systems earlier than synthetic bulk loaded data
 - Can eventually self mitigate at very high row counts due to undocumented fallback behaviour, but typically only after passing through a prolonged instability window

This paper demonstrates the behaviour empirically (Appendices 1–3), explains the underlying mathematical cause, describes partial workarounds, and outlines low cost engine level opportunities to mitigate using signals SQL Server already computes.

________________________________________
2. Core Observation

2.1 Asymmetric treatment of point vs range estimates

During sampled statistics generation:
 - EQ_ROWS values are deterministically extrapolated from sampled frequencies by approximately rows / rows_sampled. Additional logic moderates low frequency extrapolation, but by observed counts of ~ 5 or more the multiplier can be consistently reverse engineered - by dividing EQ_ROWS by rows/rows_sampled, which often produces almost exact integers (eg, 4.99999)
 - DISTINCT_RANGE_ROWS remains bounded by the number of distinct values observed in the sample, with seemingly no attempt to extrapolate.
 - AVG_RANGE_ROWS, derived directly from DISTINCT_RANGE_ROWS, therefore inflates as sampling rates fall, even when true per value frequency is unchanged.
 - Although EQ_ROWS may also inflate over time, it typically reflects the highest frequency values and therefore does not destabilize estimates at the same rate as AVG_RANGE_ROWS.

The consequence is asymmetric extrapolation: point frequencies are scaled, but the denominator governing average range frequencies is not. This causes range estimates to degrade far faster than point estimates under identical growth conditions.

________________________________________
3. Why This Matters in Real Systems

In many production schemas - particularly parent child relationships - the true average frequency is bounded (for example, 8–15 child rows per parent, such as answers per form), while the number of distinct parent keys grows over time.

In such systems:

 - True average frequency remains constant
 - Estimated average range frequency steadily increases
 - Query plans appear stable for long periods
 - Performance degrades abruptly when internal estimation thresholds are crossed

No workload change is required to trigger failure. Diagnosis is often time consuming, requires specialist expertise, and typical mitigations increase maintenance complexity rather than addressing root cause.

________________________________________
4. Reproducing the Behaviour

4.1 Synthetic Dataset (Appendix 1)
Appendix 1 provides the first successful replication approach used to isolate this behaviour. 
NOTE – replication is achieved by finding a way to work around SQL Server’s ability to infer frequency from logical and/or physical data layout, not by replicating real-world physical data layout. See Appendix 5 for SQL to identify examples in real systems.
Appendix 1’s SQL generates five tables (but further row counts can be appended to the opening @Targets table and will automatically generate similarly-named tables and statistics):

|Table|Row Count|
|--------|--------|
|FormAnswers_3m|3 million|
|FormAnswers_6m|6 million|
|FormAnswers_12m|12 million|
|FormAnswers_18m|18 million|
|FormAnswers_30m|30 million|

Construction method:

 - Fixed per key frequency (~89)
• - Data inserted in phases, specifically, 1/3 of data followed by 2/3 of data

This method was simply the first to reproduce the issue. I do not know if there is a way of generating data in a way which a mimics real world data. Real production systems tend to surface the issue earlier and more severely.

________________________________________
4.2 Statistics results across table sizes (Appendix 2)

Appendix 2 captures statistics on FormID for each table.

Despite identical true distribution:

 - Actual average frequency remains ~89 across all tables
 - Estimated averages diverge rapidly as table size increases

|Table|Avg Range Rows|Actual Avg|
|--------|--------|--------|
|3m|~310|~89|
|6m|~534|~89|
|12m|~990|~89|
|18m|~1399|~89|
|30m|~2078|~89|

Two critical findings:

1.	Statistics become progressively less representative as data grows, despite identical logical distribution.
2.	At sufficiently large row counts, SQL Server appears to invoke an undocumented fallback mechanism that arrests further divergence.

In practice, many systems fail long before this fallback is reached.

The existence and behaviour of this fallback are undocumented; it is included here for completeness but not explored further.

________________________________________
5. Deterministic scaling of EQ_ROWS (Appendix 4)

Inspection of histograms shows:

 - EQ_ROWS deltas stabilize to near constant increments
 - These increments closely match rows / rows_sampled
 - The multiplier is consistently recoverable

This demonstrates intentional, deterministic extrapolation, not sampling noise.

No comparable behaviour exists for DISTINCT_RANGE_ROWS, despite both quantities representing partial observations of the same underlying distribution.

________________________________________
6. Sampling rate sensitivity (Appendix 3)

Appendix 3 regenerates and analyzes statistics on the 30 million row table multiple times using explicit sample rates.

|Sample|Avg AVG_RANGE_ROWS|
|--------|--------|
|1%|~3200|
|2%|~2100|
|3%|~1377|
|5%|~837|

Key implications:

 - EQ_ROWS already scales with rows / rows_sampled
 - DISTINCT_RANGE_ROWS scales approximately linearly with sample size
 - For high cardinality data, linear scaling of distincts is strong evidence of systematic under representation
 - The divergence between estimated and true average frequency accelerates as sampling rates fall

This establishes that under representation is not only present, but empirically detectable using SQL Server’s own sampling behaviour.

________________________________________
7. Root cause summary

The progressive nature of the problem is not caused by:

 - Skew
 - Ascending keys
 - Stale statistics
 - Query volatility
 - Any commonly cited limitation of sampled statistics

It is caused by the combination of:

1.	Physical sampling over discontiguous data
2.	Deterministic extrapolation applied only to point frequencies
3.	Sampled distinct counts being treated as complete observations

________________________________________
8. Consequences for query optimisation

Inflated AVG_RANGE_ROWS directly affects:

 - Join algorithm selection
 - Memory grants (frequently excessive)
 - Parallelism decisions
 - Error propagation across joins

Failures often appear erratic and non intuitive, leading to fragile mitigations and excessive reliance on fullscan statistics.

________________________________________
9. Why Organically Grown Systems Are Hit Hardest

Sampling observes rows in physical order, not logical key order.

Over time:

 - Inserts interleave unrelated keys and other table data
 - Page splits destroy locality
 - Run length inference collapses

Logical data remains uniform; physical data does not.

Synthetic bulk loaded data therefore often masks the issue, explaining inconsistent reproduction attempts. The successful reproduction in Appendix 1 is simply the first method found of working around SQL Server’s ability to infer frequency from physical layout, enabling replication. It is not a reproduction of real-world physical data layout.

________________________________________
10. Opportunities for Improvement

10.1 Conditional scaling of DISTINCT_RANGE_ROWS

Applying the same multiplier used for EQ_ROWS directly to DISTINCT_RANGE_ROWS would correct the demonstrated failure case, but may degrade accuracy on other distributions.

Uniform extrapolation is not required. The goal is symmetric extrapolation when under representation is empirically detectable - a condition SQL Server can already observe at negligible cost.

________________________________________
10.2 Multi sample consistency and coverage estimation

Sampled distinct counts represent partial coverage of the true distinct domain. For high cardinality data, this coverage grows sub-linearly with sample size.

If SQL Server observes that:

 - Doubling the sample size produces a near linear increase in observed distinct values, and
 - This relationship persists across multiple sample fractions,

then the engine has direct evidence that sampled distincts materially under represent the true cardinality.

With two samples, SQL Server can detect systematic under sampling and infer that coverage is incomplete. With three samples, the engine can estimate both the curvature and asymptote of the distinct discovery curve without assuming that any single sample is “accurate enough.”

This allows estimation of:

 - The fraction of the distinct domain already observed (coverage)
 - A correction multiplier equal to 1 / coverage
 - A coverage based scaled estimate of DISTINCT_RANGE_ROWS inferred from sampling behaviour, which also generalises and could refine the existing rows / rows_sampled extrapolation applied to EQ_ROWS

Crucially, this converts the current exponential error growth of AVG_RANGE_ROWS under sampling into an approximately linear error that diminishes as sample size increases, bringing sampled statistics behaviour back in line with documented expectations of the relationship between sample size and histogram accuracy.

________________________________________
10.3 Internal consistency bounds

Cases where:

AVG_RANGE_ROWS > EQ_ROWS

are mathematically implausible, since EQ_ROWS reflects the most frequent values. As a lightweight safeguard, AVG_RANGE_ROWS could be bounded to a reasonable percentage of EQ_ROWS, with DISTINCT_RANGE_ROWS inferred accordingly.

________________________________________
10.4 Optional diagnostic aid (Appendix 5)

The intent of this paper is to describe and explain a systematic estimator behaviour in SQL Server, not to provide a comprehensive mechanism for diagnosing or remediating individual production systems.

Appendix 5 is therefore included as an informational aid only. It provides a script that compares distinct value discovery at 1% and 2% sampling rates across existing statistics in a database, without modifying or replacing those statistics. Material growth in observed distinct counts between 1% and 2% sample rates is a strong signal of incomplete distinct coverage, and therefore of elevated risk of AVG_RANGE_ROWS inflation under sampling and potential wider performance implications.

The script is not intended to be exhaustive, deterministic, or prescriptive. Absence of findings should not be interpreted as absence of risk, and presence of findings does not imply an immediate correctness issue in query execution. Its purpose is solely to demonstrate that the behaviour described in this paper can be detected empirically in real databases using signals already produced by the SQL Server statistics engine.

The script can be tuned via its opening configuration variables to reflect tolerances appropriate for specific environments or investigative use cases.

Finally, it should be noted that while SQL Server routinely regenerates statistics automatically including during peak usage periods, this script performs additional sampled statistics operations and should therefore be executed with appropriate care in production environments, as its resource usage is non trivial.

________________________________________
11. Operational Mitigations (Limited)

In the absence of an engine level change, limited operational mitigations are available. These do not address the underlying inconsistency in the sampling model, but may reduce its impact in specific operational contexts.

 - FULLSCAN statistics. Eliminates the issue by construction, as fullscan histograms accurately represent the distinct distribution of range values. This comes at the cost of increased statistics build time and maintenance overhead.
 - PERSIST_SAMPLE_PERCENT. A potentially lower cost alternative to fullscan statistics, preserving a higher effective sampling rate across rebuilds. Its usefulness depends on a more precise understanding of the specific estimation degradation being addressed, making it less generally applicable.

A practical short term workaround is to create a custom fullscan statistics object on an affected column at a controlled point in time, using OPTION (NORECOMPUTE) to reduce the likelihood of it being rebuilt by routine statistics maintenance. The optimizer should prioritze usage of a fullscan statistic over a default, sampled statistic where multiple exist for the same column, making it likely — though not guaranteed — that the fullscan statistic will be used for cardinality estimation.

This approach can temporarily restore accurate estimates without requiring repeated fullscan rebuilds during peak maintenance periods. Its effectiveness is inherently time bounded and dependent on workload characteristics, data change rates, and maintenance strategy. Over time, the statistic will become stale and estimation quality may again degrade, so this technique should be viewed as a probabilistic, interim mitigation, not a durable solution.

________________________________________
12. Conclusion

SQL Server’s sampled statistics encode an internally inconsistent model for high cardinality data under common growth patterns. Deterministic extrapolation is applied selectively, leaving range frequency estimates to diverge systematically as sampling rates fall.

The issue is:

 - Systematic
 - Observable
 - Undocumented
 - Addressable using signals SQL Server already computes, allowing behaviour to more closely in line with documented expectations
 - 
Until addressed at the engine level, this behaviour will continue to surface as unexplained instability in long lived production systems. 

/*
Appendix 1 – Generate replication tables and data
NOTE – the opening list of numbers into @Targets can be altered/appended to test tables of differing sizes. I found that somewhere between 30 and 60 million rows statistics became accurate ‘enough’ again, with only ~15% deviation from the truth. Possibly proving that problematic histogram generation only applies within a certain range.
*/
SET NOCOUNT ON;
GO

DECLARE @Targets TABLE (TargetRows int NOT NULL);
INSERT @Targets VALUES
(3000000),
(6000000),
(12000000),
(18000000),
(30000000)

DECLARE
    @TargetRows     int,
    @InitialRows    int,
    @ReplayRows     int,
    @EstimatedForms int,
    @TableName      sysname,
    @Table          sysname,
    @sql            nvarchar(max);

DECLARE c CURSOR LOCAL FAST_FORWARD FOR
SELECT TargetRows FROM @Targets;

OPEN c;
FETCH NEXT FROM c INTO @TargetRows;

WHILE @@FETCH_STATUS = 0
BEGIN
    /* Pre-compute values */
    SET @InitialRows    = @TargetRows / 3;
    SET @ReplayRows     = @TargetRows - @InitialRows;
    SET @EstimatedForms = (@InitialRows / 30) * 2; -- headroom

    SET @TableName = CONCAT('FormAnswers_', @TargetRows / 1000000, 'm');
    SET @Table     = QUOTENAME('dbo') + '.' + QUOTENAME(@TableName);

    /* Create table */
    SET @sql = N'
        IF OBJECT_ID(''' + @Table + ''',''U'') IS NOT NULL
            DROP TABLE ' + @Table + ';

        CREATE TABLE ' + @Table + '
        (
            FormID   int     NOT NULL,
            AnswerID bigint  NOT NULL
        );
    ';
    EXEC (@sql);

    /* PHASE 1: Initial 1/3 */
    SET @sql = N'
        ;WITH
        N AS
        (
            SELECT TOP (' + CONVERT(varchar(20), @EstimatedForms) + ')
                ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS FormID
            FROM sys.all_objects a
            CROSS JOIN sys.all_objects b
        ),
        FormFrequencies AS
        (
            SELECT
                FormID,
                CASE
                    WHEN FormID % 1000 = 0 THEN 100
                    WHEN FormID % 200  = 0 THEN 75
                    WHEN FormID % 50   = 0 THEN 50
                    WHEN FormID % 10   = 0 THEN 20
                    ELSE 30
                END AS RowsPerForm
            FROM N
        ),
        Expanded AS
        (
            SELECT f.FormID
            FROM FormFrequencies f
            CROSS APPLY
            (
                SELECT TOP (f.RowsPerForm) 1 AS n
                FROM sys.all_objects
            ) x
        ),
        Clipped AS
        (
            SELECT TOP (' + CONVERT(varchar(20), @InitialRows) + ')
                FormID
            FROM Expanded
            ORDER BY FormID
        ),
        Numbered AS
        (
            SELECT
                FormID,
                ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) AS AnswerID
            FROM Clipped
        )
        INSERT ' + @Table + ' (FormID, AnswerID)
        SELECT FormID, AnswerID
        FROM Numbered;
    ';
    EXEC (@sql);

    /* PHASE 2: Replay 2/3 from FormID = 1 */
    SET @sql = N'
        DECLARE @AnswerIDBase bigint;
        SELECT @AnswerIDBase = MAX(AnswerID) FROM ' + @Table + ';

        ;WITH
        ExistingCounts AS
        (
            SELECT FormID, COUNT(*) AS ExistingRows
            FROM ' + @Table + '
            GROUP BY FormID
        ),
        ReplayExpanded AS
        (
            SELECT e.FormID
            FROM ExistingCounts e
            CROSS APPLY
            (
                SELECT TOP (e.ExistingRows * 2) 1 AS n
                FROM sys.all_objects a
                CROSS JOIN sys.all_objects b
            ) x
        ),
        ReplayClipped AS
        (
            SELECT TOP (' + CONVERT(varchar(20), @ReplayRows) + ')
                FormID
            FROM ReplayExpanded
            ORDER BY FormID
        ),
        ReplayNumbered AS
        (
            SELECT
                FormID,
                ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) + @AnswerIDBase AS AnswerID
            FROM ReplayClipped
        )
        INSERT ' + @Table + ' (FormID, AnswerID)
        SELECT FormID, AnswerID
        FROM ReplayNumbered;
    ';
    EXEC (@sql);

    /* Final statistics */
    SET @sql = N'
        CREATE STATISTICS st_FormID ON ' + @Table + ' (FormID);
    ';
    EXEC (@sql);

    FETCH NEXT FROM c INTO @TargetRows;
END

CLOSE c;
DEALLOCATE c;
GO

/*
dbcc show_statistics (FormAnswers_3m, st_formid) with histogram
dbcc show_statistics (FormAnswers_6m, st_formid) with histogram
dbcc show_statistics (FormAnswers_12m, st_formid) with histogram
dbcc show_statistics (FormAnswers_18m, st_formid) with histogram
dbcc show_statistics (FormAnswers_30m, st_formid) with histogram
*/
 
/*
Appendix 2 – script to aid histogram investigation. Lists details of the histograms created on all tables created by the script in Appendix 1, and compares them to the truth they attempt to represent
*/
IF OBJECT_ID('tempdb..#Histogram') IS NOT NULL
    DROP TABLE #Histogram;

CREATE TABLE #Histogram
(
    RANGE_HI_KEY        sql_variant,
    RANGE_ROWS          float,
    EQ_ROWS             float,
    DISTINCT_RANGE_ROWS float,
    AVG_RANGE_ROWS      float
);

DECLARE @Results TABLE
(
    TableName			sysname,
    StatsName			sysname,
    EstimatedDistinct	float,
    ActualDistinct		bigint,
    AvgEqRows			float,
    AvgAvgRangeRows		float,
    ActualAvgFrequency	float
);

/*============================================================
  Discover matching tables
============================================================*/
DECLARE @SchemaName sysname;
DECLARE @TableName  sysname;
DECLARE @StatsName  sysname;
DECLARE @Sql        nvarchar(max);

DECLARE
    @ActualDistinct bigint,
    @ActualAvgFreq  float;

DECLARE c CURSOR LOCAL FAST_FORWARD FOR
SELECT
    s.name,
    t.name
FROM sys.tables t
JOIN sys.schemas s
    ON s.schema_id = t.schema_id
WHERE
    t.name LIKE 'FormAnswers[_]%m'
    AND t.is_ms_shipped = 0;

OPEN c;
FETCH NEXT FROM c INTO @SchemaName, @TableName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @StatsName = 'st_FormId';
    TRUNCATE TABLE #Histogram;
    SET @Sql = N'
        DBCC SHOW_STATISTICS (
            [' + @SchemaName + N'.' + @TableName + N'],
            [' + @StatsName + N']
        ) WITH HISTOGRAM;
    ';

    INSERT #Histogram
    EXEC (@Sql);

SET @Sql = N'
        SELECT
            @DistinctOut = COUNT(DISTINCT FormID),
            @AvgFreqOut  = AVG(1.0 * cnt)
        FROM
        (
            SELECT FormID, COUNT(*) AS cnt
            FROM ' + QUOTENAME(@SchemaName) + N'.' + QUOTENAME(@TableName) + N'
            GROUP BY FormID
        ) d;
    ';

    EXEC sp_executesql
        @Sql,
        N'@DistinctOut bigint OUTPUT, @AvgFreqOut float OUTPUT',
        @DistinctOut = @ActualDistinct OUTPUT,
        @AvgFreqOut  = @ActualAvgFreq  OUTPUT;

INSERT @Results
    SELECT
        QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName),
        @StatsName,
        AVG(EQ_ROWS)                 AS AvgEqRows,
        SUM(DISTINCT_RANGE_ROWS)     AS EstimatedDistinctHistogram,
        @ActualDistinct,
        AVG(AVG_RANGE_ROWS)          AS AvgAvgRangeRows,
        @ActualAvgFreq
    FROM #Histogram
    WHERE DISTINCT_RANGE_ROWS > 0;

    FETCH NEXT FROM c INTO @SchemaName, @TableName;
END

CLOSE c;
DEALLOCATE c;

SELECT
    TableName,
    StatsName,
    EstimatedDistinct,
    ActualDistinct,
	AvgEqRows,
    AvgAvgRangeRows,
    ActualAvgFrequency
FROM @Results
ORDER BY
    CONVERT(int,
        REPLACE(
            SUBSTRING(
                TableName,
                CHARINDEX('FormAnswers_', TableName) + LEN('FormAnswers_'),
                LEN(TableName)
            ),
            'm]',
            ''
        )
    ); 
/*
Appendix 3 – SQL to generate multiple statistics on the 30m table with different sample rates and show the results, showing that EQ_ROWS and AVG_RANGE_ROWS diverge as sample size increases, and providing the required data to infer an appropriate multiplier value for DISTINCT_RANGE_ROWS and potentially also to replace the existing multiplier for EQ_ROWS
*/
SET NOCOUNT ON;
GO

DECLARE @SchemaName sysname = N'dbo';
DECLARE @TableName  sysname = N'FormAnswers_30m';
DECLARE @StatsName  sysname = N'st_FormId';

IF OBJECT_ID('tempdb..#Histogram') IS NOT NULL
    DROP TABLE #Histogram;

CREATE TABLE #Histogram
(
    RANGE_HI_KEY        sql_variant,
    RANGE_ROWS          float,
    EQ_ROWS             float,
    DISTINCT_RANGE_ROWS float,
    AVG_RANGE_ROWS      float
);

IF OBJECT_ID('tempdb..#Results') IS NOT NULL
    DROP TABLE #Results;

CREATE TABLE #Results
(
    TableName                     sysname,
    StatsName                     sysname,
    SamplePct                     int,
    AvgEqRows                     float,
    EstimatedDistinctHistogram    float,
    ActualDistinctFormIDs         bigint,
    AvgAvgRangeRows               float,
    ActualAvgFrequency            float
);

DECLARE @Samples TABLE (SamplePct int NOT NULL);
INSERT @Samples VALUES (1),(2),(3),(5);

DECLARE
    @SamplePct      int,
    @Sql            nvarchar(max),
    @ActualDistinct bigint,
    @ActualAvgFreq  float;

DECLARE c CURSOR LOCAL FAST_FORWARD FOR
SELECT SamplePct FROM @Samples ORDER BY SamplePct;

OPEN c;
FETCH NEXT FROM c INTO @SamplePct;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @Sql = N'
        DROP STATISTICS ' + QUOTENAME(@SchemaName) + N'.'
                        + QUOTENAME(@TableName)  + N'.'
                        + QUOTENAME(@StatsName)  + N';
        
        CREATE STATISTICS ' + QUOTENAME(@StatsName) + N'
        ON ' + QUOTENAME(@SchemaName) + N'.'
             + QUOTENAME(@TableName) + N' (FormID)
        WITH SAMPLE ' + CONVERT(varchar(10), @SamplePct) + N' PERCENT;
    ';
    EXEC (@Sql);

    TRUNCATE TABLE #Histogram;

SET @Sql = N'
        DBCC SHOW_STATISTICS (
            ''' + @SchemaName + N'.' + @TableName + N''',
            ' + QUOTENAME(@StatsName) + N'
        ) WITH HISTOGRAM;
    ';
    INSERT #Histogram
    EXEC (@Sql);

    SET @Sql = N'
        SELECT
            @DistinctOut = COUNT(DISTINCT FormID),
            @AvgFreqOut  = AVG(1.0 * cnt)
        FROM
        (
            SELECT FormID, COUNT(*) AS cnt
            FROM ' + QUOTENAME(@SchemaName) + N'.' + QUOTENAME(@TableName) + N'
            GROUP BY FormID
        ) d;
    ';

    EXEC sp_executesql
        @Sql,
        N'@DistinctOut bigint OUTPUT, @AvgFreqOut float OUTPUT',
        @DistinctOut = @ActualDistinct OUTPUT,
        @AvgFreqOut  = @ActualAvgFreq  OUTPUT;

    INSERT #Results
    SELECT
        QUOTENAME(@SchemaName) + '.' + QUOTENAME(@TableName),
        @StatsName,
        @SamplePct,
        AVG(EQ_ROWS),
        SUM(DISTINCT_RANGE_ROWS),
        @ActualDistinct,
        AVG(AVG_RANGE_ROWS),
        @ActualAvgFreq
    FROM #Histogram
    WHERE DISTINCT_RANGE_ROWS > 0;

    FETCH NEXT FROM c INTO @SamplePct;
END
CLOSE c;
DEALLOCATE c;

SELECT
    TableName,
    StatsName,
    SamplePct        AS [Sample_%],
    AvgEqRows,
    EstimatedDistinctHistogram,
    ActualDistinctFormIDs,
    AvgAvgRangeRows,
    ActualAvgFrequency
FROM #Results
ORDER BY SamplePct;
GO
 
/*
Appendix 4: Reverse-engineering the EQ_ROWS extrapolation multiplier and associated other data, in turn revealing the sample frequency of values.
*/
SET NOCOUNT ON;

DECLARE @TableName sysname = N'dbo.FormAnswers_3m';
DECLARE @StatsName sysname = N'st_FormID';

DECLARE @ObjectId int = OBJECT_ID(@TableName);

IF @ObjectId IS NULL
BEGIN
    RAISERROR('Table not found: %s', 16, 1, @TableName);
    RETURN;
END;

DECLARE @StatsId int =
(
    SELECT s.stats_id
    FROM sys.stats AS s
    WHERE s.object_id = @ObjectId
      AND s.name = @StatsName
);

IF @StatsId IS NULL
BEGIN
    RAISERROR('Statistic not found: %s on table %s', 16, 1, @StatsName, @TableName);
    RETURN;
END;

DECLARE
    @Rows            bigint,
    @RowsSampled     bigint,
    @RowPerSample    float;  -- rows / rows_sampled

SELECT
    @Rows        = sp.rows,
    @RowsSampled = sp.rows_sampled
FROM sys.dm_db_stats_properties(@ObjectId, @StatsId) AS sp;

IF @RowsSampled IS NULL OR @RowsSampled = 0
BEGIN
    RAISERROR('rows_sampled is NULL/0 for %s on %s. Update statistics first.', 16, 1, @StatsName, @TableName);
    RETURN;
END;

SET @RowPerSample = (@Rows * 1.0) / @RowsSampled;

-- Capture histogram output
IF OBJECT_ID('tempdb..#Histogram') IS NOT NULL DROP TABLE #Histogram;
CREATE TABLE #Histogram
(
    RANGE_HI_KEY        sql_variant NULL,
    RANGE_ROWS          float       NULL,
    EQ_ROWS             float       NULL,
    DISTINCT_RANGE_ROWS float       NULL,
    AVG_RANGE_ROWS      float       NULL
);

DECLARE @sql nvarchar(max) =
    N'DBCC SHOW_STATISTICS (' + QUOTENAME(@TableName) + N', ' + QUOTENAME(@StatsName) + N') WITH HISTOGRAM;';

INSERT INTO #Histogram
EXEC (@sql);

;WITH H AS
(
    SELECT
        RANGE_HI_KEY,
        RANGE_ROWS,
        EQ_ROWS,
        DISTINCT_RANGE_ROWS,
        AVG_RANGE_ROWS,
        EQ_ROWS / NULLIF(@RowPerSample, 0.0) AS [EQ_ROWS / (rows/rows_sampled)]

    FROM #Histogram
)
SELECT
    @TableName  AS TableName,
    @StatsName  AS StatsName,
    @Rows       AS [rows],
    @RowsSampled AS [rows_sampled],
    @RowPerSample AS [rows/rows_sampled],
    RANGE_HI_KEY,
    RANGE_ROWS,
    EQ_ROWS,
    DISTINCT_RANGE_ROWS,
    AVG_RANGE_ROWS,
    [EQ_ROWS / (rows/rows_sampled)]
FROM H
ORDER BY
    -- you can also ORDER BY EQ_ROWS to see the "stabilised increments" region
    TRY_CONVERT(float, EQ_ROWS) ASC;
 
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

