---
title:  "Automate mysql_secure_installation Using Bash"
date:   2019-07-27 09:00:00
permalink: automate-mysql-secure-installation-using-bash
---

## Automate mysql_secure_installation Using Bash

This is a documentation on running a script to install mysql on a Ubuntu 18.04 server, specifically using the AWS EC2 image.

## Project Requirements

I am playing around with Terraform at the moment and would like to automate setting up a staging server that will run a mysql instance inside it along with a Rails application.

And here, I am placing the mysql server in the instance itself instead of relying on a managed database system like AWS RDS due to cost. Since it is not required for such a transient environment, it is more cost-effective to host it in the same server as the Rails application.

A proper way to setup mysql is to run the `mysql_secure_installation` command after installing `mysql` using `apt`.

While the installation of `mysql` packages using bash is straight forward, it is not the case for running the `mysql_secure_installation` as that requires user feedback inputs. It becomes difficult to automate those feedback.

Instead of finding out how to automate giving those feedback inputs, it might be easier to run the commands that run after we give our feedback inputs. First, we need to understand what the `mysql_secure_installation` script is actually doing.

## What Is mysql_secure_installation Actually Doing?

mysql_secure_installation is actually a script that execute some commands using the input we give during the its execution procedure. More details about this is noted down in [this blog post](https://bertvv.github.io/notes-to-self/2015/11/16/automating-mysql_secure_installation/).

We just need to execute the script one by one using pre coded configurations that is the user’s feedback inputs.

## The Script

This is the required script. This is meant to work well with using Rails on a fresh instance of Ubuntu 18.04 AWS EC2 instance.

![The Script](/docs/assets/automate-mysql-secure-installation-using-bash_script.png)

Let me explain what each line does.

Line 3 installs the packages needed for `mysql` to work. the -e flag stands for execute and will run the subsequent string as a `mysql` command.

Line 5 sets the password of the root user to a desired password. The password used in this case is something

Line 7 removes all test users, which is one of the steps in the `mysql_secure_installation` procedure.

Line 9 restricts remote access to this database. This may not be desirable for your case if you are pursuing faster debug and development over security, which is mitigatable for the case of a staging server.

Line 11 creates the current user, that is `ubuntu` for a Ubuntu 18.04 instance provided by AWS EC2, and remove the sudo requirement to work on `mysql`. This is needed for `Rails` to connect to the `mysql` database. A sample `config/database.yml` is shown below.


```yaml
default: &default
  adapter: mysql2
  encoding: utf8mb4
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  socket: /tmp/mysql.sock

staging: 
  <<: *default
  username: ubuntu
  database: staging_db
  password: something
```
