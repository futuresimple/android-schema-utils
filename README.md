Android schema utils
====================
Android library for simplifying database schema and migrations management.

Features
========
The library provides the fluent API, which allows you to define current schema:
```java
@SuppressWarnings("deprecation")
private static final Schemas SCHEMA = Schemas.Builder
    .currentSchema(10,
        new TableDefinition(Tables.ADDRESSES, ImmutableList.<TableDefinitionOperation>builder()
            .addAll(getCommonColumns())
            .add(
                new AddColumn(Addresses.REGION, "TEXT"),
                new AddColumn(Addresses.ZIP, "TEXT"),
                new AddColumn(Addresses.COUNTRY, "TEXT"),
                new AddColumn(Addresses.CITY, "TEXT"),
                new AddColumn(Addresses.STREET, "TEXT ")
            )
            .build()
        ),
        // more tables
    )
```

Migrations that have to be performed:
```java
    .upgradeTo(7, recreate(Tables.STATS))
```

And downgrades that you need to apply to get the previous db schema:
```java
    .downgradeTo(4,
        new TableDowngrade(Tables.ADDRESSES,
            new DropColumn(Addresses.REGION)
        )
    )
    .downgradeTo(2, dropTable(Tables.ADDRESSES))
    .build();
```

This might look like a tedious, unnecessary work. In reality it is tedious, but very helpful work. It reduces the usual `db.execSQL()` boilerplate in `onCreate` and `onUpgrade` to this:
```java
@Override
public void onCreate(SQLiteDatabase db) {
  Schema currentSchema = SCHEMA.getCurrentSchema();
  for (String table : currentSchema.getTables()) {
    db.execSQL(currentSchema.getCreateTableStatement(table));
  }
}

@Override
public void onUpgrade(final SQLiteDatabase db, int oldVersion, int newVersion) {
  SCHEMA.upgrade(oldVersion, mContext, db);
}
```

### Bulletproof workaround for SQLite's ALTER TABLE deficiencies
SQLite's ALTER TABLE supports only renaming the table or adding a new column. If you want to do anything more fancy like dropping the column, you have to do the following things:

* Rename table with old schema to some temporary name
* Create table with new schema
* Move the data from renamed table to new table
* Drop the old table

This library provides tested wrapper for this boilerplate with nice API:

```java
TableMigration migration = TableMigration
    .of(Tables.MY_TABLE)
    .withMapping(MyTable.SOME_COLUMN, "NULL")
    .to(CREATE_MY_TABLE)
    .build();
    
migrationHelper.performMigrations(db, migration);
```

**Warning**: the MigrationHelper is not thread-safe (but seriously, why on earth would you want to perform sequential schema migrations in parallel?).

### Write only non-trivial migrations
Most of the migrations you perform are trivial: dropping column, adding nullable column, adding table, dropping table, etc. If you specify complete schema history (which you should do anyways), this library will figure out trivial migrations for you. Of course you can still define fully custom migrations or just combine your custom migrations with our automagic:

```java
.upgradeTo(2,
    auto(),
    new SimpleMigration() {
      @Override
      public void apply(SQLiteDatabase db, Schema schema) {
        // usual db.execSQL() 
      }
    }
)
```

### Reduce merge conflicts
In your `Schemas` definition you can include `release` checkpoints. All revision numbers before this checkpoint are in fact offsets from this revision. It helps a lot when you are merging two branches, which introduced changes to your schema.

You still have to merge the section of code with `currentSchema` and you still have to make sure that both branches haven't performed the same changes, but you don't have to juggle the revision numbers and in 90% of cases you just need to decide which batch of changes should go first.

### Automatic db index creation
Define the relationships between your data models using [Thneed](https://github.com/chalup/thneed) and use this information to generate the proper indexes:
```java
for (SQLiteIndex index : AutoIndexer.generateIndexes(MODEL_GRAPH)) {
  db.execSQL(AutoIndexer.getCreateStatement(index));
}
```

Note that this will generate the indexes for both ends of the relationships, which might not be exactly what you want. For example it will generate the index for primary keys referenced from other columns. We provide the `Predicate` factory to filter the generation results:
```java
FluentIterable<SQLiteIndex> indexes = FluentIterable
    .from(AutoIndexer.generateIndexes(MODEL_GRAPH))
    .filter(Predicates.not(isIndexOnColumn(ModelColumns.ID)))
    .filter(Predicates.not(isIndexOnColumn(BaseColumns._ID)));
```

And if you don't want index generation, you can still use our API to get the create index statement (although, I admit, it's not a killer feature):
```java
AutoIndexer.getCreateStatement(new SQLiteIndex("my_table", "foobar_id"));
```

Hint: when you're automagically generating the indexes, you want to automagically clean them up as well. Use [SQLiteMaster](https://github.com/futuresimple/sqlitemaster) utility to do this:
```java
SQLiteMaster.dropIndexes(db);
```

Usage
=====
Just add the dependency to your `build.gradle`:

```groovy
dependencies {
    compile 'com.getbase.android.schema:library:0.6'
}
```

License
=======

    Copyright (C) 2013 Jerzy Chalupski

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

         http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License. 
