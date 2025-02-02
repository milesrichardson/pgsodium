# pgsodium

Postgres extension for [libsodium](https://download.libsodium.org/doc/).

## Installation

Tested with Postgres 11.5, 10.x and 9.6.  Requires libsodium >=
1.0.18.  In addition to the libsodium library and it's development
headers, you may also need the postgres header files typically in the
'-dev' packages to build the extension.

Clone the repo and run 'sudo make install'.

pgTAP tests can be run with 'sudo -u postgres pg_prove test.sql' or
they can be run in a self-contained Docker test image based on
postgres 11.5.  Run `./test.sh` if you have docker installed.

## Usage

Most of the libsodium API is available.  Keys that are generated in
pairs are returned as a record type, for example:

```
postgres=# SELECT * FROM crypto_box_new_keypair();
                               public                               |                               secret
--------------------------------------------------------------------+--------------------------------------------------------------------
 \xa55f5d40b814ae4a5c7e170cd6dc0493305e3872290741d3be24a1b2f508ab31 | \x4a0d2036e4829b2da172fea575a568a74a9740e86a7fc4195fe34c6dcac99976
(1 row)
```

pgsodium is careful to use memory cleanup callbacks to zero out all
allocated memory used by the extension on freeing.  In general it is a
bad idea to store secrets in the database itself, although this can
still be done carefully it has a higher risk.

Other options include injecting the key from an external secret
storage into each session with `SET LOCAL` statements.  If the
database is hacked or stolen, the keys will not be available to the
attacker.  This approach has problems in that if your `log_statements`
is set to `all` the `SET LOCAL` statement will log the secrets so be
careful to disable logging while injecting the key, as shown below.

Here's an example usage from the test.sql that uses `psql` client
commands to encrypt data.  Note in this example, no secrets are stored
in the db, but they are interpolated into the sql that is sent to the
server, so it's possible they can show up in logs as well.


    -- Generate a boxnonce, and public and secret keypairs for bob and alice
    SELECT crypto_box_noncegen() boxnonce \gset
    SELECT public, secret FROM crypto_box_new_keypair() \gset bob_
    SELECT public, secret FROM crypto_box_new_keypair() \gset alice_

    -- Alice encrypts the box for bob using her secret key and his public key
    SELECT crypto_box('bob is your uncle', :'boxnonce', :'bob_public', :'alice_secret') box \gset

    -- Bob decrypts the box using his secret key and Alice's public key
    SELECT is(crypto_box_open(:'box', :'boxnonce', :'alice_public', :'bob_secret'),
              'bob is your uncle', 'crypto_box_open');


A more paranoid approach disables logging and injects the keys into
local variables and then resets the logging:

    -- SET LOCAL must be done in a transaction block
    BEGIN;

    -- Generate a boxnonce, and public and secret keypairs for bob and alice
    SELECT crypto_box_noncegen() boxnonce \gset
    SELECT public, secret FROM crypto_box_new_keypair() \gset bob_
    SELECT public, secret FROM crypto_box_new_keypair() \gset alice_

    -- Turn off logging and inject secrets into session with set local, then resume logging
    SET LOCAL log_statement = 'none';
    SET LOCAL app.bob_secret = :'bob_secret';
    SET LOCAL app.alice_secret = :'alice_secret';
    RESET log_statement;

    -- Alice encrypts the box for bob using her secret key and his public key
    SELECT crypto_box('bob is your uncle', :'boxnonce', :'bob_public', current_setting('app.alice_secret')) box \gset

    -- Bob decrypts the box using his secret key and Alice's public key
    SELECT is(crypto_box_open(:'box', :'boxnonce', :'alice_public', current_setting('app.bob_secret')),
              'bob is your uncle', 'crypto_box_open');

    COMMIT;
