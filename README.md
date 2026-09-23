=head1 OVERVIEW

Document stub to help new developers get going and guidence on how
to work with the process

=head1 DEVELOPMENT MACHINES

There are dedicated computers for CATME development which are running
a version of CATME under a virtual machine.  These VMs are based on the
same golden master that is running production (although production has
some additional performance tweaks and security hardening).  All development
work needs to be on a designated VM.

For learning how to setup a local VM see L<https://github.com/jjn1056/CatmeOps-V3>

=head2 WORKING WITH GIT

All developers should have a Github account that is dedicated to themselves.
You are free to use an existing personal Github account, or you may wish to
create a new account just for the CATME work.  You cannot use a shared account.

You should always create a new branch cut from master when starting a new project.
Please keep your local git repo up to date (git fetch and rebase off the Github
master before cutting a new branch).  You must merge from master before submitting
a PR.  Please review your PR for silly changes like pointless formatting only changes
and to make sure you are not blowing away something you don't intend.  Remember that
the reviewer has to examine every single proposed change and the more changes you
have that are not on target is just going to make it harder to finish the review.

Ideally Branches should be short lived.  Try to break your bigger projects into
chunks that are no longer than 2 weeks.  Any branch older than 2 weeks will become
increasingly hard to test and properly merge.

=head2 CPAN DEPENDENCIES

All Perl CPAN dependencies are managed via the C<cpanfile> located in the root
of the checkout directory C</var/www/src> or C<~/catme-git/>.  If you need a
new dependency you should add it to the C<cpanfile> (althought please check with
John before adding dependencies).

When pulling changes down from master you may note that C<cpanfile> was updated;
if so you must update your local dependencies to match.  You can do this most
easily using a C<make> command (from the root of the project git checkout)

    make update_cpanlib

B<NOTE> Its probably a good idea to just have the habit of running this command
anytime you sync from master, just to be sure you catch any updates.

=head2 DATABASE MANAGEMENT

You can open a PSQL terminal session with the Makefile command:

    make db

Database migrations are managed vis Sqitch: L<https://sqitch.org/>.You can review
the online documentation but for the most part you need to either apply
SQL migration updates that you pull down from master or create your own.  To
apply updates you can use the Makefile command:

    make update_db

This command is idempotent so feel free to run it defensively anytime you merge
master into your working branch.  You can combine this will updating any C<cpanfile>
CPAN dependency changes with this command:

    make update

Which will both update the DB and CPAN libs.  To check the status of your local
database you can run (from the root of the git checkout).

    sqitch status 

