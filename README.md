# Overview

I sent an email about this to Jason@zx2c4.com on 9/9/2015 because I didn't
want to post to a public mailing list about a security issue.  I never got a
response back.  The email might have been overlooked or a response might have
been in the works and just never sent to me.  If either of those are the case
I apologize for exposing this, but I would like to make sure pass users have a
chance to rectify the problem if need be.

I also didn't have a workaround at the time, but I didn't want to
release information to the public without giving the user a way to make
themselves safer.

The vulnerability is inherent to all secure programs which exec another
program to perform a function.

## Description of the Problem

gpg, env, git and any other dynamic pathed binary can be overridden by someone with write access to the path.


## TL;DR

It's trivial to capture a passphrase and upload it if you have access to the
users account.

## Example PATH where this might be an issue

    rdrake@machine:~$ echo $PATH
    /home/rdrake/perl5/bin:/home/rdrake/perl5/perlbrew/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games


CPAN local:lib scripts setup perl5/bin to come first, and usually that is
writable by the user.

## Proof of concept

You can create a file like this:

    cat ~/perl5/bin/gpg2
    #!/bin/bash
    echo "args were $*" > /tmp/file.$$
    exec /usr/bin/gpg2 "$@"

This runs without saying anything and the user doesn't see a thing.  All this
does is echo the arguments into a file.

That file could just as easily prompt the user for a passphrase and
save it, then call gpg with --passphrase-fd to give it the passphrase the user
just gave them. All while uploading the entire password store somewhere else
along with the passphrase.

## Fixes

The safest solution is to hardcode all the paths in the pass script and
possibly run a one-time setup routine which finds all the binaries first.  A
good way of doing this might be to use a template engine like m4,
template-toolkit, or whatever your modern choice is.  Alternatively, you could
use semaphores like %%GIT%% and search and replace them with

    sed 's|%%GIT%%|/path/to/git|'

A final alternative would be to use a sourceable file with the paths stored as
environment variables, then change all the references in the pass script to
use the sourced file.  Something like:

    P_RM=/bin/rm
    P_GIT=/usr/bin/git

The P\_ being arbitrary and used to distinguish shell commands from other
environment variables used in pass.

The "env" line at the top of the pass program needs to be fixed with search
and replace, or again replaced with a semaphore.

I found this while I was struggling with writing a perl wrapper for pass. Most
of the perl libraries exec gpg and face the same problems when it comes to
trusting external programs. I had decided to scrap what I wrote and switch to
Crypt::OpenPGP because it was native perl, then I decided to look at how pass
did it.

## What pass-hardcodepath does

First it will check to see if any executed binaries are really scripts and
warn the user to audit them.  Then it creates a new "pass" file in the
directory with hard-coded paths.  It does this without semaphores and needs to
account for places where barewords like "git" are used in documentation, and
other special cases.

This is intended as both a workaround for users and a starting point for
developers if they wish to make changes to the way things work.  With this,
you might easily convert the input file to semaphores or bash environment
variables, then write a new script that does post-install audit/path fixup.

## Maintaining hard coded paths

In order to keep things safe, you might change the way you run pass so that it
doesn't use a PATH.  An alias like this might work:

    alias pass='PATH=/dev/null /usr/bin/pass'

This breaks "pass git pull" because git-pull is a shell script and needs sed
and other things.  That is the only limitation I've found so far.  I would
probably recommend running "pass git pull" with a reduced safe path.

## restricting the PATH environment variable vs hardcoding all paths

Maybe an installation script could build a compatible path by stripping all
user own paths from the environment.  This would be an example of the "reduced
safe path" that I mentioned above.  Another alternative might be to hardcode
the path to known safe directories for that particular distribution/OS.

I don't like this as a global fix though, I honestly prefer this not to happen
at runtime inside pass because some users might have a local copy gpg or another
binary installed in their home directory, it might be their preferred version
of a binary or it might be a host where they have no admin rights and can only
install binaries that are writable by themselves.  If it's fixed this way then
those users just have a broken program by default.

## Some references

A Style guide might help if one is not already used.  Google's seems to be
short and easy to implement while also covering some security concerns.

https://google-styleguide.googlecode.com/svn/trunk/shell.xml

Also, defensive programming in general should be practiced, but there are a
couple of articles related to this.  hackernews found one around two years ago
and the comments are very useful.

https://news.ycombinator.com/item?id=7815190

The point being that since this is an app related to security, you need to be
double sure you look at new code or features from an angle of how to break it
or attack it.

## Why bother?

If a user has a compromised account and a USER-writable PATH, then someone
could just as easily override the entire "pass" script right?

This is true, but it doesn't mean the entire problem should be ignored.  If
someone is paranoid and wants to do something about that type of issue they
could solve it with the abovementioned alias that cancels PATH usage.  They
could even rename their "pass" executable to "puppies" to avoid automated
attacks.  These protections would be meaningless if the PATH still needed to
be used in the script though.

## Other possible issues

I found a couple of other things that might be problems but I don't have time
to look at them carefully.  If any of the pass developers want additional
details, you can contact me via github or email.
