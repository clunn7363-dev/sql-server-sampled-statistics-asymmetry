```sql
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