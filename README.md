# ChronoModel [![Build Status](https://travis-ci.org/ifad/chronomodel.png?branch=master)](https://travis-ci.org/ifad/chronomodel) [![Dependency Status](https://gemnasium.com/ifad/chronomodel.png)](https://gemnasium.com/ifad/chronomodel) [![Code Climate](https://codeclimate.com/github/ifad/chronomodel.png)](https://codeclimate.com/github/ifad/chronomodel)

A temporal database system on PostgreSQL using
[upatable views](http://www.postgresql.org/docs/9.3/static/sql-createview.html#SQL-CREATEVIEW-UPDATABLE-VIEWS),
[table inheritance](http://www.postgresql.org/docs/9.3/static/ddl-inherit.html) and
[INSTEAD OF triggers](http://www.postgresql.org/docs/9.3/static/sql-createtrigger.html).

ChronoModel does what Oracle sells as "Flashback Queries", but with standard SQL on free PostgreSQL.
Academically speaking, ChronoModel implements a
[Type-2 Slowly-Changing Dimension](http://en.wikipedia.org/wiki/Slowly_changing_dimension#Type_2).

All the history keeping happens inside the database system, freeing the application code from
having to deal with it.


## Design

The application model is backed by an updatable view in the default `public` schema that behaves
like a plain table to any database client. When data in manipulated on it, INSTEAD OF [triggers](http://www.postgresql.org/docs/9.3/static/trigger-definition.html) redirect inserted
data to concrete tables.

Current data is hold in a table in the `temporal` [schema](http://www.postgresql.org/docs/9.3/static/ddl-schemas.html),
while history in hold in another table in the `history` schema. The latter
[inherits](http://www.postgresql.org/docs/9.3/static/ddl-inherit.html) from the former, to get
automated schema updates for free.
[Partitioning](http://www.postgresql.org/docs/9.3/static/ddl-partitioning.html)
of history is also possible: this design [fits the requirements](http://www.postgresql.org/docs/9.3/static/ddl-partitioning.html#DDL-PARTITIONING-CONSTRAINT-EXCLUSION) but it is not implemented yet.

See [README.sql](https://github.com/ifad/chronomodel/blob/master/README.sql) for a SQL
example defining the machinery for a simple table.


## Active Record integration

All Active Record schema migration statements are decorated with code that handles the temporal
structure by e.g. keeping the triggers in sync or dropping/recreating it when required by your
migrations.

Data extraction at a single point in time and even `JOIN`s between temporal and non-temporal data
is implemented using sub-selects and a `WHERE` generated by the provided `TimeMachine` module to
be included in your models.

The `WHERE` is optimized using [GiST indexes](http://www.postgresql.org/docs/9.3/static/gist.html) on
the `tsrange` that defines record validity. Overlapping history is prevented through [exclusion constraints](http://www.postgresql.org/docs/9.3/static/sql-createtable.html#SQL-CREATETABLE-EXCLUDE)
and the [btree_gist](http://www.postgresql.org/docs/9.3/static/btree-gist.html) extension.

All timestamps are _forcibly_ stored in as UTC, bypassing the `AR::Base.config.default_timezone` setting.


## Requirements

* Ruby &gt;= 1.9.3
* Active Record &gt;= 4.0
* PostgreSQL &gt;= 9.3
* The `btree_gist` PostgreSQL extension


## Installation

Add this line to your application's Gemfile:

    gem 'chrono_model', github: 'ifad/chronomodel'

And then execute:

    $ bundle


## Configuration

Configure your `config/database.yml` to use the `chronomodel` adapter:

    development:
      adapter: chronomodel
      username: ...


## Schema creation

ChronoModel hooks all `ActiveRecord::Migration` methods to make them temporal aware.

    create_table :countries, :temporal => true do |t|
      t.string :common_name
      t.references :currency
      # ...
    end

This creates the _temporal_ table, its inherited _history_ one the _public_ view
and all the trigger machinery.
Every other housekeeping of the temporal structure is handled behind the scenes
by the other schema statements. E.g.:

 * `rename_table`  - renames tables, views, sequences, indexes and triggers
 * `drop_table`    - drops the temporal table and all dependant objects
 * `add_column`    - adds the column to the current table and updates triggers
 * `rename_column` - renames the current table column and updates the triggers
 * `remove_column` - removes the current table column and updates the triggers
 * `add_index`     - creates the index in both _temporal_ and _history_ tables
 * `remove_index`  - removes the index from both tables


## Adding Temporal extensions to an existing table

Use `change_table`:

    change_table :your_table, :temporal => true

If you want to also set up the history from your current data:

    change_table :your_table, :temporal => true, :copy_data => true

This will create an history record for each record in your table, setting its
validity from midnight, January 1st, 1 CE. You can set a specific validity
with the `:validity` option:

    change_table :your_table, :temporal => true, :copy_data => true, :validity => '1977-01-01'


## Selective Journaling

By default UPDATEs only to the `updated_at` field are not recorded in the history.

You can also choose which fields are to be journaled, passing the following options to `create_table`:

  * `:journal => %w( fld1 fld2 .. .. )` - record changes in the history only when changing specified fields
  * `:no_journal => %w( fld1 fld2 .. )` - do not record changes to the specified fields
  * `:full_journal => true`             - record changes to *all* fields, including `updated_at`.

These options are stored as JSON in the [COMMENT](http://www.postgresql.org/docs/9.3/static/sql-comment.html)
area of the public view, alongside with the ChronoModel version that created them.

This is visible in `psql` if you issue a `\d+`. Example after a test run:

    chronomodel=# \d+
                                                           List of relations
     Schema |     Name      |   Type   |    Owner    |    Size    |                           Description
    --------+---------------+----------+-------------+------------+-----------------------------------------------------------------
     public | bars          | view     | chronomodel | 0 bytes    | {"temporal":true,"chronomodel":"0.7.0.alpha"}
     public | foos          | view     | chronomodel | 0 bytes    | {"temporal":true,"chronomodel":"0.7.0.alpha"}
     public | plains        | table    | chronomodel | 0 bytes    |
     public | test_table    | view     | chronomodel | 0 bytes    | {"temporal":true,"journal":["foo"],"chronomodel":"0.7.0.alpha"}


## Data querying

Include the `ChronoModel::TimeMachine` module in your model.

    module Country < ActiveRecord::Base
      include ChronoModel::TimeMachine

      has_many :compositions
    end

This will create a `Country::History` model inherited from `Country`, and add
an `as_of` class method.

    Country.as_of(1.year.ago)

Will execute:

    SELECT "countries".* FROM (
      SELECT "history"."countries".* FROM "history"."countries"
      WHERE '#{1.year.ago}' <@ "history"."countries"."validity"
    ) AS "countries"

The returned `ActiveRecord::Relation` will then hold and pass along the timestamp
given to the first `.as_of()` call to queries on associated entities. E.g.:

    Country.as_of(1.year.ago).first.compositions

Will execute:

    SELECT "countries".*, '#{1.year.ago}' AS as_of_time FROM (
      SELECT "history"."countries".* FROM "history"."countries"
      WHERE '#{1.year.ago}' <@ "history"."countries"."validity"
    ) AS "countries" LIMIT 1
    
and then, using the above fetched `as_of_time` timestamp, expand to:

    SELECT * FROM  (
      SELECT "history"."compositions".* FROM "history"."compositions"
      WHERE '#{as_of_time}' <@ "history"."compositions"."validity"
    ) AS "compositions" WHERE country_id = X

`.joins` works as well:

    Country.as_of(1.month.ago).joins(:compositions)

Expands to:

    SELECT "countries".* FROM (
      SELECT "history"."countries".* FROM "history"."countries"
      WHERE '#{1.year.ago}' <@ "history"."countries"."validity"
    ) AS "countries" INNER JOIN (
      SELECT "history"."compositions".* FROM "history"."compositions"
      WHERE '#{1.year.ago}' <@ "history"."compositions"."validity"
    ) AS "compositions" ON compositions.country_id = countries.id

More methods are provided, see the
[TimeMachine](https://github.com/ifad/chronomodel/blob/master/lib/chrono_model/time_machine.rb) source
for more information.


## Running tests

You need a running PostgreSQL 9.3 instance. Create `spec/config.yml` with the
connection authentication details (use `spec/config.yml.example` as template).

You need to connect as  a database superuser, because specs need to create the
`btree_gist` extension.

Run `rake`. SQL queries are logged to `spec/debug.log`. If you want to see them
in your output, use `rake VERBOSE=true`.

## Caveats

 * Rails 4 support requires disabling tsrange parsing support, as it
   [is broken](https://github.com/rails/rails/pull/13793#issuecomment-34608093) and
   [incomplete](https://github.com/rails/rails/issues/14010) as of now,
   mainly due to a [design clash with ruby Range](https://bugs.ruby-lang.org/issues/6864). 

 * There is (yet) no upgrade path from [v0.5](https://github.com/ifad/chronomodel/tree/c2daa0f),
   (PG 9.0-compatible, box() and hacks) to v0.6 and up (9.3-only, tsrange and _less_ hacks).

 * The triggers and temporal indexes cannot be saved in schema.rb. The AR
   schema dumper is quite basic, and it isn't (currently) extensible.
   As we're using many database-specific features, Chronomodel forces the
   usage of the `:sql` schema dumper, and included rake tasks override
   `db:schema:dump` and `db:schema:load` to do `db:structure:dump` and
   `db:structure:load`.
   Two helper tasks are also added, `db:data:dump` and `db:data:load`.

 * `.includes` is quirky when using `.as_of`.

 * The choice of using subqueries instead of [Common Table Expressions](http://www.postgresql.org/docs/9.3/static/queries-with.html)
   was dictated by the fact that CTEs [currently acts as an optimization
   fence](http://archives.postgresql.org/pgsql-hackers/2012-09/msg00700.php).
   If it will be possible [to opt-out of the
   fence](http://archives.postgresql.org/pgsql-hackers/2012-10/msg00024.php)
   in the future, they will be probably be used again as they were [in the
   past](https://github.com/ifad/chronomodel/commit/18f4c4b), because the
   resulting queries are much more readable, and do not inhibit using ARel's
   `.from()`.


## Contributing

 1. Fork it
 2. Create your feature branch (`git checkout -b my-new-feature`)
 3. Commit your changes (`git commit -am 'Added some great feature'`)
 4. Push to the branch (`git push origin my-new-feature`)
 5. Create new Pull Request
