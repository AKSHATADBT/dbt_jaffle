
# catalog building

## same database

### get relations in the `ryanw` schema
```sql
select
        table_catalog as database,
        table_name as name,
        table_schema as schema,
        'table' as type
    from information_schema.tables
    where table_schema ilike 'ryanw'
    and table_type = 'BASE TABLE'
    union all
    select
      table_catalog as database,
      table_name as name,
      table_schema as schema,
      case
        when view_definition ilike '%create materialized view%'
          then 'materialized_view'
        else 'view'
      end as type
    from information_schema.views
    where table_schema ilike 'ryanw'
```


### build DAG of model dependencies

```sql
with
    relation as (
        select
            pg_class.oid as relation_id,
            pg_class.relname as relation_name,
            pg_class.relnamespace as schema_id,
            pg_namespace.nspname as schema_name,
            pg_class.relkind as relation_type
        from pg_class
        join pg_namespace
          on pg_class.relnamespace = pg_namespace.oid
        where pg_namespace.nspname != 'information_schema'
          and pg_namespace.nspname not like 'pg\_%'
    ),
    dependency as (
        select distinct
            coalesce(pg_rewrite.ev_class, pg_depend.objid) as dep_relation_id,
            pg_depend.refobjid as ref_relation_id,
            pg_depend.refclassid as ref_class_id
        from pg_depend
        left join pg_rewrite
          on pg_depend.objid = pg_rewrite.oid
        where coalesce(pg_rewrite.ev_class, pg_depend.objid) != pg_depend.refobjid
    )

select distinct
    dep.schema_name as dependent_schema,
    dep.relation_name as dependent_name,
    ref.schema_name as referenced_schema,
    ref.relation_name as referenced_name
from dependency
join relation ref
    on dependency.ref_relation_id = ref.relation_id
join relation dep
    on dependency.dep_relation_id = dep.relation_id
```

### get column metadata

```sql
with
            late_binding as (
    
    select
        table_schema,
        table_name,
        'LATE BINDING VIEW'::varchar as table_type,
        null::text as table_comment,
        column_name,
        column_index,
        column_type,
        null::text as column_comment
    from pg_get_late_binding_view_cols()
        cols(
            table_schema name,
            table_name name,
            column_name name,
            column_type varchar,
            column_index int
        )

    where ((
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('stg_payments')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('stg_orders')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('customers')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('raw_orders')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('hogehoge')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('stg_customers')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('raw_payments')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('orders')
            ) or (
                upper(table_schema) = upper('ryanw')
            and upper(table_name) = upper('raw_customers')
            ))
),
            early_binding as (
    
    select
        sch.nspname as table_schema,
        tbl.relname as table_name,
        case
            when tbl.relkind = 'v' and mat_views.table_name is not null then 'MATERIALIZED VIEW'
            when tbl.relkind = 'v' then 'VIEW'
            else 'BASE TABLE'
        end as table_type,
        tbl_desc.description as table_comment,
        col.attname as column_name,
        col.attnum as column_index,
        pg_catalog.format_type(col.atttypid, col.atttypmod) as column_type,
        col_desc.description as column_comment
    from pg_catalog.pg_namespace sch
    join pg_catalog.pg_class tbl
        on tbl.relnamespace = sch.oid
    join pg_catalog.pg_attribute col
        on col.attrelid = tbl.oid
    left outer join pg_catalog.pg_description tbl_desc
        on tbl_desc.objoid = tbl.oid
        and tbl_desc.objsubid = 0
    left outer join pg_catalog.pg_description col_desc
        on col_desc.objoid = tbl.oid
        and col_desc.objsubid = col.attnum
    left outer join information_schema.views mat_views
        on mat_views.table_schema = sch.nspname
        and mat_views.table_name = tbl.relname
        and mat_views.view_definition ilike '%create materialized view%'
        and mat_views.table_catalog = 'ci'
    where tbl.relkind in ('r', 'v', 'f', 'p')
    and col.attnum > 0
    and not col.attisdropped

    and ((
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('stg_payments')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('stg_orders')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('customers')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('raw_orders')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('hogehoge')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('stg_customers')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('raw_payments')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('orders')
            ) or (
                upper(sch.nspname) = upper('ryanw')
            and upper(tbl.relname) = upper('raw_customers')
            ))
),
            unioned as (select * from early_binding union all select * from late_binding),
            table_owners as (
    select
        schemaname as table_schema,
        tablename as table_name,
        tableowner as table_owner
    from pg_tables
    union all
    select
        schemaname as table_schema,
        viewname as table_name,
        viewowner as table_owner
    from pg_views
)
        select 'ci' as table_database, *
        from unioned
        join table_owners using (table_schema, table_name)
        order by "column_index"
```

