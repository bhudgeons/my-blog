title: A Slick Tool for Database Schema Generation
date: 2013-29-06
author: Brandon Hudgeons
tags: [scala, slick, migrations, mysql, bhudgeons]


*tl;dr: The Slick migrations tool will be a badass way to keep your database in version control.  In the meantime: need to generate a Slick schema to apply to a legacy database? See [my fork of the Slick migration tool proof-of-concept](https://github.com/bhudgeons/migrations).*

####The Way to Do RDBMS in Scala

A common use case for Scala is to replace or supplement a legacy system by replacing the system altogether or by (for example) [taking over the API layer](http://www.artima.com/scalazine/articles/twitter_on_scala.html).  When you do that, you've got data in a (usually traditional RDBMS) database.  You might keep some or all of that data where it is, or you might move it to alternative datastores that fit your application's requirements.  Either way, to get started with Scala, you need to get at your data with Scala.

[Typesafe's Slick library](http://slick.typesafe.com/) offers a way to get at data stored in a database using a query API that closely matches the way you're used to working with Scala collections.  If you've had to work with either raw SQL or with the likes of Hibernate, just looking at the examples of lifted embedding on the Slick home page is enough to make you sigh with relief. 

But, there is a cost: to get that sexy functional query syntax, you have to write `Table` classes for each of your database tables.  This feels an awful lot like writing those damn Hibernate ORM XML mappings, though not nearly as bad.  Here's an example from the [Slick Getting Started Guide](http://slick.typesafe.com/doc/1.0.1/gettingstarted.html):

	// Definition of the SUPPLIERS table
	object Suppliers extends Table[(Int, String, String, String, String, String)]("SUPPLIERS") {
	  def id = column[Int]("SUP_ID", O.PrimaryKey) // This is the primary key column
	  def name = column[String]("SUP_NAME")
	  def street = column[String]("STREET")
	  def city = column[String]("CITY")
	  def state = column[String]("STATE")
	  def zip = column[String]("ZIP")
	  // Every table needs a * projection with the same type as the table's type parameter
	  def * = id ~ name ~ street ~ city ~ state ~ zip
	}

	// Definition of the COFFEES table
	object Coffees extends Table[(String, Int, Double, Int, Int)]("COFFEES") {
	  def name = column[String]("COF_NAME", O.PrimaryKey)
	  def supID = column[Int]("SUP_ID")
	  def price = column[Double]("PRICE")
	  def sales = column[Int]("SALES")
	  def total = column[Int]("TOTAL")
	  def * = name ~ supID ~ price ~ sales ~ total
	  // A reified foreign key relation that can be navigated to create a join
	  def supplier = foreignKey("SUP_FK", supID, Suppliers)(_.id)
	}
At least it's code and not XML, but it is still tedious, especially if you have a large database.  And, if you're going to be operating with two codebases hitting that schema (perhaps Scala serving the mobile API and PHP serving the website), you're going to have to keep your `Table`s in sync with changes to the schema.

This is a problem that I was facing when I ran across a very cool thing.

####The Slick Migration Library Proof-of-Concept

In case you didn't know it, [your database should be under version control](http://www.codinghorror.com/blog/2008/02/get-your-database-under-version-control.html).  You might be doing this with scripts.  Or, maybe you're using a tool like [Liquibase](http://www.liquibase.org/).  If you're working on a legacy codebase, your schema management may be a database dude at a mysql prompt.

[Christopher Vogt](https://github.com/cvogt) from the [Slick](http://slick.typesafe.com/) team at [Typesafe](http://typesafe.com/) (as in the company, uppercase) recently built a proof-of-concept for a database migration library that will allow Slick users to keep database migrations in code in a sane and type-safe (lowercase, as in safety-with-types) way.  

To those of us who have had to deal with database migration nightmares in the past, this sounds good.  Even though it is just in proof-of-concept stage, it looks even better than it sounds -- see my video below.  You write your database migrations in code where they can be automatically applied, and then the library introspects the resulting database to generate the `Table` code for all of your database tables.  

Wait a minute.  It introspects the database and generates the `Table`s?  Could I use that now, so that I don't have to write all of this code to match my legacy mysql schema?

####My Laziness Triumphs

As it turns out, yes, and so was born [my fork of Christopher's migration proof-of-concept](https://github.com/bhudgeons/migrations).  I made it (theoretically) support any Slick-compatible database and added lots more configuration options.  It turns out that, even though you have to use a funky version of Slick to get the tool running (as seen in the README of the project), the generated code works just fine in the current stable version of Slick (1.0.1).  I even have a configuration option to pump the generated code directly into my project and into the right package.  Now, when the schema changes, I can just re-run the tool to generate the new code, and anything that affects my application will probably show up as soon as I try to compile.

Note: using this as a code generation tool is a temporary thing.  [Amir Shaikhha](https://github.com/amirsh) is adding official Slick code generation to Slick 2.x (see e.g. [this pull request](https://github.com/slick/slick/pull/164)), so code generation will move out of the migration tool altogether.  Something like my tool with my configuration options should exist, but the migrations library should be a library (and thus separated from all of this tool stuff), and I'll be separating my tool (and its configuration) from the library going forward.  ([See the discussion here.](https://github.com/cvogt/migrations/pull/2))

But, in any case, it was useful to me now, so I thought it might be useful to someone else. Even if it doesn't work perfectly, you'll likely be a lot closer with what gets generated than with starting at zero.  Cheers.

*Here are two YouTube videos with demos.  Be sure to switch the player to HD!*

####[Demo One: The Slick Migration Proof of Concept](http://www.youtube.com/watch?v=J9DJ9jjhxn0)




####[Demo Two: Using Slick Code Generation on a MySQL Database](http://www.youtube.com/watch?v=wUZg28hca6s)


