```sql
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