### extended catalog

`svv_table_info` is queried one time for each database. for each database call, only the tables/views are touched



```sql
    select
        "schema" as table_schema,
        "table" as table_name,

        'Encoded'::text as "stats:encoded:label",
        encoded as "stats:encoded:value",
        'Indicates whether any column in the table has compression encoding defined.'::text as "stats:encoded:description",
        true as "stats:encoded:include",

        'Dist Style' as "stats:diststyle:label",
        diststyle as "stats:diststyle:value",
        'Distribution style or distribution key column, if key distribution is defined.'::text as "stats:diststyle:description",
        true as "stats:diststyle:include",

        'Sort Key 1' as "stats:sortkey1:label",
        -- handle 0xFF byte in response for interleaved sort styles
        case
            when sortkey1 like 'INTERLEAVED%' then 'INTERLEAVED'::text
            else sortkey1
        end as "stats:sortkey1:value",
        'First column in the sort key.'::text as "stats:sortkey1:description",
        (sortkey1 is not null) as "stats:sortkey1:include",

        'Max Varchar' as "stats:max_varchar:label",
        max_varchar as "stats:max_varchar:value",
        'Size of the largest column that uses a VARCHAR data type.'::text as "stats:max_varchar:description",
        true as "stats:max_varchar:include",

        -- exclude this, as the data is strangely returned with null-byte characters
        'Sort Key 1 Encoding' as "stats:sortkey1_enc:label",
        sortkey1_enc as "stats:sortkey1_enc:value",
        'Compression encoding of the first column in the sort key.' as "stats:sortkey1_enc:description",
        false as "stats:sortkey1_enc:include",

        '# Sort Keys' as "stats:sortkey_num:label",
        sortkey_num as "stats:sortkey_num:value",
        'Number of columns defined as sort keys.' as "stats:sortkey_num:description",
        (sortkey_num > 0) as "stats:sortkey_num:include",

        'Approximate Size' as "stats:size:label",
        size * 1000000 as "stats:size:value",
        'Approximate size of the table, calculated from a count of 1MB blocks'::text as "stats:size:description",
        true as "stats:size:include",

        'Disk Utilization' as "stats:pct_used:label",
        pct_used / 100.0 as "stats:pct_used:value",
        'Percent of available space that is used by the table.'::text as "stats:pct_used:description",
        true as "stats:pct_used:include",

        'Unsorted %' as "stats:unsorted:label",
        unsorted / 100.0 as "stats:unsorted:value",
        'Percent of unsorted rows in the table.'::text as "stats:unsorted:description",
        (unsorted is not null) as "stats:unsorted:include",

        'Stats Off' as "stats:stats_off:label",
        stats_off as "stats:stats_off:value",
        'Number that indicates how stale the table statistics are; 0 is current, 100 is out of date.'::text as "stats:stats_off:description",
        true as "stats:stats_off:include",

        'Approximate Row Count' as "stats:rows:label",
        tbl_rows as "stats:rows:value",
        'Approximate number of rows in the table. This value includes rows marked for deletion, but not yet vacuumed.'::text as "stats:rows:description",
        true as "stats:rows:include",

        'Sort Key Skew' as "stats:skew_sortkey1:label",
        skew_sortkey1 as "stats:skew_sortkey1:value",
        'Ratio of the size of the largest non-sort key column to the size of the first column of the sort key.'::text as "stats:skew_sortkey1:description",
        (skew_sortkey1 is not null) as "stats:skew_sortkey1:include",

        'Skew Rows' as "stats:skew_rows:label",
        skew_rows as "stats:skew_rows:value",
        'Ratio of the number of rows in the slice with the most rows to the number of rows in the slice with the fewest rows.'::text as "stats:skew_rows:description",
        (skew_rows is not null) as "stats:skew_rows:include"

    from svv_table_info

        where ((
                    upper("schema") = upper('ci')
                and upper("table") = upper('hogehoge')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('stg_orders')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('stg_payments')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('raw_orders')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('raw_payments')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('orders')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('stg_customers')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('customers')
                ) or (
                    upper("schema") = upper('ci')
                and upper("table") = upper('raw_customers')
                ))
```

## multi-db


the `fizz` db query is exactly the same except for the where clause which looks like

```sql
where ((
        upper("schema") = upper('ci')
    and upper("table") = upper('raw_customers')
    ))
```