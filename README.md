# Rerun

A fork of the original rerun: <http://github.com/alexch/rerun>

This fork only releases the latest commit to rubygems, which includes the `--force-polling`-flag and therefore work on WSL.

Rerun launches your program, then watches the filesystem. If a relevant file
changes, then it restarts your program.

Rerun works for both long-running processes (e.g. apps) and short-running ones
(e.g. tests). It's basically a no-frills command-line alternative to Guard,
Shotgun, Autotest, etc. that doesn't require config files and works on any
command, not just Ruby programs.

Rerun's advantage is its simple design. Since it uses `exec` and the standard
Unix `SIGINT` and `SIGKILL` signals, you're sure the restarted app is really
acting just like it was when you ran it from the command line the first time.

By default it watches files ending in: `rb,js,coffee,css,scss,sass,erb,html,haml,ru,yml,slim,md,feature`.
Use the `--pattern` option if you want to change this.

As of version 0.7.0, we use the Listen gem, which tries to use your OS's
built-in facilities for monitoring the filesystem, so CPU use is very light.

Rerun does not work on Windows. Sorry, but you can't do much relaunching
without "fork".

# Installation:

        gem install rerun

("sudo" may be required on older systems, but try it without sudo first.)

If you are using RVM you might want to put this in your global gemset so it's
available to all your apps. (There really should be a better way to distinguish
gems-as-libraries from gems-as-tools.)

        rvm @global do gem install rerun

