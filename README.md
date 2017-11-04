# Githook

A ruby gem that help to setup git hooks easily, base on Rake, inspired from Capistrano.

## TODO

- [x] install task
- [x] setup hooks task
- [x] clear hooks task
- [x] disable/enable hooks task
- [x] list hooks task
- [x] version/help task
- [x] pre-commit hook tasks
- [x] prepare-commit-msg hook tasks
- [x] commit-msg hook tasks
- [x] implement as a gem
- [ ] more document

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'git-hook'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install git-hook

## Usage

Help:

    $ githook help
    Usage: githook task_name

    task_name:
      install  -- Init githook, create .githook folder, prepare template files
      setup    -- Setup hooks, copy hooks from .githook/hooks to .git/hooks
      backup   -- Backup old hooks in .git/hooks
      clearup  -- Clear backup hooks in .git/hooks
      disable  -- Disable hooks: [HOOKS=pre_commit,commit_msg] githook disable
      enable   -- Enable hooks: [HOOKS=pre_commit,commit_msg] githook enable
      list     -- List all hooks
      version  -- Version
      help     -- Help

Getting started:

    $ githook install
    $ githook setup

Demos:

1. install, setup hooks

   ![](./art/install_setup_hooks.gif)

1. pre-commit, prepare-commit-msg, commit-msg hooks

   ![](./art/main_hooks.gif)

1. disable, enable, list hooks

   ![](./art/disable_enable_list_hooks.gif)

## Principle

Git will call some hooks when you commit, merge, rebase code, etc. The hooks place in `.git/hooks` folder, there are many default sample hooks, you need remove the `.sample` appendix to make it works if you want to execute the hook. The 3 hooks are most useful for us: pre-commit, prepare-commit-msg, commit-msg.

[See more hooks introduction in git official website - Customizing Git - Git Hooks](https://git-scm.com/book/gr/v2/Customizing-Git-Git-Hooks).

The default behavior of these 3 hooks are a few, but we can add more additional behaviors.

In `pre-commit`, we add the following code:

    # custom pre-commit hooks
    githook pre_commit
    if [ $? -ne 0 ]; then
      exit 1
    fi

We will execute `githook pre_commit` in `pre-commit`, if it fails, it will abort later operations.

What does `githook pre_commit` do? don't forget the githook base on Rake, so the `pre_commit` is a task defined in `tasks/pre-commit.rake`:

    desc 'run all pre-commit hook tasks'
    task :pre_commit do |t|
      Githook::Util.log_task(t.name)
      Githook::Util.run_tasks(t.name.to_sym)
    end

It nearly does nothing but just calls `Githook::Util.run_tasks(:pre_commit)`, let's continue to dive into `run_tasks()`, it is defined in `util.rb`:

    def self.run_tasks(hook_stage)
      tasks = fetch(hook_stage, [])
      tasks.each do |task|
        if Rake::Task.task_defined?(task)
          Rake::Task[task].invoke
        else
          puts "#{task} task doesn't exist."
        end
      end
    end

So it will get the value of `:pre_commit`, if it is an empty array, then nothing will happen, else, it will traverse this array, execute all the tasks in this array one by one.

Where do we define the `:pre_commit` value? It is defined by user in `.githook/config.rb`, let's see how it looks like by default:

    set :pre_commit, fetch(:pre_commit, []).push(
      # uncomment following lines if it is a ruby project
      # 'pre_commit:rubocop',
      # 'pre_commit:rspec',

      # uncomment following lines if it is a java project built by gradle
      # 'pre_commit:checkstyle'
    )
    set :prepare_commit_msg, fetch(:prepare_commit_msg, []).push(
      # comment following lines if you want to skip it
      'prepare_commit_msg:prepare'
    )
    set :commit_msg, fetch(:commit_msg, []).push(
      # comment following lines if you want to skip it
      'commit_msg:check_msg'
    )

We use `set` method to set the value. It seems the `:pre_commit` value is an empty array by default, all items are commented, so `githook pre_commit` in `pre-commit` hook will do nothing, but if your project is a ruby or rails project, there are 2 default tasks are prepared for `:pre_commit`, they are `pre_commit:rubocop` and `pre_commit:rspec`, if you uncomment the above 2 lines code, then `pre_commit:rubocp` and `pre_commit:rspec` will be executed when you commit code, of course, you can define your customized task and add to some hook (later I will introduce how to add a customized task).

How the default tasks `pre_commit:rubocop` and `pre_commit:rspec` looks like, they are defined in `tasks/pre-commit.rake`:

    namespace :pre_commit do
      desc 'check ruby code style by rubocop'
      task :rubocop do |t|
        Githook::Util.log_task(t.name)
        exit 1 unless system("bundle exec rubocop")
      end

      desc 'test ruby code by rspec'
      task :rspec do |t|
        Githook::Util.log_task(t.name)
        exit 1 unless system("bundle exec rspec")
      end

      desc 'check java code style by checkstyle'
      task :checkstyle do |t|
        Githook::Util.log_task(t.name)
        exit 1 unless system("./gradlew checkstyle")
      end
    end

The `pre_commit:rubocop` just simply runs `bundle exec rubocop` command to check ruby code style, while `pre_commit:rspec` runs `bundle exec rspec` to test ruby code.

At last, let's see what do `prepare_commit_msg:prepare` and `commit_msg:check_msg` tasks do?

The `prepare_commit_msg:prepare` is defined in `tasks/prepare-commit-msg.rake`, it is executed in `prepare-commit-msg` hook. It will check whether the commit message is empty, if it is, it will help generate the commit message according the branch name, for example, if the branch name is `feature/24_add_help_task`, then the auto generated commit message is "FEATURE #24 - Add help task".

    namespace :prepare_commit_msg do
      desc 'prepare commit msg'
      task :prepare do |t|
        Githook::Util.log_task(t.name)

        commit_msg_file = Githook::Util.commit_msg_file
        commit_msg = Githook::Util.get_commit_msg(commit_msg_file)
        if Githook::Util.commit_msg_empty?(commit_msg)
          branch_name = Githook::Util.branch_name
          pre_msg = Githook::Util.gen_pre_msg(branch_name)
          puts "pre-msg:"
          puts pre_msg
          Githook::Util.prefill_msg(commit_msg_file, pre_msg)
        end
      end
    end

The `commit_msg:check_msg` is defined in `tasks/commit-msg.rake`, it is executed in `commit-msg` hook after you save the commit message. It will check your commit message style, if it doesn't match the expected format, then this commit will be aborted. In default, our expected commit message summary format is "FEAUTER|BUG|MISC|REFACTOR|WIP #issue_num - Summary", if you want to another format, you need to define yourself task to replace the default behavior.

    namespace :commit_msg do
      desc 'check commit msg style'
      task :check_msg do |t|
        Githook::Util.log_task(t.name)

        commit_msg_file = Githook::Util.commit_msg_file
        commit_msg = Githook::Util.get_commit_msg(commit_msg_file)
        puts "commit-msg:"
        puts commit_msg.join("\n")
        exit 1 unless Githook::Util.check_msg_format?(commit_msg)
      end
    end

## Customize yourself task

TODO:

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
