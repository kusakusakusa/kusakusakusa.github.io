## Automate mysql_secure_installation Using Bash

This is a documentation on running a script to install mysql on a Ubuntu 18.04 server, specifically using the AWS EC2 image.

Project Requirements
I am playing around with Terraform at the moment and would like to automate setting up a staging server that will run a mysql instance inside it along with a Rails application.

And here, I am placing the mysql server in the instance itself instead of relying on a managed database system like AWS RDS due to cost. Since it is not required for such a transient environment, it is more cost-effective to host it in the same server as the Rails application.

A proper way to setup mysql is to run the mysql_secure_installation command after installing mysql using apt.

While installation of mysql packages using bash is straight forward, it is not the case for running the mysql_secure_installation as that requires user feedback inputs. It becomes difficult to automate those feedback.

Instead of finding out how to automate giving those feedback inputs, it might be easier to run the commands that run after we give our feedback inputs. First, we need to understand what the mysql_secure_installation script is actually doing.


