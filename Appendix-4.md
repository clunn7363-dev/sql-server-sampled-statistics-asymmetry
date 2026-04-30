```sql
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