# Upgrading

## Bundler 3

The following is a summary of the changes that we plan to introduce in Bundler
3, why we will be making those changes, and what the deprecation process will
look like. All this deprecations will be printed by default in the upcoming
Bundler 2.1 release.

If you don't want to deal with deprecations right now and want to toggle them
off, you can do it through configuration. Set the `BUNDLE_SILENCE_DEPRECATIONS`
environment variable to "true", or configure it through `bundle config` either
globally through `bundle config set silence_deprecations true` command, or
locally through `bundle config set --local silence_deprecations true`. From now
on in this document we will assume that all three of these configuration options
are available, but will only mention `bundle config set <option> <value>`.

As a general note, these changes are intended to improve the experience using
bundler for _new_ users, who have no existing usage routines nor possibly biased
opinions about how the tool should work based on how it has historically worked.
We do understand that changing behaviour that have been existing for years can
be annoying for old users, that's why we intend to make this process as smooth
as possible for everyone.

I'll be dividing the deprecations into three groups: CLI deprecations, DSL
deprecations, and misc deprecations. Let's dive into each of them.

#### CLI deprecations

The CLI defines a set of commands and options that can be used by our users to
create command lines that bundler can understand. There's a number of changes
that we plan to make to this set of commands and options.

* Flags passed to `bundle install` will no longer be remembered as configuration.

  This has been a source of historical confusion and bug reports, not only for
  beginners but also for experienced users. A CLI tool should not behave
  differently accross exactly the same invokations _unless_ explicitly
  configured to do so. This is what configuration is about after all, and things
  should never be silently configured without the user knowing about it. The
  problem with changing this behavior is that very common workflows are relying
  on it. For example, when you run `bundle install --without development:test`
  in production, those flags are persisted in the app's configuration file and
  further `bundle` invokations will happily ignore development and test gems.
  This magic will disappear from future bundler versions, and you will
  explicitly need to configure this, either through environment variables,
  application configuration, or machine configuration.

* The `--force` flag to `bundle install` and `bundle update` has been renamed to `--redownload`.

  This is just a simple renaming of the flag, to make more apparent what it
  actually does. This flag forces redownloading very gem, it doesn't "force"
  anything else.

* `bundle viz` will be removed and extracted to a plugin.

  This is the only bundler command requiring external dependencies, both an OS
  dependency (the `graphviz` package) a gem dependency (the `ruby-graphviz`
  gem). Remove these dependencies make development easier and it was also seen
  by the bundler team as an opportunity to develop a [bundler plugin] that it's
  officially maintained by the bundler team, and that users can take as a
  reference to develop their own plugins. The plugin contains the same code as
  the old core command, the only difference being that the command is now
  implemented as `bundle graph` which is much easier to understand. Have a look
  at https://github.com/bundler/bundle-viz for more information about how to
  install and use the new plugin.

* The `bundle console` will be removed and replaced with `bin/console`.

  Over time we found `bundle console` hard to maintain because every user would
  want to add her own specific tweaks to it. In order to ease maintainance and
  reduce bikeshedding discussions, we're removing the `bundle console` command
  in favor of a `bin/console` script created by `bundle gem` on gem generation
  that users can tweak to their needs.

  [NOTE: Should we provide an easy way to add `bin/console` to an existing gem?]

