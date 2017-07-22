---
layout: post
title:  "Spring and RSpec 3 in RubyMine"
permalink: /2015/02/08/spring-rspec3-rubymine
date:   2015-02-08 12:00:00 -0700
categories: tools rubymine
author_slug: matt
is_featured: false
feature_image: feature-turtle-rock.jpg
---

I love [RubyMine](https://www.jetbrains.com/ruby/) - I've said so in a [previous post](http://barbarian.engineer/2015/01/23/i-use-an-ide-and-i-am-not-ashamed/) - and I do almost all my work within the IDE.

This includes **RSpec**, which is delightfully integrated with the editor.  Until recently, I had used **spork** as a persistent test runner - spork would run continuously, and each time I pressed the key combination to run a test, it would be handed off to the long-running spork process in the background.

But this stopped working when I upgraded to RSpec 3.  Though some StackOverflow answers suggested fixes, many also suggested migrating to **spring**, a similar quick-start service that was now part of Rails by default.

OK, I thought.  Using the same tools as everyone else is good.  Standards are good.  Omakase is good.  Let's convert to Spring.

## Start from zero.

I began by deleting my old `.rspec` and `spec_helper.rb` files - the spec_helper  was full of Spork and RSpec 2.x configuration - nothing worth keeping.

In the Gemfile I uncommented the `gem "spork"` line, and also told it to grab the latest RSpec, now 3.2.0.

I ran `rails g rspec:install` to give me new configuration files.

Then I ran a single spec from the terminal, to prove rspec worked:

    $ rspec spec/models/foo_spec.rb
    Finished in 0.00871 seconds (files took 3.54 seconds to load)

## Enable Spring in RubyMine

I started a fresh copy of RubyMine, and navigated its menus to edit the run configurations:

    Run > Edit Configurations > Defaults > Rspec
    Use pre-load server: - Spring -

I selected Spring in the pre-load server menu.  I also ensured that all my existing Rspec run configurations (the ones it creates automatically for each `*_spec.rb` file) were gone - I wanted it to create them anew, based on the current settings.

Back to the editor windows, I opened the same spec file I'd just run from the terminal and hit Shift-Control-F10 to run it as a test.

But instead of running the test, it just printed Spring's usage message:

    Usage: spring COMMAND [ARGS]

    Commands for spring itself:

      binstub         Generate spring based binstubs. Use --all to generate a binstub for all known commands.
      help            Print available commands.
      status          Show current status.
      stop            Stop all spring processes for this project.

    Commands for your application:

      rails           Run a rails command. The following sub commands will use spring: console, runner, generate, destroy.
      rake            Runs the rake command

Binstubs, eh?  That looks promising...

```
$ spring binstub --all
* bin/rake: spring inserted
* bin/rails: spring inserted
```

Running from RubyMine again, still no joy - it prints a usage statement only.  To see if it was invoking `bin/spring`, I added a line to that file - `puts "* this is bin/spring"`, and indeed, that prints, but spring doesn't seem to know how to invoke rspec.

## Spring Commands RSpec't

As is the custom of my people, I consulted the google, and the README at  https://github.com/rails/spring provided an answer. Spring doesn't know about RSpec out of the box; a plugin is required.  That plugin is the `spring-commands-rspec` gem, which is a great name that reads perfectly well as an English sentence.  (As an aside, it would have been helpful if Spring printed suggestions to install plugins whenever presented with a missing command that matched a known plugin name!  That would save newcomers a bit of time).

Adding `gem 'spring-commands-rspec'` and running Bundler again, Spring binstub now adds "rspec" to the list of commands in its usage statement.  That's progress.

While doing this, my RubyMine instance was still open, and without changing anything I just hit the button to run the last test again (**[>]** the "Play" symbol).  **Success!**

That first run took five seconds or so, but when I pressed the button a second time it was instantaneous - indeed, it successfully ran the test in Spring.  I hadn't even generated a binstub yet - RubyMine knew how to invoke "spring rspec" on its own.

## spring binstub rspec

Occasionally I run all tests from the command line, not RubyMine, so let's add that to Spring as well:

    spring binstub rspec
    * bin/rspec: generated with spring

The global "rspec" does not invoke this, but if I run bin/rspec directly it works, running all tests super-fast.  I can live with having to remember to type "bin/rspec" or "spring rspec"... not that it matters, as I mostly run it in RubyMine anyway.

## There Can Be Only One!

After running "spring rspec" from a shell prompt, I pressed the **[>]** button to re-run a spec in RubyMine:

...active_support/dependencies.rb:274:in `require': cannot load such file -- teamcity/spec/runner/formatter/teamcity/formatter (LoadError)

Teamcity is Rubymine-specific code that gets injected when rspec is invoked from Rubymine.  When I ran spring from a terminal, however, this module was missing.  As RubyMine expected it to be there when it next tried to communicate with my running Spring process, it failed.

**A Spring process started from outside RubyMine will confuse and thwart RubyMine's attempts to use Spring.**

To fix it, kill the spare:

    $ spring status
    28729 spring server | TheTubes | started 7 mins  
    28732 spring app    | TheTubes | started 7 mins ago | test mode   
    $ spring stop

Then in RubyMine, run your test - Shift-Ctrl-F10 or the **[\>]** button - and it runs happily. 

## The Lost Spring

*(This section is based on my experiences on two Linux machines - Ubuntu 12 & 14 - and may not apply on other systems)*

After killing the standalone Spring and working exclusively in RubyMine for some hours, all of a sudden it stopped working:

```
Testing started at 2:16 PM ...
/usr/local/rvm/gems/ruby-2.2.0@the_tubes/gems/spring-1.3.0/lib/spring/sid.rb:39:in `getpgid': No such process (Errno::ESRCH)
  from /usr/local/rvm/gems/ruby-2.2.0@the_tubes/gems/spring-1.3.0/lib/spring/sid.rb:39:in `pgid'
```

Spring was no longer running - and RubyMine was failing to start up an instance of it as it usually did automatically.

The only way I could get it to work again was to **shut down RubyMine completely** and start it up again.

Through trial and error, I eventually discovered that a Spring process would die if the terminal where I had started RubyMine went away - and it couldn't start a new one without being connected to a terminal (leading to the getpgid error - apparently Spring was really close to its process group leader, in a Norman Bates kind of way).

I'd always started RubyMine by cd'ing to the project directory in a terminal ("konsole" in KDE) and typing `mine . &`.  After a few seconds, the IDE would appear... and I'd still have use of the terminal window for running other things.  Sometimes, though, I'd just close that window, forgetting that there was something special about it: it was home to a backgrounded RubyMine process.  Doing this for years, I'd never had a problem, until Spring came along.

**Spring does not abide loss of its controlling terminal.** Now, when I start RubyMine from a terminal, I no longer place it in the background with "&" - I let it be the foreground process, so it has exclusive use of that `konsole` window, and then minimize that window so I don't have to think about how sad and lonely that terminal process is.

It's kind of an ugly hack, but it works.

## It's Terminals All The Way Down

Writing the previous section got me thinking - not everyone starts GUI programs from the command line.  Some people like clicking on menus and icons.  Might that actually work better?

The controlling terminal of the KDE Desktop shell is the system console - which of course never goes away.  What if I could get that to be the terminal where Spring attaches itself?

RubyMine wasn't in my KDE menu system, because I hadn't installed it through the Ubuntu package manager.  So I ran it directly from the desktop shell - Ctrl-F2, then "/usr/local/bin/mine".  A splash screen appeared, followed by the project picker.  I selected my project and ran a spec - success!  Ran it a second time, it was instantaneous, proving it was using Spring.  A `ps` listing showed that the `spring` process wasn't associated with a terminal, like pts/8, but had only a question mark in the TTY column.

What if I kill the spring process?  RubyMine simply started another one.

Very, very good.

I then created a KDE launcher icon, in my always-visible taskbar - selecting "/usr/local/bin/mine" as the executable and "/usr/local/RubyMine/bin/rubymine.png"  as the icon.  I next launched RubyMine from there, opened my project, ran a spec, and it worked flawlessly.

After shutting down RubyMine, spring is still running.  Curiously, I start another RubyMine, run the spec again - and it works, using the Spring server started by the previous RubyMine!  Perfect, this is a better solution than I was hoping for.

## Spring, Devourer of Keystrokes.

One last flaw remained with my setup: when invoking "rails console" from a terminal, if I then control-Z out of it, my terminal session becomes seriously messed up - when I type a shell command, Spring grabs half my keystrokes and feeds them to the rails process that I wanted to suspend.  I can't even type "fg" to get back to the rails process, as those characters also get misdirected.  The only way out is to close the terminal and start another.

Apparently this is a known issue that has not yet been fixed.  My workaround is to not use spring for the "rails" command - use it only for rake and rspec - and just deal with a few extra seconds every time I type "rails console".

In bin/rails, I comment out the "load spring" line.  "rails console" now bypasses spring, and my terminal doesn't go all wonky.

*(Yeah, I use Control-Z / Suspend far too much.  I'm old, I remember when opening another terminal took more than a few seconds, and still have those habits)*

## Living with Spring

* Remember that spring doesn't know about rspec unless you add the gem **spring-commands-rspec**
* When running in RubyMine, a **teamcity/formatter** error means there's a non-RubyMine spring running - kill it.
* If **getpgid** gives a fatal error, spring's terminal went away - you have to quit RubyMine and start it up again
* Starting RubyMine from the desktop - not a terminal! - will work around that error.
* Don't let "rails console" use spring if you're in the habit of suspending shell processes.


