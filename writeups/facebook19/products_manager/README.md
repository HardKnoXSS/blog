The site itself is pretty barebones.

* index.php - See the top 5 products from FB
* add.php   - Add your own product to a massive DB. Requires a name, secret, and description
* view.php  - View the inner data of a particular product. Use a secret to pul lthe description.

A major hint was found within db.php

```php
/*
CREATE TABLE products (
  name char(64),
  secret char(64),
  description varchar(250)
);

INSERT INTO products VALUES('facebook', sha256(....), 'FLAG_HERE');
INSERT INTO products VALUES('messenger', sha256(....), ....);
INSERT INTO products VALUES('instagram', sha256(....), ....);
INSERT INTO products VALUES('whatsapp', sha256(....), ....);
INSERT INTO products VALUES('oculus-rift', sha256(....), ....);
*/
```

First, the flag is clearly discovered within the description field of the 
facebook product. 

Second, is a more subtle hint that missed by me but caught by Joseph, the
table properties for `char` and `varchar` variables are different.

A more detailed description comes from MYSQL's [webpage](https://dev.mysql.com/doc/refman/5.7/en/char.html):

    The length of a CHAR column is fixed to the length that you declare when you create the table. The length can be any value from 0 to 255. When CHAR values are stored, they are right-padded with spaces to the specified length. When CHAR values are retrieved, trailing spaces are removed unless the PAD_CHAR_TO_FULL_LENGTH SQL mode is enabled.

Well, if they're right-padded with spaces, what happens if there's a collision between `facebook` and `facebook  `. To test this, we'll add the new `facebook ` product with a secret that meets the minimum criteria `Aa00000000`. Well both products are stored in the database, but our query will return the *first* value. This is our target: `facebook`. Which returns the key.

[web100.png](images/web100.png)
