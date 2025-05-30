Discogs-xml2db is a python program for importing discogs data dumps into several databases.

This *speedup* code is a rewrite of the original discogs-xml2db.
It is several times faster and based on a [branch by RedApple](https://github.com/redapple/discogs-xml2db).

Currently *speedup* supports MySQL and Postgresql as target databases.
Instructions for importing into MongoDB and CouchDB are provided below, though these are untested. Let us know how it goes!


### Comparison to classic discogs-xml2db

*speedup* is many times faster than *classic* because it uses a different approach.
The discogs xml dumps are first converted into one csv file per database table.
These csv files are then imported into the different target databases.
This is different from *classic* discogs-xml2db which loads records into the databases one by one while parsing the xml file, waiting on the database after every row.

*speedup* requires less disk space than *classic* as it can work while the dump files are still compressed. While the uncompressed dumps for May 2020 take up 57GB of space the compressed dumps are only 8.8GB. The dumps can be deleted after converting them to compressed CSV files (6.1GB).

As many databases can import CSV files out of the box it should be easy to add support for more databases to discogs-xml2db *speedup* in the future.


### Requirements
discogs-xml2db *speedup* requires python3 and some python modules.
Additionally the bash shell is used for automating some tasks.
Importing to some databases may require additional dependencies, see the documentation for your target database below.

```shell
# Install the required python packages using pip.
$ sudo pip3 install -r speedup/requirements.txt
```

While Python 3.4 and onwards ship with pip by default you may need to install it if using older versions. On Ubuntu or Debian run `$ sudo apt-get install python3-pip`. Installation instruction for other platforms can be found in the [pip documentation](https://pip.pypa.io/en/stable/installing/).

### Downloading discogs dumps
Download the latest dump files from discogs manually from [discogs](https://data.discogs.com/) or run `get_latest_dumps.sh`.

To check the files' integrity download the appropriate checksum file from [https://data.discogs.com/](https://data.discogs.com/), place it in the same directory as the dumps and compare the checksums.
```shell
$ sha256sum -c discogs_*_CHECKSUM.txt
```


### Converting dumps to CSV
Run `exporter.py` to convert the dump files to csv.

```shell
$ python3 speedup/exporter.py --bz2 --apicounts --export artist --export label --export master --export release dump-dir csvdir
```

`exporter.py` takes the folowing arguments:
- `--export`: the types of dump files to export
- `--bz2`: Compress output files using bz2 compression library.
- `--limit=<lines>`: Limit export to some number of entities
- `--apicounts`: Make progress report more accurate by getting total amounts from Discogs API.

The exporter provides progress information in real time:
```
Processing      labels:  99%|█████████████████████████████████████████▊| 1523623/1531339 [01:41<00:00, 14979.04labels/s]
Processing     artists: 100%|████████████████████████████████████████▊| 6861991/6894139 [09:02<00:02, 12652.23artists/s]
Processing    releases:  78%|█████████████████████████████▌        | 9757740/12560177 [2:02:15<36:29, 1279.82releases/s]
```
The total amount and percentages might be off a bit as the exact amount is not known while reading the file.
Specifying `--apicounts` will provide more accurate predictions by getting the latest amounts from the Discogs API.


### Importing

If `pv` is available it will be used to display progress during import. To install it run `$ sudo apt-get install pv` on Ubuntu and Debian or check the [installation instructions for other platforms](http://www.ivarch.com/programs/pv.shtml). Example output if using `pv`:
``` shell
$ mysql/importcsv.sh 2020-05-01/csv/*
artist_alias.csv.bz2: 12,5MiB 0:00:03 [3,75MiB/s] [===================================>] 100%
artist.csv.bz2:  121MiB 0:00:29 [4,09MiB/s] [=========================================>] 100%
artist_image.csv.bz2:  7,3MiB 0:00:01 [3,72MiB/s] [===================================>] 100%
artist_namevariation.csv.bz2: 2,84MiB 0:00:01 [2,76MiB/s] [==>                          ] 12% ETA 0:00:07
```

#### Importing into Postgresql

```shell
# install PostgreSQL libraries (might be required for next step)
$ sudo apt-get install libpq-dev

# install the PostgreSQL package for python
$ sudo pip3 install -r postgresql/requirements.txt

# Configure PostgreSQL username, password, database, ...
$ nano postgresql/postgresql.conf

# Create database tables
$ python3 postgresql/psql.py < postgresql/sql/CreateTables.sql

# Import CSV files
$ python3 postgresql/importcsv.py /csvdir/*

# Configure primary keys and contraints, build indexes
python3 postgresql/psql.py < postgresql/sql/CreatePrimaryKeys.sql
python3 postgresql/psql.py < postgresql/sql/CreateFKConstraints.sql
python3 postgresql/psql.py < postgresql/sql/CreateIndexes.sql
```

#### Importing into Mysql

```shell
# Configure MySQL username, password, database, ...
$ nano mysql/mysql.conf

# Create database tables
$ mysql/exec_sql.sh < mysql/CreateTables.sql

# Import CSV files
$ mysql/importcsv.sh /csvdir/*

# Configure primary keys and build indexes
$ mysql/exec_sql.sh < mysql/AssignPrimaryKeys.sql
```

#### Importing into MongoDB
The CSV files can be imported into MongoDB using [mongoimport](https://docs.mongodb.com/manual/reference/program/mongoimport/).

```shell
$ mongoimport --db=discogs --collection=releases --type=csv --headerline --file=release.csv
```

#### Importing into CouchDB
CouchDB only supports importing JSON files. [`couchimport`](https://github.com/glynnbird/couchimport) can be used to convert the CSV files to JSON and import them into CouchDB, as explained in [this tutorial](https://medium.com/codait/simple-csv-import-for-couchdb-71616200b095).


### Database schema changes

The database schema was changed to be more consistent and normalize some more data.
The folowing things changed compared to *classic* `discogs-xml2db`:

- renamed table: `releases_labels` => `release_label`
- renamed table: `releases_formats` => `release_format`
- renamed table: `releases_artists` => `release_artist`
- renamed table: `tracks_artists` => `release_track_artist`
- renamed table: `track` => `release_track`
- renamed column: `release_artists.join_relation` => `release_artist.join_string`
- renamed column: `release_track_artist.join_relation` => `release_track_artist.join_string`
- renamed column: `release_format.format_name` => `release_format.name`
- renamed column: `label.contactinfo` => `label.contact_info`
- renamed column: `label.parent_label` => `label.parent_name`
- added: `label` has new `parent_id` field
- added: `release_label` has extra fields
- moved: `aliases` now in `artist_alias` table
- moved: `tracks_extra_artists` now in `track_artist` table with extra flag
- moved: `releases_extra_artists` now in `release_track_artist` table with extra flag
- moved: `release.genres` now in own `release_genre` table
- moved: `release.styles` now in own `release_style` table
- moved: `release.barcode` now in `release_identifier` table
- moved: `artist.anv` fields now in `artist_namevariation` table
- moved: `artist.url` fields now in `artist_url` table
- removed: `release_format.position` no longer exists but can use id field to preserve order when release has multiple formats.
- `release_track_artist` now use `tmp_track_id` to match to `tmp_track` in `release_track`
