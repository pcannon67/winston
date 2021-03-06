# Winston, a smart virtual assistant

Winston is a voice-activated virtual assistant meant to be used in home automation projects, among other things. You can ask him questions, interact with him and whatnot. Questions and commands can easily be created in seconds without a deep understanding of how everything works.

## How does it work?

Winston tries as hard as possible to abstract most of the steps below so you can easily create your own commands. Nonetheless, you will still need to understand a few concepts to get started. Winston works this way:

1. GStreamer splits the microphone's input into *utterances* and sends them to pocketsphinx, a speech recognition framework.
2. Pocketsphinx tries to match the utterance with a command in its finite-state grammar (a list of possible commands).
3. Pocketsphinx sends the understood string to Winston's `Listener`, which simply acts as a bridge between pocketsphinx and the application.
4. The `Listener` sends the string to the `Interpreter`, which will match a command string to an actual `Command`.
5. The `Command` performs an action based on the command it was given and its parameters.

### The grammar

The [finite-state grammar](http://cmusphinx.sourceforge.net/wiki/tutoriallm#building_a_grammar) is the only relic from pocketsphinx you will have to deal with. It is generated from a JSGF grammar using `sphinx_jsgf2fsg`, and is used by pocketsphinx to limit the vocabulary to a fixed list of possible commands. By default, the Listener sets the grammar to `winston/grammar.fsg`.

The grammar needs to perfectly mirror what the Interpreter can match to a command. Otherwise, your command simply won't be matched to 

### The Listener

The Listener sets up and starts pocketsphinx and Gstreamer. It then forwards any result from pocketsphinx to its Listener object(s). This is where you can change where the raw commands come from and what to do with them.

### The Interpreter

The Interpreter matches raw commands such as "winston could you please open the door for our guests" to Command objects using regular expressions. In essence, it uses regex generated by Command instances, and calls their callback if they match.

The Interpreter also trims sentence decorations such as "please", "could you" and "would you kindly". If you have to rename Winston, this is also where you'd do it.

A Listener can forward its data to multiple Interpreters.

### The Command

A command does two things:

1. Generate a regex string to match all possible variations of a command string ("set an alarm/wake me up at [time]") for the Interpreter
2. Call a function when the command is matched, possibly with parameters.

Commands have an action (e.g. "Set an alarm at 8:30") and a subject ("8:30"). The subject is a part of the command that will be passed to the callback. Both are used when generating the regex string, and can be made optional.

The `commands.say` command (in commands.say.SayCommand) is very simple example to start with. For a more advanced command example, see the `commands.set_alarm` command.


## Setup

### Requirements

To use Winston, you will need the following packages:

* `gstreamer0.10-pocketsphinx`
* `python-gst0.10`
* `python-gobject`
* `pocketsphinx-hmm-en-hub4wsj`
* `sphinxbase-utils` to use `sphinx_jsgf2fsg`
* `at` for some commands

You will also need `festival` so Winston can reply back to you. If you are using OS X, you can replace the call in utils.texttospeech.text_to_speech command to use the much better `say` command.

If you use virtualenv, make sure you supply the `--system-site-packages` option. This will include the python packages installed with the above packages.

*Important notes: Make sure you have up-to-date packages because some older versions will make your life much harder. Do not hesitate to open an issue or send an email if the instructions are incomplete or unclear. I take these to heart, and will respond as quickly as possible.*

### Configuration options

`config.py` contains all the settings used by Winston.

**`COMMANDS`** contains a list of Commands used by the Interpreter.

**`SCHEDULER`** is the scheduler object that stores timed and repeating events set by commands.


### Creating commands

You can easily add new commands to Winston:

1. Define your own commands by extending or instanciating the commands.Command object. You will find plenty of examples in the commands directory.
2. Add the command to your grammar file (`grammar.fsg` by default, see `config.py`). The easiest way to do this is to edit the `jsgf.txt` grammar, then convert it to .fsg using `sphinx_jsgf2fsg`. Make sure you use words that are already present in the Interpreter's dictionary.
3. Add your command to the COMMANDS list in `config.py`.

Make sure to open `config.py` it contains simple examples that will show you how to define commands. It also defines the path to your dictionary and grammar files.

#### Proactive commands

The default Interpreter offers access to a Scheduler object. This lets you set recurring events (e.g. Read the time hourly or set an alarm) and make Winston a bit more proactive.

To do this, you need to extend the @interpreter.setter property as such:

    @Command.interpreter.setter
    def interpreter(self, value):
        """
        When attaching the interpreter to the command, add some timed events
        to its scheduler.
        """
        Command.interpreter.fset(self, value)  # Call the parent setter

        if self._interpreter:
            # A sample scheduler event that occurs daily at 18:00
            self._interpreter.scheduler.add_cron_job(self.say_balance, hour=18, minute=00)  # Reads the account balance at 17:30, daily

### Running Winston

Once installed, run winston by running `python winston/`. You may also use it in your applications by importing the required modules in your own project. Look at `__main__.py` to see how Winston is started, and what classes come into play.