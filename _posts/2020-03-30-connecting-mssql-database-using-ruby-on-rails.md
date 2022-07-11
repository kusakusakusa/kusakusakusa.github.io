---
title:  "Connecting MSSQL Database Using Ruby On Rails"
date:   2020-03-30 09:00:00
permalink: connecting-mssql-database-using-ruby-on-rails
---

This is a documentation on how to connect to a MSSQL database in a Rails application. We will use [FreeTDS])(https://www.freetds.org/) as the main toolkit to establish the connection.

## Motivation

I came across a gig that requires me to connect to a MSSQL database to extract the data via the application that I was building in Ruby On Rails. I spend quite some time experimenting  and playing with it before I can manage to get it to work.

It will be good to document my steps and reasons in case I come across another such request and my memory fails me.

## Installation

While Ruby on Rails has a [gem](https://github.com/rails-sqlserver/tiny_tds) that serves as a wrapper around the FreeTDS library of files, it requires the FreeTDS binaries to be installed natively on the machine that is running the application.

This presents a number of challenges. First, the local machine used may be different for different users. Second, the operating system used in the servers and local machine may be different too.

For my case, I use macOS for my development work, and the Amazon flavored linux for my staging and production sites.

### Installing FreeTDS on macOS

The steps listed here follows this guide closely.

First, install using these files locally in the kernel using [homebrew](https://brew.sh/).

```bash
brew update
brew install unixodbc freetds
```

[ODBC](https://en.wikipedia.org/wiki/Open_Database_Connectivity) is an API that is meant for database access across different platforms. unixODBC is the driver manager that allows unix systems to connect to ODBC-capable databases.

MSSQL is one such database. However, while it uses ODBC for connection, it uses the [TDS](https://en.wikipedia.org/wiki/Tabular_Data_Stream) protocol on the application layer for communication. Hence, a ODBC driver alone is insufficient for the machine to process the data in the database. This is where FreeTDS comes in.

FreeTDS is a set of libraries that will do the translation and allow our application to connect to the database and retrieve the data.

### Installing TDS on Amazon Linux

Credits to [this answer on stackoverflow](https://stackoverflow.com/questions/48770235/ms-sql-driver-with-amazon-linux-ami-an-python/50420788#50420788). He even gave the steps required to install the packages via Elastic Beanstalk, which is convenient for me as I also use [Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) for deployment.

```bash
[ ! -e /home/ec2-user/freetds-1.00.86.tar.gz ] && \
wget -nc ftp://ftp.freetds.org/pub/freetds/stable/freetds-1.00.86.tar.gz -O /home/ec2-user/freetds-1.00.86.tar.gz || \
true
```
The first section of the code that is enclosed within a pair of square bracket is a unix command to check the existence of the zip file, which contains the necessary libraries, in the home path of the server. In the Amazon Linux system, the home path is /home/ec2-user by default. Adjust accordingly if you are installing in a linux local machine.

Should the file exist, the subsequent command to download the file will not be executed due to the logical && operation.

The last || operation with a true ensures the command returns a true, and the whole Elastic Beanstalk process will continue even if the file already exist. Of course, this step is not necessary if we are installing the libraries manually on our local linux machine.

```bash
[ ! -e /home/ec2-user/freetds-1.00.86 ] && \
tar -xvf /home/ec2-user/freetds-1.00.86.tar.gz -C /home/ec2-user/ || \
true
```

Similarly this step check for the presence of the unzipped file to prevent repeated and unnecessary unzipping of the compressed library.

```bash
[ ! -e /usr/local/etc/freetds.conf ] && cd /home/ec2-user/freetds-1.00.86 && \
sudo ./configure --prefix=/usr/local --with-tdsver=7.4 || \
true

[ ! -e /usr/local/etc/freetds.conf ] && \
( cd /home/ec2-user/freetds-1.00.86 && sudo make && sudo make install ) || \
true
```

The next 2 commands set up the configurations for FreeTDS and start finally installing its libraries. Upon installation, the config file freetds.conf will be produced, which explains the checks against its existence to prevent duplicate installation operations.

## Application in Ruby on Rails

With the FreeTDS libraries installed in the kernel, we can look at how to use the tiny_tds gem to communicate with the MSSQL database. After installing it via bundler, we can sue the following commands to connect.

``` ruby
client = TinyTds::Client.new(
  username: Rails.application.credentials.dig(Rails.env.to_sym, :deltek, :username),
  password: Rails.application.credentials.dig(Rails.env.to_sym, :deltek, :password),
  host: Rails.application.credentials.dig(Rails.env.to_sym, :deltek, :host),
  port: Rails.application.credentials.dig(Rails.env.to_sym, :deltek, :port),
  database: Rails.application.credentials.dig(Rails.env.to_sym, :deltek, :database)
)
```

Following the new practice of using credential file to store secrets, I have stored all the database credentials in the encrypted credential.yml.enc file.

``` ruby
client.execute("
  SET ANSI_WARNINGS ON;
  SET ANSI_PADDING ON;
  SET ANSI_NULLS ON;
  SET QUOTED_IDENTIFIER ON;
  SET ANSI_NULL_DFLT_ON ON;
  SET CONCAT_NULL_YIELDS_NULL ON;
  SELECT @@OPTIONS;
").each
```

This next snippet sets the settings of the connection. I would not pretend to understand the reasons for the settings made here. However, this is the final settings that worked for me to make the subsequent queries to the database tables. I came to this final configurations after googling around for the different errors that were thrown at me while getting TDS to work.

``` ruby
result = client.execute("SELECT TOP 1 * FROM SOME_TABLE").each
```

This is an example of executing a query in SQL language. The result variable will be an array of hashes, where each hash represent 1 row of record.

``` ruby
result = client.execute("SELECT TOP 1 * FROM SOME_TABLE").each
```

Last but not least, make sure to close the client’s connection. This is not active record that “automagically” does that for you.

