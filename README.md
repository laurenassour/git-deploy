
# git-deploy

git-deploy is an ultra-lightweight continuous deployment tool packaged as a git plugin. It works by creating a bare repository on a remote server, with receive hooks that run a command to deploy your code. This creates a single point of integration and deployment that can be used by small and medium-sized teams that don't have to capacity or interest to set up more complex systems.

## Dependencies
 * Git  (Obviously)
 * Make (Used to run deployment tasks)

## Installation

Like all git plugins, to install git-deploy you simply add it to your PATH. 

There are many ways to do this. Here is one way that will only change the path in the current terminal session. You'll probably want something more permanent.

```
$ git clone https://github.com/benrady/git-deploy.git
$ export PATH=$PWD/git-deploy/bin:$PATH
```

## Usage

First, if you don't already have one, you need to create a [Makefile](http://mrbook.org/blog/tutorials/make/). This makefile should have a target named `git-deploy` that takes the current working directory and does whatever you need to do to deploy your code.

For example, if you have a static website in the `public` directory of your repository, you'll want a makefile that looks like this:

```Makefile
git-deploy:
        ln -f -s -T ${PWD}/public /var/www/html/my_app
```

If you have a Ruby/Python/Perl/etc application that can be run directly from a /service directory using a tool like [runit](http://smarden.org/runit/) or [daemontools](https://cr.yp.to/daemontools.html), you can create a symlink to the /service directory in this target:

```Makefile
git-deploy:
        ln -s -f -T ${PWD} /service/my_app
```

If you have a Java application that needs to be compiled first, you'll want something like this:

```Makefile
git-deploy:
        mvn clean install
        ln -s -f -T ${PWD}/target /service/my_app
```

## Continuous Integration

The hooks that git-deploy installs will reject a push if the build command fails. This means you can add tests or other sanity checks to the build to ensure that everything that is deployed passes a minimum threshold of correctness. If the build fails, the currently running service will not be interrupted.

## Rollback

git-deploy keeps a copy of every version of your application that you've ever deployed. These are kept (named for the SHA of the HEAD commit) in the `~/.git-deploy/[repo name].git/.build` directory on the remote server.

You can simple list the subdirectories in this directory to see what versions you've deployed. `ls -alt` is useful here. 

To roll back to a specific version, run `make git-deploy` from inside one of those directories. This will re-run the deploy for that version and roll your service back. Any logs or data files that were in that directory will have been preserved, allowing you to roll your application state back along with the code.

## Special Cases and FAQ

### Why won't my application stop running?
Using a service manager like runit or daemontools means you need to understand how your process starts and stops. These tools send a (signal)[] to the run script in your application when they're supposed to stop. Be sure that you're using `exec` in your run script to _replace_ the bash process with python/ruby/java/binary so that it actually receives the signal.

If that's not sufficient, you may need to take steps in your makefile to shut down the existing service. You'll probably want to do this _after_ building and/or running tests though. For example:

```Makefile
git-deploy:
        mvn clean install
        pkill -f hard_to_kill.jar
        ln -s -f -T ${PWD} ~/service/hard_to_kill
```

### I deployed once but it didn't work. Now things are messed up.

The safest way to "reset" things is to the delete:
 1. The bare repository on the remote server (located at ~/.git-deploy/[repo name].git/)
 2. The git-deploy remote that was added to your local git configuration. You'll see it when you run `git remote -v`