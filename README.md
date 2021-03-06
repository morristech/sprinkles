Sprinkles ![Icon](https://github.com/emilsjolander/sprinkles/raw/master/sprinkles.png)
=========
Sprinkles is a boiler-plate-reduction-library for dealing with databases in android applications. Some would call is a kind of ORM but i don't see it that way. Sprinkles does lets SQL do what it is good at, making complex queries. SQL is however a mess (in my opinion) when is comes to everything else. This is why sprinkles helps you with things such as inserting, updated and destroying models, spinkles will also help you with the tedious task of unpacking a cursor into a model. Sprinkles actively supports version 2.3 of android and above but it should work on alder versions as well.

Sprinkles works great together with https://github.com/square/retrofit.

Download
--------
I prefer cloning my libraries and adding them as a dependency manually. This way i can easily fix a bug in the library as part of my workflow and commit it upstream (pleas do!).

However i know a lot of people like using gradle so here is the dependency to add to your `build.gradle`. Just replace `x.x.x` with the correct version of the library (found under the releases tab).

```Groovy
dependencies {
    compile 'se.emilsjolander:sprinkles:x.x.x'
}
```

Getting started
---------------
When you have added the library to your project add a model class to it. I will demonstrate this with a `Note.java` class. I have omitted import statements to keep it brief.
```java
@Table("Notes")
public class Note extends Model {

	@AutoIncrementPrimaryKey
	@Column("id")
	private long id;

	@Column("title")
	public String title;

	@Column("body")
	public String body;

	public long getId() {
		return id;
	}
	
}
```
Ok, a lot of important stuff in this short class. First of all. A model must subclass `se.emilsjolander.sprinkles.Model` and it also must have a `@Table` annotations specifying the table name the model corresponds to. After the class declaration we have declared three members: `id`, `title` and `body`. Notice how all of them have a `@Column` annotation to mark that they are not only a member of this class but also a column of the table that this class represents. We have one last annotation in the above example. The `@AutoIncrementPrimaryKey`, this annotation tells sprinkles that the field is both an autoincrement field and a primary key field. A field with this annotation will automatically be set upon the creation of its corresponding row in the table.

Before using this class you must migrate it into the database. I recomend doing thing in the `onCreate()` method of an `Application` subclass like this:
```java
public class MyApplication extends Application {
	
	@Override
	public void onCreate() {
		super.onCreate();
		
		Sprinkles sprinkles = Sprinkles.getInstance(getApplicationContext());
		
		Migration initialMigration = new Migration();
		initialMigration.createTable(Note.class);
		sprinkles.addMigration(initialMigration);
	}

}
```

Now you can happilty create new instances of this class and save it to the database like so:
```java
public void saveStuff() {
	Note n = new Note();
	n.title = "Sprinkles is awesome!";
	n.body = "yup, sure is!";
	n.save(); // when this call finishes n.getId() will return a valid id
}
```

You can also query for this note like this:
```java
public void queryStuff() {
	Note n = Query.one("select * from Notes where title=?", "Sprinkles is awesome!").get();
}
```

There is a lot more you can do with sprinkles so please read the next section which covers the whole api!

API
---
###Annotations
- `@Table` Used to associate a model class with a sql table.
- `@AutoIncrementPrimaryKey` Used to mark a field a an autoincrementing primary key. The field must be an `int` or a `long` and cannot be in the same class as any other primary key.
- `@Column` Used to associate a class field with a sql column.
- `@PrimaryKey` Used to mark a field as a primary key. Multiple primary keys in a class are allowed and will result in a composite primary key.
- `@ForeignKey` Used to mark a field as a foreign key. The argument given to this annotation should be in the form of foreignKeyTable(foreignKeyColumn).
- `@CascadeDelete` Used to mark a field also marked as a foreign key as a cascade deleting field.

###Saving
The save method is both an insert and a update method, the correct thing will be done depending on if the model exists in the database or not. The two first methods below and syncronous, the second is for using together with a transaction (more on the later). There are also two asyncronous methods, one with a callback and one without. The syncronous methods will return a boolean indicating if the model was saved or not, The asyncronous method with a callback will just not invoke the callback if saving failed.
```java
boolean save();
boolean save(Transaction t);
void saveAsync();
void saveAsync(OnSavedCallback callback);
```

All the save methods use this method to check if a model exists in the database. You are free to use it as well.
```java
boolean exists();
```

###Deleting
Similar to saving there are four methods that let you delete a model. These work in the same way as save but will not return a boolean indicating the result.
```java
void delete();
void delete(Transaction t);
void deleteAsync();
void deleteAsync(OnDeletedCallback callback);
```

###Querying
Start a query with on of the following static methods:
```java
Query.One(Class<? extends Model> clazz, String sql, Object[] args);
Query.Many(Class<? extends Model> clazz, String sql, Object[] args);
```
Notice that unlike android built in query methods you can send in an array of objects instead of an array of strings.

Once the query has been started you can get the result with three different methods:
```java
get();
getAsync(LoaderManager lm, OnQueryResultHandler<? extends Model> handler);
getAsyncWithUpdates(LoaderManager lm, OnQueryResultHandler<? extends Model> handler, Class<? extends Model>... dependencies);
```

`get()` return either the model or a list of the model represented by the `Class` you sent in as the first argument to the query method. `getAsync()` is the same only that the result is delivered on a callback function after the executeing `get()` on another thread. `getAsyncWithUpdates()` is the same as `getAsync()` only that it delivers updated results once the backing model of the query is updated. Both of the async methods use loaders and therefore need a `LoaderManager` instance. `getAsyncWithUpdates()` takes in an optional array of classes, this is used when the query relies on more models than the one you are querying for and you want the query to updated when those models change as well.

###ModelList
All Queries return a `ModelList` subclass. You can also instantiate a instance of this `List` subclass yourself. It has some nice helper methods for saving and deleting mutiple models in one go.

###Transactions
Both `save()` and `delete()` methods exists which take in a `Transaction`. Here is a quick example on how t use them. If any exception is thrown while saving a note or if any note fails to save the transaction will be rolled back.
```java
public void doTransaction(List<Note> notes) {
	Transaction t = new Transaction();
	try {
		for (Note n : notes) {
			if (n.save(t)) {
				return;
			}
		}
		t.setSuccessful(true);
	} finally {
		t.finish();
	}
}
```

###Callbacks
Each model subclass can override a couple of callbacks.

Use the following callback to ensure that your model is not saved in an invalid state.
```java
@Override
public boolean isValid() {
	// check model validity
}
```

Use the following callback to update a variable before the model is created
```java
@Override
protected void beforeCreate() {
	mCreatedAt = System.currentTimeMillis();
}
```

Use the following callback to update a variable before the model is saved. This is called directly before `beforeCreate()` if the model is saved for the first time.
```java
@Override
protected void beforeSave() {
	mUpdatedAt = System.currentTimeMillis();
}
```

User the following callback to clean up things related to the model but not stored in the database. Perhaps a file on the internal storage?
```java
@Override
protected void afterDelete() {
	// clean up some things?
}
```

###Migrations
Migrations are the way you add things to your database. I suggest putting all your migrations in the `onCreate()` method of a `Application` subclass. Here is a quick example of how that would look:
```java
public class MyApplication extends Application {
	
	@Override
	public void onCreate() {
		super.onCreate();
		
		Sprinkles sprinkles = Sprinkles.getInstance(getApplicationContext());
		
		Migration initialMigration = new Migration();
		initialMigration.createTable(Note.class);
		sprinkles.addMigration(initialMigration);
	}

}
```
The above example uses the `createTable()` method on a migration. Here is a full list of the methods it supports:
```java
void createTable(Class<? extends Model> clazz);
void dropTable(Class<? extends Model> clazz);
void renameTable(String from, String to);
void addColumn(Class<? extends Model> clazz, String columnName);
void addRawStatement(String statement);
```
Any number of calls to any of the above migrations are allowed, if for example `createTable()` is called twice than two tables will be created once that migration has been added. Remember to never edit a migration, always create a new migration (this only applies to production version of the app of course).

###Relationships
Sprinkles does nothing to handle relationships for you, this is by design. You will have to use the regular ways to handle relationships in sql. Sprinkles gives you all the tools needed for this and it works very well.