* The `bundle update` command will no longer update all gems, you'll need to pass `--all` to it.

  [NOTE: I'm not elaborating on this change yet because I have hesitations about
  it which I've mentioned in https://github.com/bundler/bundler/pull/7008. After
  reading up on it a bit more and seeing that it all starts with "many people
  think that they should run bundle update after git pull". Why is this? Isn't
  this a documentation issue that shouldn't make us change bundler?]

* The `bundle install` command will no longer accept a `--binstubs` flag.

  [NOTE: I'm not sure if I got the rationale for this one right, and I couln't
  find any related Github discussion other than an upgrading document in the
  2-0-stable flag that mentions it. Please double check this]

  The `--binstubs` option has been removed from `bundle install` and replaced
  with the `bundle binstubs` command. The `--binstubs` flag would create
  binstubs for all executables present inside the gems in the project. This was
  hardly useful since most users will only use a subset of all the binstubs
  available to them. Also, it would force the introduction of a bunch of most
  likely unused files into source control. Because of this, binstubs now must
  must be created and checked into version control individually.

* The `bundle config` command has a new subcommand-based interface.

  We believe the old interface where the kind of operation was guessed from the
  combination of flags and number of arguments being passed to the command was
  confusing. Instead we have introduced a compulsory subcommand argument that
  can be either `list`, `get`, `set` or `unset`. We believe this will make the
  `config` command much easier to interact with. The old interface is
  deprecated, but we are giving suggestions about the new commands that should
  be used along with the deprecation messages.

* The `bundle inject` command is deprecated and replaced with `bundle add`.

  We believe the new command fits the user's mental model better and it support
  a wider set of use cases. The interface supported by `bundle inject` works
  exactly the same in `bundle add`, so it's should be easy to migrate to the new
  command.

#### Helper deprecations

* `Bundler.clean_env`, `Bundler.with_clean_env`, `Bundler.clean_system`, and `Bundler.clean_exec` are deprecated.

  All of these helpers ultimately use `Bundler.clean_env` under the hood, which
  makes sure all bundler-related environment are removed inside the block it
  yields.

  After quite a lot user reports, we noticed that users don't usually want this
  but instead want the bundler environment as it was before the current process
  was started. Thus, `Bundler.with_original_env`, `Bundler.original_system`, and
  `Bundler.original_exec` were born. They all use the new `Bundler.original_env`
  under the hood.

  There's however some specific cases where the good old `Bundler.clean_env`
  behavior can be useful. For example, when testing Rails generators, you really
  want an environment where `bundler` is out of the picture. This is why we
  decided to keep the old behavior under a new more clear name, because we
  figured the word "clean" was too ambiguous. So we have introduced
  `Bundler.unbundled_env`, `Bundler.with_unbundled_env`,
  `Bundler.unbundled_system`, and `Bundler.unbundled_exec`.

* `Bundler.environment` is deprecated in favor of `Bundler.load`.

  We're not sure how people might be using this directly but we have removed the
  `Bundler::Environment` class which was instantiated by `Bundler.environment`
  since we realized the `Bundler::Runtime` class was the same thing. During the
  transition `Bundler.environment` will delegate to `Bundler.load`, which holds
  the reference to the `Bundler::Environment`.

#### DSL deprecations

* Github, Bitbucket and Gist source use https by default.

  You can still opt out of this, but doing it is deprecated and will no longer
  be supported in future versions of bundler.

* Multiple global Gemfile sources will no longer be supported.

  Using multiple global Gemfile is a security vulnerability because it's
  undefined where each specific dependency will be picked up from, and thus
  someone could publish a malicious gem overriding a well known gem in some of
  the global sources in your Gemfile, and `bundle` might start picking it up.

  So, something like this is deprecated

  ```ruby
  source "https://main_source"
  source "https://another_source"

  gem "dependency1"
  gem "dependency2"
  ```

  Instead, do something like

  ```ruby

  source "https://main_source"

  gem "dependency1"

  source "https://another_source" do
    gem "dependency2"
  end
  ```

#### Misc deprecations

* Deployment helpers for `vlad` and `capistrano` are being removed.

  These are natural deprecations since the `vlad` tool has had no activity for
  years whereas `capistrano` capistrano 3 has built-in Bundler integration in
  the form of the `capistrano-bundler` gem, and everyone using Capistrano 3
  should be using that instead. If for some reason, you are still using
  Capistrano 2, feel free to copy the Capistrano tasks out of the Bundler 2 file
  `bundler/deployment.rb` and put them into your app.

  In general, we don't want to maintain integrations for every deployment system
  out there, so that's why we are removing these.