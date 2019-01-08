---
layout:      post
title:       "Hibernate: Multi-table update/delete handling strategies"
date:        2016-12-24 15:20:47
update_date: 2017-07-18 07:26:00
summary:     "Hibernate's MultiTableBulkIdStrategy and its implementations"
categories:  hibernate
hidden:      true
---

If you ever used a hierarchical entity model with Hibernate, you might have noticed similar DDL statements being generated on application startup:

```sql
declare global temporary table session.HT_PARENT_1 (ID varchar(36) not null) not logged
declare global temporary table session.HT_CHILD_1 (ID varchar(36) not null) not logged
declare global temporary table session.HT_CHILD_2 (ID varchar(36) not null) not logged
...
```

What's going on here? Let's take a closer look.

## Entity inheritance mapping

Consider the following entity hierarchy:

```java
@Entity
@Table(name = "EMPLOYEE")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Employee {
    @Id
    @Column(name = "ID", length = 36)
    protected UUID employeeId;
    // ...
}

@Entity
@Table(name = "EMPLOYEE_FULL_TIME")
public class FullTimeEmployee extends Employee {
    protected int salary;
    // ...
}

@Entity
@Table(name = "EMPLOYEE_PART_TIME")
public class PartTimeEmployee extends Employee {
    protected float hourlyWage;
    // ...
}
```

Because `InheritanceType.JOINED` is used, the entity hierarchy is mapped to the following database schema:

```sql
create table EMPLOYEE (ID varchar(36) not null, primary key (ID))
create table EMPLOYEE_FULL_TIME (ID varchar(36) not null, salary integer, primary key (ID))
create table EMPLOYEE_PART_TIME (ID varchar(36) not null, hourlyWage float, primary key (ID))
alter table EMPLOYEE_FULL_TIME add constraint FK_emp_id_ft foreign key (ID) references EMPLOYEE
alter table EMPLOYEE_PART_TIME add constraint FK_emp_id_pt foreign key (ID) references EMPLOYEE
```

A fairly straightforward mapping. But what happens once a bulk update or delete operation is executed?

## Bulk updates and deletes

