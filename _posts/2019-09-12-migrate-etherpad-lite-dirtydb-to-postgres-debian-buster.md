---
layout: post
title: How to migrate etherpad-lite dirty.db to postgres on debian 10 (buster)
category: tutorial
tags:
  - debian
  - etherpad-lite
  - postgres
---

The first part of this tutorial comes from [etherpad-lite](https://github.com/ether/etherpad-lite/wiki/How-to-use-Etherpad-Lite-with-PostgreSQL) wiki but the wiki content is outdated. Here is a recent version of the procedure, with some additions.

Note: This is a repost of [my tutorial](https://framagit.org/snippets/3851) with some minor corrections (typos).

1. Start by installing postgres:

	``` terminal
	# apt install postgresql
	```

2. We create an user etherpad-lite because postgres needs a pam user to be able to use unix sockets (we don't want to use tcp sockets):

	``` terminal
	# adduser etherpad-lite --no-create-home
	```

3. We create an user on postgres:

	``` terminal
	# sudo -u postgres sh -c 'createuser -d etherpad-lite && createdb -O etherpad-lite etherpad-lite'
	```

4. Add a password (here, `yourpassword`) to our new postgres user:

	``` terminal
	# sudo -u postgres psql
	psql=# alter user etherpad-lite with encrypted password 'yourpassword';
	```

	You can type `\q` to exit psql prompt.

5. Assuming you have installed your postgres on the same machine running etherpad-lite, add this to your `etherpad-lite/settings.json` (replace `yourpassword` with the password you created)

	~~~ json-doc
	{
	...

	  "dbType" : "postgres",
	  "dbSettings" : {
	      "user"    : "etherpad-lite",
	      "host"    : "localhost",
	      "password": "yourpassword", // replace by your actual password
	      "database": "etherpad-lite",
	      "charset" : "utf8mb4"
	  }

	...
	}
	~~~


	At this point, you should be able to run etherpad-lite using postgres without any error. In next steps, we will see how to migrate your old `dirty.db` to postgres.

6. We will run a perl script. To be able to run the script, install `libdbd-pg-perl`:

	``` terminal
	# apt install libdbd-pg-perl
	```

7. `dirty.db` is a json file. You can run the following perl script based on [a script from etherpad-lite wiki](https://github.com/ether/etherpad-lite/wiki/Manipulating-the-database). We made some modifications in order to be usable with postgres. You have to change `yourpassword` and `/path/to/your/dirty.db` to real values. Make sure your etherpad-lite is stopped before running the script.

	~~~ perl
	#!/usr/bin/env perl
	use strict;
	use DBI;

	my $dbh = DBI->connect("DBI:Pg:database=etherpad-lite;host=localhost", "etherpad-lite", "yourpassword",) or die;
	$dbh->prepare("TRUNCATE TABLE store")->execute();

	open(F,"/path/to/your/dirty.db") or die;

	while (<F>) {
	    if (m|\{\"key\":\"(.*)\",\"val\":(.*)\}|) {
	    my ($k,$v) = ($1,$2);
	    my $sth = $dbh->prepare("SELECT key FROM store WHERE key = ?") or die;
	    $sth->execute($k) or die;
	    my @a = $sth->fetchrow();
	    if ($a[0]) {
		$sth = $dbh->prepare("UPDATE store set value = ? WHERE key = ?") or die;
		$sth->execute($v,$k) or die;
	    } else {
		$sth = $dbh->prepare("INSERT INTO store (key,value) VALUES (?,?)") or die;
		$sth->execute($k,$v) or die;
	    }
	    } else {
	    die "Err!\n";
	    }
	}
	close F;
	~~~

	Now, your postgres is populated with your old `dirty.db` content. You can start your etherpad-lite again and check your previously created pads are online.
