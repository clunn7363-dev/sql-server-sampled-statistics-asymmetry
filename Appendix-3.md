```sql
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