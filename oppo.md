```
SET SESSION TiDB_batch_insert = 1;
SET SESSION TiDB_batch_delete = 1;set autocommit=1;
```

```
set @@session.TiDB_dml_batch_size = 5000; 
```

```
timeout queue 30m
timeout connect 30m
timeout client 30m
timeout server 30m
```

`location_labels = ["host"]`

```
config set location-labels "host"
```

```
dbname.tablename.columnname 
```

```
select dbname.tablename.columnname from dbname.tablename 
```

```
select A.columnname from dbname.tablename  as A 
```