The Listen gem looks for certain platform-dependent gems, and will complain if
they're not available. Unfortunately, Rubygems doesn't understand optional
dependencies very well, so you may have to install extra gems (and/or put them
in your Gemfile) to make Rerun work more smoothly on your system.
(Learn more at <https://github.com/guard/listen#listen-adapters>.)
For example, on Mac OS X, use

        gem install rb-fsevent

# Usage:

        rerun [options] [--] cmd

For example, if you're running a Sinatra app whose main file is `app.rb`:

        rerun ruby app.rb

If the first part of the command is a `.rb` filename, then `ruby` is
optional, so the above can also be accomplished like this:

        rerun app.rb

Rails doesn't automatically notice all config file changes, so you can force it
to restart when you change a config file like this:

        rerun --dir config rails s

Or if you're using Thin to run a Rack app that's configured in config.ru
but you want it on port 4000 and in debug mode, and only want to watch
the `app` and `web` subdirectories:

        rerun --dir app,web -- thin start --debug --port=4000 -R config.ru

The `--` is to separate rerun options from cmd options. You can also
use a quoted string for the command, e.g.

        rerun --dir app "thin start --debug --port=4000 -R config.ru"

Rackup can also be used to launch a Rack server, so let's try that:

        rerun -- rackup --port 4000 config.ru

Want to mimic [autotest](https://github.com/grosser/autotest)? Try

        rerun -x rake

or

        rerun -cx rspec

And if you're using [Spork](https://github.com/sporkrb/spork) with Rails, you
need to [restart your spork server](https://github.com/sporkrb/spork/issues/201)
whenever certain Rails environment files change, so why not put this in your
Rakefile...

    desc "run spork (via rerun)"
    task :spork do
      sh "rerun --pattern '{Gemfile,Gemfile.lock,spec/spec_helper.rb,.rspec,spec/factories/**,config/environment.rb,config/environments/test.rb,config/initializers/*.rb,lib/**/*.rb}' -- spork"
    end

and start using `rake spork` to launch your spork server?

(If you're using Guard instead of Rerun, check out
[guard-spork](https://github.com/guard/guard-spork)
for a similar solution.)

How about regenerating your HTML files after every change to your
[Erector](http://erector.rubyforge.org) widgets?

        rerun -x erector --to-html my_site.rb

Use Heroku Cedar? `rerun` is now compatible with `foreman`. Run all your
Procfile processes locally and restart them all when necessary.

        rerun foreman start

# Options:

`--dir` directory (or directories) to watch (default = "."). Separate multiple paths with ',' and/or use multiple `-d` options.

`--pattern` glob to match inside directory. This uses the Ruby Dir glob style -- see <http://www.ruby-doc.org/core/classes/Dir.html#M002322> for details.
By default it watches files ending in: `rb,js,coffee,css,scss,sass,erb,html,haml,ru,yml,slim,md,feature`.
On top of this, it also ignores dotfiles, `.tmp` files, and some other files and directories (like `.git` and `log`).
Run `rerun --help` to see the actual list.

`--ignore pattern` file glob to ignore (can be set many times). To ignore a directory, you must append '/*' e.g.
  `--ignore 'coverage/*'`.

  *On top of --pattern and --ignore, we ignore any changes to files and dirs starting with a dot.*

`--signal` (or `-s`) use specified signal (instead of the default SIGTERM) to terminate the previous process.
This may be useful for forcing the respective process to terminate as quickly as possible.
(`--signal KILL` is the equivalent of `kill -9`)

`--restart` (or `-r`) expect process to restart itself, using signal HUP by default
(e.g. `-r -s INT` will send a INT and then resume watching for changes)

`--clear` (or -c) clear the screen before each run

`--exit` (or -x) expect the program to exit. With this option, rerun checks the return value; without it, rerun checks that the launched process is still running.

`--background` (or -b) disable on-the-fly commands, allowing the process to be backgrounded

`--notify NOTIFIER` use `growl` or `osx` for notifications (see below)

`--no-notify` don't use growl (or osx) notifications

`--name` set the app name (for display)

`--force-polling` use polling instead of a native filesystem scan (useful for Vagrant)

Also `--version` and `--help`, naturally.

# Notifications

If you have `growlnotify` available on the `PATH`, it sends notifications to
growl in addition to the console.
If you have `terminal-notifier`, it sends notifications to
the OS X notification center in addition to the console.

If you have both, Rerun will pick one, or you can choose between them using `--notify growl` or `--notify osx` respectively.

If you have a notifier installed but don't want rerun to use it,
set the `--no-notify` option.

Download [growlnotify here](http://growl.info/downloads.php#generaldownloads)
now that Growl has moved to the App Store.

Install [terminal-notifier](https://github.com/julienXX/terminal-notifier) using `gem install terminal-notifier`. (You may have to put it in your system gemset and/or use `sudo` too.) Using Homebrew to install terminal-notifier is not recommended.

# On-The-Fly Commands

While the app is (re)running, you can make things happen by pressing keys:

* **r** -- restart (as if a file had changed)
* **c** -- clear the screen
* **x** or **q** -- exit (just like control-C)
* **p** -- pause/unpause filesystem watching

If you're backgrounding or using Pry or a debugger, you might not want these
keys to be trapped, so use the `--background` option.

# Signals

The current algorithm for killing the process is:

* send [SIGTERM](http://en.wikipedia.org/wiki/SIGTERM) (or the value of the `--signal` option)
* if that doesn't work after 4 seconds, send SIGINT (aka control-C)
* if that doesn't work after 2 more seconds, send SIGKILL (aka kill -9)

This seems like the most gentle and unixy way of doing things, but it does
mean that if your program ignores SIGTERM, it takes an extra 4 to 6 seconds to
restart.

# Vagrant and VirtualBox

If running inside a shared directory using Vagrant and VirtualBox, you must pass the `--force-polling` option. You may also have to pass some extra `--ignore` options too; otherwise each scan can take 10 or more seconds on directories with a large number of files or subdirectories underneath it.

# Troubleshooting
If you are using `zsh` as your shell, and you are specifying your `--pattern` as `**/*.rb`, you may face this error
```
Errno::EACCES: Permission denied - <filename>
```
This is because `**/*.rb` gets expanded into the command by `zsh` instead of passing it through to rerun. The solution is to simply quote ('' or "") the pattern.
i.e
```
rerun -p **/*.rb rake test
```
becomes
```
rerun -p "**/*.rb" rake test
```

# To Do:

## Must have for v1.0
* ".rerun" file to specify options per project or in $HOME.
* Make sure to pass through quoted options correctly to target process [bug]
* Optionally do "bundle install" before and "bundle exec" during launch

## Nice to have
* Smarter --signal option (specifying signal to try and a timeout to wait, repeated)
* If the last element of the command is a `.ru` file and there's no other command then use `rackup`
* Figure out an algorithm so "-x" is not needed (if possible) -- maybe by accepting a "--port" option or reading `config.ru`
* Specify (or deduce) port to listen for to determine success of a web server launch
* see also [todo.md](todo.md)

## Wacky Ideas
* Make it work on Windows, like Guard now does. See
  * https://github.com/guard/guard/issues/59
  * https://github.com/guard/guard/issues/27
* On OS X:
    * use a C library using growl's developer API <http://growl.info/developer/>
    * Use growl's AppleScript or SDK instead of relying on growlnotify
    * "Failed" icon for notifications
* On Linux:
    * Use libnotify or notify-send http://www.linuxjournal.com/content/tech-tip-get-notifications-your-scripts-notify-send

# Other projects that do similar things

* Restartomatic: <http://github.com/adammck/restartomatic>
* Shotgun: <http://github.com/rtomayko/shotgun>
* Rack::Reloader middleware: <http://github.com/rack/rack/blob/5ca8f82fb59f0bf0e8fd438e8e91c5acf3d98e44/lib/rack/reloader.rb>
* The Sinatra FAQ has a discussion at <http://www.sinatrarb.com/faq.html#reloading>
* Kicker: <http://github.com/alloy/kicker/>
* Watchr: <https://github.com/mynyml/watchr>
* Guard: <http://github.com/guard/guard>
* Autotest: <https://github.com/grosser/autotest>

# Why would I use this instead of Shotgun?

Shotgun does a "fork" after the web framework has loaded but before
your application is loaded. It then loads your app, processes a
single request in the child process, then exits the child process.

Rerun launches the whole app, then when it's time to restart, uses
"kill" to shut it down and starts the whole thing up again from
scratch.

So rerun takes somewhat longer than Shotgun to restart the app, but
does it much less frequently. And once it's running it behaves more
normally and consistently with your production app.

Also, Shotgun reloads the app on every request, even if it doesn't
need to. This is fine if you're loading a single file, but if your web
pages all load other files (CSS, JS, media) then that adds up quickly.
(I can only assume that the developers of shotgun are using caching or a
front web server so this isn't a pain point for them.)

And hey, does Shotgun reload your Worker processes if you're using Foreman and
a Procfile? I'm pretty sure it doesn't.

YMMV!

# Why would I use this instead of Rack::Reloader?

Rack::Reloader is certifiably beautiful code, and is a very elegant use
of Rack's middleware architecture. But because it relies on the
LOADED_FEATURES variable, it only reloads .rb files that were 'require'd,
not 'load'ed. That leaves out (non-Erector) template files, and also,
at least the way I was doing it, sub-actions (see
[this thread](http://groups.google.com/group/sinatrarb/browse_thread/thread/7329727a9296e96a#
)).

Rack::Reloader also doesn't reload configuration changes or redo other
things that happen during app startup. Rerun takes the attitude that if
you want to restart an app, you should just restart the whole app. You know?

# Why would I use this instead of Guard?

Guard is very powerful but requires some up-front configuration.
Rerun is meant as a no-frills command-line alternative requiring no knowledge
of Ruby nor config file syntax.

# Why did you write this?

I've been using [Sinatra](http://sinatrarb.com) and loving it. In order
to simplify their system, the Rat Pack removed auto-reloading from
Sinatra proper. I approve of this: a web application framework should be
focused on serving requests, not on munging Ruby ObjectSpace for
dev-time convenience. But I still wanted automatic reloading during
development. Shotgun wasn't working for me (see above) so I spliced
Rerun together out of code from Rspactor, FileSystemWatcher, and Shotgun
-- with a heavy amount of refactoring and rewriting. In late 2012 I
migrated the backend to the Listen gem, which was extracted from Guard,
so it should be more reliable and performant on multiple platforms.

# Credits

Rerun: [Alex Chaffee](http://alexchaffee.com), <mailto:alex@stinky.com>, <http://github.com/alexch/>

Based upon and/or inspired by:

* Shotgun: <http://github.com/rtomayko/shotgun>
* Rspactor: <http://github.com/mislav/rspactor>
  * (In turn based on http://rails.aizatto.com/2007/11/28/taming-the-autotest-beast-with-fsevents/ )
* FileSystemWatcher: <http://paulhorman.com/filesystemwatcher/>

## Patches by:

* David Billskog <billskog@gmail.com>
* Jens B <https://github.com/dpree>
* Andrés Botero <https://github.com/anbotero>
* Dreamcat4
* <https://github.com/FND>
* Barry Sia <https://github.com/bsia>
* Paul Rangel <https://github.com/ismell>
* James Edward Gray II <https://github.com/JEG2>
* Raul E Rangel <https://github.com/ismell> and Antonio Terceiro <https://github.com/terceiro>
* Mike Pastore <https://github.com/mwpastore>
* Andy Duncan <https://github.com/ajduncan>

# Version History

*
  * `--force-polling` option (thanks ajduncan)

* 0.11.0    7 October 2015
  * better 'changed' message
  * `--notify osx` option
  * `--restart` option (with bugfix by Mike Pastore)
  * use Listen 3 gem
  * add `.feature` files to default watchlist (thanks @jmuheim)

* v0.10.0   4 May 2014
  * add '.coffee,.slim,.md' to default pattern (thanks @xylinq)
  * --ignore option

* v0.9.0    6 March 2014
  * --dir (or -d) can be specified more than once, for multiple directories (thanks again Barry!)
  * --name option
  * press 'p' to pause/unpause filesystem watching (Barry is the man!)
  * works with Listen 2 (note: needs 2.3 or higher)
  * cooldown works, thanks to patches to underlying Listen gem
  * ignore all dotfiles, and add actual list of ignored dirs and files

* v0.8.2
  * bugfix, forcing Rerun to use Listen v1.0.3 while we work out the troubles we're having with Listen 1.3 and 2.1

* v0.8.1
  * bugfix release (#30 and #34)

* v0.8.0
  * --background option (thanks FND!) to disable the keyboard listener
  * --signal option (thanks FND!)
  * --no-growl option
  * --dir supports multiple directories (thanks Barry!)

* v0.7.1
  * bugfix: make rails icon work again

* v0.7.0
  * uses Listen gem (which uses rb-fsevent for lightweight filesystem snooping)

# License

Open Source MIT License. See "LICENSE" file.
