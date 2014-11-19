I am doing Test Driven Development on a Groovy/Grails project. I use IntelliJ for my IDE and there was a time when working this way and running tests in IntelliJ would be fine, but I got spoiled. For the last year I have been working with AngularJS and using Karma to run my Jasmine JavaScript unit tests. These tests ran fast and continuously. Now every time I run my unit tests for Grails in IntelliJ I am waiting and can feel how the delay impacts my flow. By using grails interactive TMUX and Gaurd, I was able to produce the same results.

## Step one - Grails Interactive
Running the tests in IntelliJ or by typing in 
```bash
grails test-app unit:
```
was just to slow. It starts up the container every time which took at least 15 seconds. This doesn't seem like a lot, but it adds up over the course of the day. By using grails in interactive mode the start up time is removed and the tests complete faster. Once in interactive mode the tests command can be repeated by pressing the up arrow. This was the first step, but chaning to the terminal window to hit up arrow after every change was taking to much time also. Now to make them run automatically when I save a file so I don't have to change windows.

## Step Two - Watch Mode
To accomplish watch mode I had to use a few tools. The first thing is to watch the files and to execute a command when something changes. To do this I used the Ruby tool [Guard](https://github.com/guard/guard). I set up a guard file with the following
```ruby
guard :shell do
  watch /.groovy/ do |m|
   n m[0], 'Changed'
        `tmux send-keys -t :1.1 'test-app unit: ' C-m`
  end
end
```
This file watches all groovy files under the directory and then executes
``` bash 
tmux send-keys -t :1.1 'test-app unit: ' C-m
```
The tmux send-keys command acts like I typed the command myself in the session making it execute the test-app every time a file is saved.
tmux and tmuxinator

In order for the above command to work, I need to have a tmux session set up. I use tmux with tmuxinator to create the session. My tmuxinator file for create my session looks like this
```yaml
# ~/.tmuxinator/server.yml
name: server_dev
root: ~/Documents/workspace/server 
windows:
  - testing:
      layout: main-horizontal
      panes:
        - grails:
          - grails
        - guard -G ../guardFile
        - #empty, just shell
```
This layout gives me three panes. Pane one the grails interactive pane, pane two launches Guard, pane three is an shell prompt in case anything needs to be run.

One note about the Gurad file and the tmuxinator script, I changed my tmux settings to make the windows and panes have 1 based lists instead of 0 based lists. If you do not modify this setting you will need to modify the Guardfile for this.

This set up works great, although, when I am working on one test, it would be nice if only that test was run, and not the whole suite. To do this I modified the guardfile to this
``` ruby
guard :shell do
  watch /.groovy/ do |m|
   n m[0], 'Changed'
    if(! ENV['GRAILS_TEST'] or ENV['GRAILS_TEST'].nil?)
  `tmux send-keys -t :1.1 'test-app unit:' C-m`
 else
  `tmux send-keys -t :1.1 'test-app unit: #{ENV["GRAILS_TEST"]}' C-m`
 end
  end
end
```
The change here is to look for an environment variable in the guard session, if it is set, it will add that test to the send keys command and only that test will run. Setting the environment variable in the guard prompt is a bit verbose, fortunately guard runs using Pry, that allows me to create commands. I created two commands

* setTest - takes a test name and will set it to the env variable
* clearTest - clears the variable and will make all the tests run again

To create the pry commands for guard you need to modify the .guardrc file in your home directory. I added the following
```ruby
Pry::Commands.block_command "setTest", "Set a specific test to run from guard" do |x|
  ENV['GRAILS_TEST'] = x
  output.puts "Guard will only run #{x} now"
end
Pry::Commands.block_command "clearTest", "Clears out specific test so all are run" do
  ENV['GRAILS_TEST'] = ''
  output.puts "Guard will run all unit tests now"
end 
```

## Mission Complete

Now I have my grails tests running every time I save a file. I can specify which tests to run and they execute quickly.