If you need to add a new migration please be sure to have read the Postgresql
Sqitch tutorial (L<https://sqitch.org/docs/manual/sqitchtutorial/>) to become
more deeply aware of the system.  The most important command is the C<add>
command (example, run from the root of the git checkout):

    sqitch add 2020012800_class_timestamp -n 'timestamp for class'

And that will create stubs under C</sql> directory for updates, reverts and verification
scripts.  These scripts are SQL scripts.  You can review others in the directory
for examples.  All three are mandatory (No PR without all three will be accepted).

L<Sqitch> doesn't require it but for this project you must name each migration
with a timestamp prefix (this just makes it easier for me to review the progression of
migrations without having to review the plan file).  The pattern is:

      YYYYMMDDII_meaningful_title

Where C<YYYY> is a 4 digit year (2020), C<MM> is a two digit month (01-12 = Jan - Dec) and
C<DD> is a two digit day (01-[28,29,30,31]).  <II> is a increment counter starting with '00'
for each patch you do in a given day.

The C<-n> flag is required and should expand on the meaninful name in order to help reviewers
quickly grasp the value of the patch.

You should group all the changes in a patch this is associated with a given unit of work.  Ideally
the patch is as small as possible but don't scatter changes associated with one assignment over
several patches if you can avoid that since it makes rollbacks harder and interfers with people
understanding the totality of the work.

Don't include 'extras' in your patch that are things you want but unrelated to the current
project.  Place those in separate patches.

B<NOTE>: If you add new tables to the database you need to review if you need to make changes
to the bin/create_devdb.sh script.  This script is used to build a database dump suitable for
development.  Its complicated, you'll probably need to ask John for details.

=head2 ACCESSING THE DATABASE FROM THE HOST

During the build we change the security on the Postgresql server running inside the guest
virtual machine so tht you can access it from your HOST operating system.  You might find
this useful if you are running database design and managment tools on your host OS.  Due to
Vagrant port mapping, the Postgresql server will be accessible via port 5433.  Example logging
into the guest virtual machine Postgresql server from a Psql commandline utility running in a
terminal on the Host operating system:

    psql --host localhost --port 5433 --dbname catme --user catmerw

You can use those configuration values in a GUI tool like PgAdmin.

=head2 ACCESSING THE WEB APPLICATION FROM THE HOST

Although the Apache webserver runs on port 80/443 inside the guest virtual machine, Vagrant will
map that to 8080 / 8443 on the host OS (because virtualbox doesn't run as root on your host OS).
So if you want to open a web browser to the CATME application you need to specify the correct
port in the URL:

    https://localhost:8443/login/index

Please note that we have a fake certificate on the vagrant development server so your browser
might complain about that and require you to add a security exception.  I also find that 
recent versions of Safari won't allow it at all.

You can use the following credentials to get logged in.  This is a multi role user that can
access administration, student and instructor screens:

    user/email: faculty2@sysiphus.com
    password:   greatone123

=head2 ADVICE

If you find yourself performing some wacky hack to get around how something
is not working according to the process, please stop and ask John for the
right solution ;)  CATME is not brain surgery and you shouldn't have to jump
thru a ton of hoops to do simple things.

For getting started on Perl try https://learn.perl.org/

=head2 COOKBOOK

General Makefile usage.  These commands are intended to be run from inside the GUEST
virtual machine, off the root of the git repository checkout:

    make help

Tailing the web server logs:

    make tail_logs

opening a C<psql> session on the database:

    make db

Updating CPAN libraries and running outstanding database migrations.  Note that
this command is idempotent so you can run it as often as you want without risk:

    make update

Reset the database to the current saved development DB.  Useful for testing and
you need the DB to return to a known state.  Please note that when you reset
the DB you lose existing sessions info and related transactional info so you 
might need to log in again.  Also things like 'magic_strings' from outstanding
password resets are canceled.  There maybe so other similar things.  For example
the 'system_stats' table will be empty and you'll need to kick off a rebuilt of
it from the administration screen if you need to see it.

    make reset_devdb

B<NOTE> This command will only run a a development virtual machine.

There's other helpers built into the Makefile, its worth checking 'make help'
for updates (and the README.pod in the Ops repo also has details). Just be
warning that running the hourly and nightly jobs cn result in sending emails
so be sure rto always use the sysiphus test accounts.

=head1 DEPLOYMENT AND CHANGE MANAGEMENT

=head2 Working on a ticket

When you begin working on a new ticket you should cut a new branch from the top of
the master branch.  While working on that ticket you should commit frequently your
changes to the branch and you must push the branch up to the git repository at least
once a day.  Never end the day or walk away from your development machine without
pushing the current work up to the git repo (even if your code is in a broken state
you should do this to avoid losing work in the event you suffer issues with your
development machine).

When work is completed on the ticket you should send a pull request for your branch
and request a review.   Developers are not permitted to work directly off master or
to merge to master branch without permission and code review.

=head2 Deploying to production

CATME generally follows a 'release frequently' approach to deployments.  We prefer
to release smaller code batches which pass code review and QA rather than queue up a
larger change set.   However we do not release code Friday - Sunday, nor after 4PM US
Central time unless there's a clear emergency code release needed to address critical
and time sensitive bugs and security issues.

CATME Deployment is currently a manual process.  The release manager logs into the CATME
production system via SSH (using a private key, and from an IP address that is opened to
SSH via the firewall), elevates their priviledges and then pulls down code updates from
git.  If necessary new code dependencies are installed (these dependencies are managed via
the C<cpanfile>) and database migrations are applied (via C<sqitch>).   If neccessary
we initiate a web server soft restart to load any new shared libraries.

The C<Makefile> contains commands to semi automate this process.

Once deployment is completed, the release manager should test basic systems such as login
as well as check that any new behaviors contained in the release are functioning as expected.
Also one should monitor both the system logs as well as Google Analytics for at least an hour
to verify that no sudden upticks in error conditions occur.

After the release is completed the release manager should email the product owners and inform
them of the system status as well as any exception conditions that occured during the release (
such as unexpected errors or downtime).

=head2 Reverting 

Generally we prefer to 'roll forward' on errors when possible, but if we encounter a critical
hard error on production following a deployment and when the root cause is not quickly determined
you can roll back using the following procedure:

Check out the local git to th commit of the last release.  Run any C<sqitch> reversions needed.
Soft restart the httpd server if needed.  Then verify the site function.   Afterwards you must
email a report to the product owners.

=cut
