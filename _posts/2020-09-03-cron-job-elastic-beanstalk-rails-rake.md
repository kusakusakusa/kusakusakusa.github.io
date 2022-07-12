---
title:  "Cron Job, Elastic Beanstalk, Rails Rake"
date:   2020-07-13 09:00:00
permalink: cron-job-elastic-beanstalk-rails-rake.md
---

I ran into some issues lately where the the cron job is not running in my Rails deployed on Amazon Elastic Beanstalk. After much debugging, I found that it was because the environment variables are not loaded when the rake tasks were run.

This caused errors like bundler not installed or XX gems not found to appear all over my logs to frustrate the hell out of my weekends.

Took a deep dive and I found the solution. Or should I say the reason, because it remains unsolved and I had to take inspiration from an ancient war tactic to side step this issue. Before we get to the pseudo-solution, let’s find out the reason. And the main reason is…

> Long story short: Rake tasks on cron job with Rails on Elastic Beanstalk will not work with Amazon Linux 2.

This is something I wished someone would have told me before I decided to upgrade my stack to the new version.

According to this [pretty recent issue on Github](https://github.com/awsdocs/elastic-beanstalk-samples/issues/111), the Beanstalk team in AWS is aware that environment variables are not retrievable on the AL2 platforms like how it was on the old AL1.

The issue mentioned the fact that environment variables in the Beanstalk environment settings did not load, but according to my experiences, the environment variables on the system’s level are not loaded either.

I’m talking about $GEM_PATH, $BUNDLE_PATH etc. These are needed for ruby and rails, in particular, and the bundler to know where the gems are to run its operations and avoid the cursed bundle: command not found or gem not found errors.

I’m not the best candidate to debug a solution to fix it on AL2 platforms. Furthermore, by the time I can miraculously figure how to load the variables, Amazon would probably have already solved it.

So it’s much wiser to switch back to AL1. Henceforth, I will cover the method to run rails rake tasks via cron jobs in elastic beanstalk.

And yes this solution is indeed the [last war tactic from Sun Tzu Art Of War](https://en.wikipedia.org/wiki/The_Art_of_War).

These are the files you need.

# .ebextensions/01_cron.config
container_commands:
  01_remove_cron_jobs:
    command: "rm /etc/cron.d/cron_jobs || exit 0"
  02_add_cron_jobs:
    command: "cat .ebextensions/cron_tasks.txt > /etc/cron.d/cron_jobs && chmod 644 /etc/cron.d/cron_jobs"
    leader_only: true
# .ebextensions/cron_tasks.txt
* * * * * root bash -l -c "su -m webapp; cd /var/app/current && rake test"

# Take NOTE it ends with a new line!!
The commands of the 01_cron.config file will be run during deployment.

The first command, 01_remove_ctron_jobs, will remove all cron jobs. Should it fail, it will return a success status code to prevent the deployment procedure from stopping. You can also use [ignoreErrors](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-ec2.html) flag to achieve the same thing.

Why do we need to remove the cron jobs each time we deploy? Great question. We will get to that in a bit.

The second command 02_add_cron_jobs will copy the cron job definitions located inside the cron_tasks.txt file into another file. This other file will be located in /etc/cron.d directory, where the [cron daemon will pick up files](https://www.cyberciti.biz/faq/how-do-i-add-jobs-to-cron-under-linux-or-unix-oses/) from and run the cron jobs as instructed.

The cron job basically runs the rake task as the webapp user, which has the knowledge of the necessary environment variables to execute rails command related to your application.

Now comes a very important point!

Make sure the last line of cron job end with a new line.

The definition of a cron job will be deemed invalid if it does not end with a line break. Hence, a common error occurs where the last cron job did not run due to this reason. I had this error for breakfast last Sunday.


Lastly, the [leader_only flag will be set to ensure your cron job will only run on a single leader instance](https://docs.aws.amazon.com/elasticbeanstalk/latest/dg/customize-containers-windows-ec2.html) in your Elastic Beanstalk environment. If you intend to run the cron job on all instances, simply remove it and the option will revert to false as its default.

So back to the cliffhanger. Why do we have to remove the all the cron jobs during deployment?

This is because the leader of the cluster in the elastic beanstalk environment may not stay the same during each deployment. Maybe it was lost, physically, in an annual California wildfire event, or maybe it was scaled down and the leader was lost as a result

Doing this will ensure that at all times, there will only be 1 instance that will be executing your cron job, if so desired.

HOWEVER~

I just lied.

According to [this forum thread](https://forums.aws.amazon.com/thread.jspa?threadID=113053), the leader of a cluster is determined during the elastic beanstalk environment creation. And it will remain the same throughout all subsequent deployments.

If the leader was lost during a timed scale down event, the leader is lost forever and the cluster in Elastic Beanstalk will remain leaderless until the end of time.

Which means the removal of cron jobs during each deployment is like taking off your pants to fart. It’s trivial.


Well, I’m just placing it there in case a leader reelection feature gets implemented.

The best way to solve this leaderless problem is to use a separate dedicated worker environment and [period tasks](https://repost.aws/forums?origin=thread.jspa&threadID=113053). It is the official and cleaner way to do cron jobs anyway. Having your main server clusters running cron jobs eat at their computation prowess and that is something you would should architect your system to avoid.

The downside of it is that you will need to consider the cost of having another server to handle this.