There are some challenges explained well in this [Multi-table Bulk Operations](http://in.relation.to/2005/07/20/multitable-bulk-operations/) post. The essence is that, for example, deleting a multi-table entity (like `FullTimeEmployee`) means:

* Finding IDs for rows that need to be deleted and storing them somewhere
* Deleting rows from multiple tables (in proper order!) by previously stored ID values

As you see, the IDs have to be stored somewhere. Let's consider two approaches.

#### In-memory approach

An obvious approach is to keep the IDs around in application's memory. The following SQL statements would have to be generated then:

```sql
select ID from EMPLOYEE_FULL_TIME where SALARY > 1000000;
delete from EMPLOYEE_FULL_TIME where ID IN ( [list of IDs] );
delete from EMPLOYEE where ID IN ( [list of IDs] );
```

However, the simplicity of this approach carries a performance penalty (transferring IDs over network, mapping them to/from Java objects, etc.).

#### Table-based approach

A more performant approach is to store the selected ID values on the database server itself. The generated SQL would be similar to this: 

```sql
insert into ID_TABLE (
   select ID from EMPLOYEE_FULL_TIME where SALARY > 1000000
);
delete from EMPLOYEE_FULL_TIME where (ID) IN (select ID from ID_TABLE);
delete from EMPLOYEE where (ID) IN (select ID from ID_TABLE);
```

Note that as different entities could have different ID definitions, an equivalent of `ID_TABLE` needs to exist for each entity that is a part of any entity hierarchy. That could be a significant number of tables.

Let's take a look at what Hibernate can do for us here.

## Hibernate's `MultiTableBulkIdStrategy` implementations

Hibernate has defined a generalized interface for handling multi-table bulk HQL operations called  `MultiTableBulkIdStrategy`. Currently (Hibernate 5.2.6) ships with three implementations of this interface, all of them table-based:

* `GlobalTemporaryTableBulkIdStrategy` - based on ANSI SQL's definition of a "global temporary table".
* `LocalTemporaryTableBulkIdStrategy` - based on ANSI SQL's definition of a "local temporary table" (local to each database session).
* `PersistentTableBulkIdStrategy` - based on regular tables with an extra `hib_sess_id` column to segment rows from different Hibernate sessions.

The strategy to use is determined by `hibernate.hql.bulk_id_strategy` property, with `Dialect#getDefaultMultiTableBulkIdStrategy` used as a fallback.

#### Table-creation at startup

Both `GlobalTemporaryTableBulkIdStrategy` and `PersistentTableBulkIdStrategy` try to create the ID tables at startup and that is where the DDL statements mentioned in the beginning of the post come from.

It's worth noting that any of these DDL statements that fail are just logged at `DEBUG` level and quietly ignored. That is, until a bulk update or delete operation is executed, at which point it would fail if the required table does not exist.

The point here is that failing to create the ID tables does not prevent application from starting and working, as long as:

* No bulk update or delete operations are used, or
* The ID tables are pre-created by other means (e.g. by a DBA having required authorizations).

There are also alternative implementations that might be useful in some circumstances.

## Alternative implementations

Alternative strategy implementations discussed below might help you when:

* DDL statements cannot be executed by your application
* Bulk update/delete operations are actually used
* Pre-creating ID tables is either difficult or impossible

#### When pre-creating ID tables is difficult

Assume there's a way for you to get the ID tables created, either by doing some manual work or by involving other parties who can do it for you.

If your entity model is stable, you might just bite the bullet and do the work to create the ID tables once and for all. Afterwards you could forget about it and stick with using one of Hibernate's standard strategies. However, if your model is constantly evolving, you might want to alleviate the pain of dealing with extra ID tables.

For this purpose I created a [Single Global Temporary Table Bulk Id Strategy](https://github.com/grimsa/hibernate-single-table-bulk-id-strategy). It's a slightly modified `GlobalTemporaryTableBulkIdStrategy` that uses a single shared global temporary table to store IDs for all entities, segmenting the rows by entity name held in an extra column.

The benefit of it is that you retain the advantages of the table-based approach, while also minimizing the overhead to create and manage tables for each entity.

The strategy was developed for a project where the same ID type and column name was used across all entities. Naturally, the same type and name was also used for the shared ID table too.

It should either work out-of-the-box or be easily extendable to support other cases (different types and/or names of single-column IDs, maybe even multi-column IDs), but you should verify it works for your circumstances.

#### When pre-creating ID tables is impossible

If there's no way to create the ID tables at all, then any of the built-in table-based strategies are not an option. In this case you'd have to turn to the in-memory approach, which, to my surprise, is not implemented in Hibernate.

Luckily, there's [CTE Bulk Id Strategy](https://github.com/epiresdasilva/cte-multi-table-bulk-id-stategy).

The project includes tests on PostgreSQL, although when I tried it on DB2 and H2 databases a while ago, it did not work. Hence, minor alterations might once again be needed.

## In place of a summary

The need for bulk update or delete operations (based on non-PK columns) on entity hierarchies is not a very common one, and Hibernate's ID table-based strategies cover this need well in most environments.

However, when creation of ID tables is either problematic or not possible, one of existing alternative `MultiTableBulkIdStrategy` implementations could be used.

Or, as they say, you can always roll your own.

## 2017-07-18 Update

Hibernate 5.2.8 added a number of new [Bulk-id strategies when you can’t use temporary tables](http://in.relation.to/2017/02/01/non-temporary-table-bulk-id-strategies/).

These new strategies have only slight differences and a good starting point is to go with `InlineIdsOrClauseBulkIdStrategy`, which is supported by all major relational database systems.