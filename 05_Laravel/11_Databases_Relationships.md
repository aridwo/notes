## Reference:

+ <http://daylerees.com/codebright/eloquent-relationships>
+ <http://laravel.com/docs/eloquent#relationships>
+ <http://laravel.com/docs/schema#foreign-keys>


---

A web application often contains multiple database tables, and those tables often relate to one another in some way. 

For example, imagine the `Foobooks` example expanded to include a `authors`, and `tags` table, in addition to the original `books` table.

<img src='http://making-the-internet.s3.amazonaws.com/laravel-foobooks-final-db-schema@2x.png' class='' style='max-width:1264px; width:100%' alt=''>

The relationship amongst these tables could be described as follows:

__[One to One](http://laravel.com/docs/eloquent#one-to-one)__

+ Each book has one author
+ Book *belongs to* Author

__[One to Many](http://laravel.com/docs/eloquent#one-to-many)__

+ Each author may have many books
+ Author *has many* Books

__[Many to Many](http://laravel.com/docs/eloquent#many-to-many)__

+ Each book may have many tags
+ Each tag may belong to many books
+ Tags *belongs to many* Books
+ Books *belongs to many* Tags

The relationships between tables are created using **foreign keys** (FK).

For example, the `books` table has a foreign key field `author_id` which connects it to the `authors` table. 

*Many to Many* relationships require an additional table, called a **pivot table** to keep track of the connections.



## Pivot Tables

+ Table that links the two tables together using foreign keys.
+ Also known as: &ldquo;lookup table&rdquo;, &ldquo;join table&rdquo;. 
+ A primary key is not needed on a pivot table.
+ Naming: Use singular version of the name of the two tables it's joining, separate with underscore, list in alphabetical order. Ex: If you're joining the `books` table with the `tags` table, the resulting pivot table name would be `book_tag`.


## Schema for relationships

When building your migrations, there are two things you'll need to know (in addition to what you already know about schemas) to set up your relationships:

__First:__ If a column is going to be a FK that connects to an auto-incrementing column on another table, it must be `unsigned`.

	$table->integer('author_id')->unsigned();

__Second:__ Here's the syntax for defining a FK:

	$table->foreign('author_id')->references('id')->on('authors'); 

[Here's the Schema building for all of the Foobooks tables](https://gist.github.com/susanBuck/992b1323f6cc0f68427d).



## Identifying Relationships in Models

Relationships amongst tables need to be defined in their corresponding Models via relationship methods.

For example, the Author class should have a method called `book()` which returns an Eloquent relationship [hasMany](http://devdocs.io/laravel/api/4.2/illuminate/database/eloquent/model#method_hasMany):

	class Author extends Eloquent { 
	
   		public function book() {
			return $this->hasMany('Book');
   		}
   	}
   	
On the flip side, the Book class should have a method called `author()` which returns an Eloquent relationship [belongsTo](http://devdocs.io/laravel/api/4.2/illuminate/database/eloquent/model#method_belongsTo):

	class Book extends Eloquent { 

		public function author() {
			return $this->belongsTo('Author');
		}
	}
	
The Tag model should indicate that a tag [belongsToMany](http://devdocs.io/laravel/api/4.2/illuminate/database/eloquent/model#method_belongsToMany) books:

	class Tag extends Eloquent { 
		
	    public function books() {
		    return $this->belongsToMany('Book');
	    }
	    
	}
	
Likewise, the Book model should indicate that a tag belongsToMany book [belongsToMany](http://devdocs.io/laravel/api/4.2/illuminate/database/eloquent/model#method_belongsToMany) tags:

	class Book extends Eloquent { 
		
	    public function author() {
		    return $this->belongsTo('Author');
	    }
	    
	    public function tags() {
	        return $this->belongsToMany('Tag');
	    }
	    
	}

Once these relationship methods are created, you can put them to use...


## Associate an author with a book

Every book should be associated with an author...

For example, you could create a new author:

	$author = new Author;
	$author->name = 'F. Scott Fiztgerald';
	$author->birth_date = '1896-09-24';
	$author->save();
	
And then you could creat a new book, associating it with the author:

	$book = new Book;
	$book->title = 'The Great Gatsby';
	$book->published = 1925;
	$book->cover = 'http://img2.imagesbn.com/p/9780743273565_p0_v4_s114x166.JPG';
	$book->purchase_link = 'http://www.barnesandnoble.com/w/the-great-gatsby-francis-scott-fitzgerald/1116668135?ean=9780743273565';
	$book->author()->associate($author);
	$book->save();
	
The key line in the above statement is this one:

	$book->author()->associate($author);

Note how this line is called *before* the `save()` method. This is because `associate` is setting the `author_id` field on the books row; that setting should happen before the row is created.


## Attach a tag to a book

Assuming you've created a tag...

	$tag = new Tag;
	$tag->name = 'novel';
	$tag->save();
	
You can now create a book and attach a tag to it:

	$book = new Book;
	$book->title = 'The Great Gatsby';
	$book->published = 1925;
	$book->cover = 'http://img2.imagesbn.com/p/9780743273565_p0_v4_s114x166.JPG';
	$book->purchase_link = 'http://www.barnesandnoble.com/w/the-great-gatsby-francis-scott-fitzgerald/1116668135?ean=9780743273565';
	$book->author()->associate($fitzgerald); # <-----
	$book->save();
	$book->tags()->attach($tag); 

The key line in the above statement is this one:

	$book->tags()->attach($tag); 

Unlike `associate()`, `attach()` needs to happen *after* the `save()` method. This is because it's creating a row in the `book_tag` pivot table and it needs a `book_id` to do so. The `book_id` won't exist until after the book as been added.


## Querying with relationships

Once your Models have been programmed with relationships, it's easy join data amongst multiple tables.

For example, if you're querying for all books, you may want to join in the related author data. This can be done via the [with](http://devdocs.io/laravel/api/4.2/illuminate/database/eloquent/model#method_with) method and is referred to as **eager loading**:

	$books = Book::with('author')->get(); 

	foreach($books as $book) {
		echo $book->author->name.' wrote '.$book->title.'<br>';
	}
	
Same idea, but with tags:

	$books = Book::with('tags')->get(); 

	foreach($books as $book) {
		
		echo $book->title."<br>";
		foreach($book->tags as $tag) {
			echo $tag->name.", ";
		}
		
		echo "<br><br>";
		
	}

Or maybe you want the author *and* tags:

	$books = Book::with('tags','author')->get(); 

	foreach($books as $book) {
		
		echo $book->title.' by '.$book->author->name.'<br>';
		foreach($book->tags as $tag) {
			echo $tag->name.", ";
		}
		
		echo "<br><br>";
		
	}



## Tips:

__Tip 1__
If you receive an error like this:

	General error: 1005 Can't create table 'database-name.#sql-13993_88' 
	(errno: 150) (SQL: alter table `books` add constraint books_author_id_foreign foreign key 
	(`author_id`) references `authors` (`id`))`                                                                      

There may be two possible causes:

__Cause 1)__ When building relationships, it's important that you create tables in a logical order. For example: if the table `books` will have a FK connecting it to `authors`, then the `authors` table should exist before attempting to create that FK.


__Cause 2)__ When creating a column that will contain a FK relating it to another table, make sure that column is unsigned. Example:

	# Important! FK has to be unsigned because the PK it will reference is auto-incrementing
	$table->integer('author_id')->unsigned(); 
	

__Tip 2__

As you're playing around with table relationships, you'll likely generate a lot of junk data you'll want to quickly clean out. Wiping the data from tables becomes tricky, though, once you add foreign keys, because your database will want to prevent you from deleting data that is potentially connected to other existing data.

To get around this, you can disable foreign key checks.

Here's a quick and dirty route to clear a bunch of tables, which you can adapt to match your own tables:

	Route::get('/truncate', function() {
		
		# Clear the tables to a blank slate
		DB::statement('SET FOREIGN_KEY_CHECKS=0'); # Disable FK constraints so that all rows can be deleted, even if there's an associated FK
		DB::statement('TRUNCATE books');
		DB::statement('TRUNCATE authors');
		DB::statement('TRUNCATE tags');
		DB::statement('TRUNCATE book_tag');
	}
	
Obviously, you'll want to make sure this route is removed from your codebase before launching your live site; you don't want anyone to accidentally stumble on this route and wipe out your data.