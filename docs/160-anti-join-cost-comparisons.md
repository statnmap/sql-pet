Anti-join cost comparisons {#chapter_anti-join-cost-comparisons}




Verify Docker is up and running:

```r
sp_check_that_docker_is_up()
```

```
## [1] "Docker is up but running no containers"
```

Verify pet DB is available, it may be stopped.


```r
sp_show_all_docker_containers()
```

```
## CONTAINER ID        IMAGE                COMMAND                  CREATED              STATUS                     PORTS               NAMES
## 9bdc1e748065        postgres-dvdrental   "docker-entrypoint.s…"   About a minute ago   Exited (0) 2 seconds ago                       sql-pet
```

Start up the `docker-pet` container


```r
sp_docker_start("sql-pet")
```

Now connect to the database with R


```r
# need to wait for Docker & Postgres to come up before connecting.

con <- sp_get_postgres_connection(
  user = Sys.getenv("DEFAULT_POSTGRES_USER_NAME"),
  password = Sys.getenv("DEFAULT_POSTGRES_PASSWORD"),
  dbname = "dvdrental",
  seconds_to_test = 30
)
```


```r
dbExecute(  con,"delete from customer where customer_id >= 600;")
```

```
## [1] 5
```

```r
dbExecute(con, "delete from store where store_id > 2;")
```

```
## [1] 1
```

```r
dbExecute(
  con,
  "insert into customer 
  (customer_id,store_id,first_name,last_name,email,address_id,activebool
  ,create_date,last_update,active)
   values(600,3,'Sophie','Yang','sophie.yang@sakilacustomer.org',1,TRUE,now(),now()::date,1)
         ,(601,2,'Sophie','Yang','sophie.yang@sakilacustomer.org',1,TRUE,now(),now()::date,1)
         ,(602,4,'John','Smith','john.smith@sakilacustomer.org',2,TRUE,now()::date,now()::date,1)
         ,(603,5,'Ian','Frantz','ian.frantz@sakilacustomer.org',3,TRUE,now()::date,now()::date,1)
         ,(604,6,'Ed','Borasky','ed.borasky@sakilacustomer.org',4,TRUE,now()::date,now()::date,1)
         ;"
)
```

```
## [1] 5
```

```r
dbExecute(con, "ALTER TABLE store DISABLE TRIGGER ALL;")
```

```
## [1] 0
```

```r
df <- data.frame(
    store_id = 10
  , manager_staff_id = 10
  , address_id = 10
  , last_update = Sys.time()
)
dbWriteTable(con, "store", value = df, append = TRUE, row.names = FALSE)
dbExecute(con, "ALTER TABLE store ENABLE TRIGGER ALL;")
```

```
## [1] 0
```

Explain plans [here](https://robots.thoughtbot.com/reading-an-explain-analyze-query-plan)

## SQL anti join Costs


```r
sql_aj1 <- dbGetQuery(
  con,
  "explain analyze
   select 'aj' join_type, customer_id, first_name, last_name, c.store_id,count(*) ajs
   from customer c left outer join store s on c.store_id = s.store_id
  where s.store_id is null
  group by customer_id, first_name, last_name, c.store_id
order by c.customer_id;"
)

sql_aj2 <- dbGetQuery(
  con,
  "explain analyze
   select 'aj' join_type, customer_id, first_name, last_name, c.store_id,count(*) ajs
   from customer c 
  where c.store_id NOT IN (select store_id from store)
  group by  customer_id, first_name, last_name, c.store_id
order by c.customer_id;"
)

sql_aj3 <- dbGetQuery(
  con,
  "explain analyze
   select 'aj' join_type, customer_id, first_name, last_name, c.store_id,count(*) ajs
   from customer c 
  where not exists (select s.store_id from store s where s.store_id = c.store_id)
 group by customer_id, first_name, last_name, c.store_id
order by c.customer_id
"
)
```

##### SQL Costs


```r
print(glue("sql_aj1 loj-null costs=", sql_aj1[1, 1]))
```

```
## sql_aj1 loj-null costs=GroupAggregate  (cost=33.28..38.53 rows=300 width=266) (actual time=10.407..10.494 rows=4 loops=1)
```

```r
print(glue("sql_aj2 not in costs=", sql_aj2[1, 1]))
```

```
## sql_aj2 not in costs=GroupAggregate  (cost=29.86..35.11 rows=300 width=262) (actual time=0.312..0.399 rows=4 loops=1)
```

```r
print(glue("sql_aj3 not exist costs=", sql_aj3[1, 1]))
```

```
## sql_aj3 not exist costs=GroupAggregate  (cost=33.28..38.53 rows=300 width=262) (actual time=9.971..10.058 rows=4 loops=1)
```

## dplyr Anti joins  

In this next section we look at two methods to implemnt an anti join in dplyr.


```r
customer_table <- tbl(con, "customer") # DBI::dbReadTable(con, "customer")
rental_table <- tbl(con, "rental") # DBI::dbReadTable(con, "rental")

# Method 1.  dplyr anti_join
daj1 <-
  anti_join(customer_table, rental_table, by = "customer_id", suffix = c(".c", ".r")) %>%
  select(c("first_name", "last_name", "email")) %>%
  explain()
```

```
## <SQL>
## SELECT "first_name", "last_name", "email"
## FROM (SELECT * FROM "customer" AS "TBL_LEFT"
## 
## WHERE NOT EXISTS (
##   SELECT 1 FROM "rental" AS "TBL_RIGHT"
##   WHERE ("TBL_LEFT"."customer_id" = "TBL_RIGHT"."customer_id")
## )) "cqpqwqagrn"
```

```
## 
```

```
## <PLAN>
## Hash Anti Join  (cost=510.99..552.63 rows=300 width=334)
##   Hash Cond: ("TBL_LEFT".customer_id = "TBL_RIGHT".customer_id)
##   ->  Seq Scan on customer "TBL_LEFT"  (cost=0.00..14.99 rows=599 width=338)
##   ->  Hash  (cost=310.44..310.44 rows=16044 width=2)
##         ->  Seq Scan on rental "TBL_RIGHT"  (cost=0.00..310.44 rows=16044 width=2)
```


```r
customer_table <- tbl(con, "customer") # DBI::dbReadTable(con, "customer")
rental_table <- tbl(con, "rental") # DBI::dbReadTable(con, "rental")

# Method 2.  dplyr loj with NA
daj2 <-
  left_join(customer_table, rental_table, by = c("customer_id", "customer_id"), suffix = c(".c", ".r")) %>%
  filter(is.na(rental_id)) %>%
  select(c("first_name", "last_name", "email")) %>%
  explain()
```

```
## <SQL>
## SELECT "first_name", "last_name", "email"
## FROM (SELECT "TBL_LEFT"."customer_id" AS "customer_id", "TBL_LEFT"."store_id" AS "store_id", "TBL_LEFT"."first_name" AS "first_name", "TBL_LEFT"."last_name" AS "last_name", "TBL_LEFT"."email" AS "email", "TBL_LEFT"."address_id" AS "address_id", "TBL_LEFT"."activebool" AS "activebool", "TBL_LEFT"."create_date" AS "create_date", "TBL_LEFT"."last_update" AS "last_update.c", "TBL_LEFT"."active" AS "active", "TBL_RIGHT"."rental_id" AS "rental_id", "TBL_RIGHT"."rental_date" AS "rental_date", "TBL_RIGHT"."inventory_id" AS "inventory_id", "TBL_RIGHT"."return_date" AS "return_date", "TBL_RIGHT"."staff_id" AS "staff_id", "TBL_RIGHT"."last_update" AS "last_update.r"
##   FROM "customer" AS "TBL_LEFT"
##   LEFT JOIN "rental" AS "TBL_RIGHT"
##   ON ("TBL_LEFT"."customer_id" = "TBL_RIGHT"."customer_id")
## ) "ihebfvnxvb"
## WHERE ((("rental_id") IS NULL))
```

```
## 
```

```
## <PLAN>
## Hash Right Join  (cost=22.48..375.33 rows=80 width=334)
##   Hash Cond: ("TBL_RIGHT".customer_id = "TBL_LEFT".customer_id)
##   Filter: ("TBL_RIGHT".rental_id IS NULL)
##   ->  Seq Scan on rental "TBL_RIGHT"  (cost=0.00..310.44 rows=16044 width=6)
##   ->  Hash  (cost=14.99..14.99 rows=599 width=338)
##         ->  Seq Scan on customer "TBL_LEFT"  (cost=0.00..14.99 rows=599 width=338)
```

<!-- 


### dplyr Costs

```
<PLAN>
Hash Anti Join  (cost=510.99..529.72 rows=1 width=45)
  Hash Cond: ("TBL_LEFT".customer_id = "TBL_RIGHT".customer_id)
  ->  Seq Scan on customer "TBL_LEFT"  (cost=0.00..14.99 rows=599 width=49)
  ->  Hash  (cost=310.44..310.44 rows=16044 width=2)
        ->  Seq Scan on rental "TBL_RIGHT"  (cost=0.00..310.44 rows=16044 width=2)
```

```
<PLAN>
Hash Right Join  (cost=22.48..375.33 rows=1 width=45)
  Hash Cond: ("TBL_RIGHT".customer_id = "TBL_LEFT".customer_id)
  Filter: ("TBL_RIGHT".rental_id IS NULL)
  ->  Seq Scan on rental "TBL_RIGHT"  (cost=0.00..310.44 rows=16044 width=6)
  ->  Hash  (cost=14.99..14.99 rows=599 width=49)
        ->  Seq Scan on customer "TBL_LEFT"  (cost=0.00..14.99 rows=599 width=49)
```
-->


In this example, the dplyr anti_join verb is *1.4113447 to 22.7308719* times more expensive than the left outer join with a null condition.



```r
sql_aj1 <- dbGetQuery(
  con,
  "explain analyze select c.customer_id,count(*) lojs
   from customer c left outer join rental r on c.customer_id = r.customer_id
  where r.customer_id is null
  group by c.customer_id
order by c.customer_id;"
)
sp_print_df(sql_aj1)
```

<!--html_preserve--><div id="htmlwidget-124fb78127817ec02cd9" style="width:100%;height:auto;" class="datatables html-widget"></div>
<script type="application/json" data-for="htmlwidget-124fb78127817ec02cd9">{"x":{"filter":"none","data":[["1","2","3","4","5","6","7","8","9","10","11","12","13"],["GroupAggregate  (cost=564.97..570.22 rows=300 width=12) (actual time=256.401..256.499 rows=5 loops=1)","  Group Key: c.customer_id","  -&gt;  Sort  (cost=564.97..565.72 rows=300 width=4) (actual time=256.369..256.409 rows=5 loops=1)","        Sort Key: c.customer_id","        Sort Method: quicksort  Memory: 25kB","        -&gt;  Hash Anti Join  (cost=510.99..552.63 rows=300 width=4) (actual time=256.208..256.318 rows=5 loops=1)","              Hash Cond: (c.customer_id = r.customer_id)","              -&gt;  Seq Scan on customer c  (cost=0.00..14.99 rows=599 width=4) (actual time=0.018..4.199 rows=604 loops=1)","              -&gt;  Hash  (cost=310.44..310.44 rows=16044 width=2) (actual time=247.447..247.453 rows=16044 loops=1)","                    Buckets: 16384  Batches: 1  Memory Usage: 661kB","                    -&gt;  Seq Scan on rental r  (cost=0.00..310.44 rows=16044 width=2) (actual time=0.015..122.658 rows=16044 loops=1)","Planning time: 0.172 ms","Execution time: 256.672 ms"]],"container":"<table class=\"display\">\n  <thead>\n    <tr>\n      <th> <\/th>\n      <th>QUERY PLAN<\/th>\n    <\/tr>\n  <\/thead>\n<\/table>","options":{"order":[],"autoWidth":false,"orderClasses":false,"columnDefs":[{"orderable":false,"targets":0}]}},"evals":[],"jsHooks":[]}</script><!--/html_preserve-->

```r
sql_aj1
```

```
##                                                                                                                              QUERY PLAN
## 1                                 GroupAggregate  (cost=564.97..570.22 rows=300 width=12) (actual time=256.401..256.499 rows=5 loops=1)
## 2                                                                                                              Group Key: c.customer_id
## 3                                        ->  Sort  (cost=564.97..565.72 rows=300 width=4) (actual time=256.369..256.409 rows=5 loops=1)
## 4                                                                                                               Sort Key: c.customer_id
## 5                                                                                                  Sort Method: quicksort  Memory: 25kB
## 6                              ->  Hash Anti Join  (cost=510.99..552.63 rows=300 width=4) (actual time=256.208..256.318 rows=5 loops=1)
## 7                                                                                            Hash Cond: (c.customer_id = r.customer_id)
## 8                           ->  Seq Scan on customer c  (cost=0.00..14.99 rows=599 width=4) (actual time=0.018..4.199 rows=604 loops=1)
## 9                                  ->  Hash  (cost=310.44..310.44 rows=16044 width=2) (actual time=247.447..247.453 rows=16044 loops=1)
## 10                                                                                      Buckets: 16384  Batches: 1  Memory Usage: 661kB
## 11                     ->  Seq Scan on rental r  (cost=0.00..310.44 rows=16044 width=2) (actual time=0.015..122.658 rows=16044 loops=1)
## 12                                                                                                              Planning time: 0.172 ms
## 13                                                                                                           Execution time: 256.672 ms
```

```r
sql_aj3 <- dbGetQuery(
  con,
  "explain analyze 
select c.customer_id,count(*) lojs
   from customer c 
  where not exists (select customer_id from rental r where c.customer_id = r.customer_id)
 group by c.customer_id
"
)

print(glue("sql_aj1 loj-null costs=", sql_aj1[1, 1]))
```

```
## sql_aj1 loj-null costs=GroupAggregate  (cost=564.97..570.22 rows=300 width=12) (actual time=256.401..256.499 rows=5 loops=1)
```

```r
print(glue("sql_aj3 not exist costs=", sql_aj3[1, 1]))
```

```
## sql_aj3 not exist costs=HashAggregate  (cost=554.13..557.13 rows=300 width=12) (actual time=244.275..244.318 rows=5 loops=1)
```