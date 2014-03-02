# Overview

The purpose of this demo app is to show a way of visualizing data using lightweight web technologies.

* A data set is parsed and stored in a database.

* The data is served using a REST API.

* The data is visualized with `D3.js`

The main points of interest in this demo are:

* Dependencies are specified in a cpanfile and can be installed with cpanm.

* There is only one executable. Everything from database setup to importing the data and serving it are done with this executable, which uses `App::Cmd`.

* After parsing, the data is stored in a PostgreSQL or SQLite database using `DBIx::Class`. The DDL was written manually; the schema classes were generated by `dbicdump`, also driven by the executable.

* The data is served by a REST API using `Web::Machine`.

* A PSGI application exposes the REST API and also serves static files like HTML and JavaScript files.

* There is a single HTML file which contains a basic Bootstrap-based UI and an example visualization using `D3.js`.

* JavaScript dependencies are managed by `Bower`.

All the pieces work together in a lightweight way. No dynamic HTML is generated on the server-side.

# Data

This demo uses [historical S&P500 stock data](http://pages.swcp.com/stocks/#historical%20data). See `data/sp500hst.txt`.

# Installation

You can run this demo from its git working directory. There is no Makefile.PL because it is not intended to be installed.

Dependencies are kept in `cpanfile`, so you can install them with:

    cpanm --installdeps .

JavaScript dependencies are specified in `bower.json`. Install them with:

    bower install

# Command line interface

The app is controlled with a single executable, `bin/stocks`. It uses `App::Cmd` to provide various commands. Just running the command without options will display a list of available commands. `bin/stocks help <command>` will display more detailed information about a command.

# Configuration

The demo is configured using an INI file. See `etc/basic.conf` for an example.

Configuration keys are strictly checked using `Data::Domain`. The following keys are recognized:

* `dbdriver`: Name of the database driver. Default: Pg.

* `dbname`: Name of the database. Mandatory.

* `dbhost`, `dbpost`, `dbuser`, `dbpass`: Options extra fields for the database connect string.

* `ddl_file`: Path to the DDL file. It can be absolute or relative to the working directory root (i.e., where the `.git` directory is). Mandatory.

* `htdocs`: Path to the directory that holds static Web resources like HTML files or CSS files. It can be absolute or relative to the working directory root. Mandatory.

To use a configuration file, you can either specify it as an option to a command or use an environment variable.

    ./bin/stocks <command> -c path/to/config

or

    export STOCKS_CONF=path/to/config
    ./bin/stocks <command>

# Workflow

## Database setup

The `dbsetup` command drops the current database, creates it again and imports the DDL. You need to use the `--yesreally` or `-y` option to make this command run.

    ./bin/stocks dbsetup -y

## Data import

The `import` command takes one or more paths to data files. It reads each file, parses it and inserts corresponding rows into the database.

    ./bin/stocks import data/sp500hst.txt

## Web server

The `web` command runs a Plack-based web server that serves both the REST API as well as static content.

    ./bin/stocks web

As you can see in `Stocks::Web`, the Javascript libraries managed by `bower` are mounted on `/components`, and the `stocks` REST resource is mounted on `/api/v1/stocks`.

The `stocks` resource is implemented in `Stocks::Web::Resource::Stocks` using `Web::Machine`. There are CSV and JSON representations, which you can request by setting the `Accept` header in the request. If you don't use any query parameters, you will get the full data set.

    curl -H 'Accept: text/csv' "http://localhost:5000/api/v1/stocks"

However, often this is inefficient; you will want only part of the data, possibly aggregated. You can specify which fields you want by listing them in a comma-separated list in the `fields` query parameter. Each field can be either just the field name or it can have a colon-prepended operation. The operation can be `groupby` or `sum`, using the standard SQL semantics. For example:

    curl -H 'Accept: text/csv' "http://localhost:5000/api/v1/stocks?fields=groupby:ticker,sum:volume"

You can also filter the data by using query parameters corresponding to the column names - `day` or `ticker`. (It doesn't really make sense to filter by `open`, `high`, `low`, `close` or `volume`.)  The result will then only contain those rows that have the specified value. So using those query parameters will effectively build up a `WHERE` clause.

    curl -H 'Accept: text/csv' "http://localhost:5000/api/v1/stocks?ticker=AAPL"

## Visualization

When the web server is running, you can open `http://localhost:5000/` in your browser. This will load `index.html`, which contains a basic Bootstrap UI and a simple visualization using D3.js that uses the REST API to load data.

## Schema generation

The `genschema` command uses [`dbicdump`](https://metacpan.org/pod/distribution/DBIx-Class-Schema-Loader/script/dbicdump) to regenerate the schema classes. It is only needed during development and is only available if the `STOCKS_DEV` environment variable is set.

    STOCKS_DEV=1 ./bin/stocks genschema
