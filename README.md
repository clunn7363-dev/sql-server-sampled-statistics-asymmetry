# sql-server-sampled-statistics-asymmetry
SQL Server sampled statistics extrapolate point frequencies (EQ_ROWS) but treat sampled distinct counts for range estimation as complete, creating an asymmetry between point and range estimates that can lead to unstable cardinality estimates and plan regressions. The attached paper demonstrates this empirically and outlines low‑cost remedies.
