---
layout: post
title: "Parsing Command Line Input"
date: 2014-10-03 11:20:03 -0600
comments: true
categories: coding
---
One of the biggest challenges with a terminal/command line driven interface (CLI) is parsing out the relavent pieces of the input.  For a lot of web and touch based interfaces this kind of input is constrained by input widgets.

We recently completed an 'Event Reporter' project, and it allows the user to interact with a library of data about attendees to an event. Commands would include: `find last_name Smith`, `load data.csv`, `save to results.csv`, or `queue count` and `queue print`.

Ruby provides a capable set of methods for breaking input into strings and arrays, which then allows you to manipulate and extract the parts that are meaningful so they can be passed as messages to the other classes in the application.

The basic CLI requires a REPL loop that Reads input, Evaluates it, Prints a response, and repeats until some condition is met that terminates the loop.

These are examples of how we tackled some of the problems we came up against:

``` ruby The initial REPL loop
  def start
    printer.intro
    until finished?
      printer.command_request
      process_command
    end
    printer.ending
  end
```

In the code snippet above, the two methods on lines 4 & 5 handle the request for input and then pass that input off for processing.

The `process_command` method is a `case` statement that has boolean conditions based on the input received, and for each type of command request, breaks the command down to extract the required content.  For example, the command `load` has two conditions: as a single term that will load a default file, and with a filename that will load a given file.  One strategy looks like this:

``` ruby Parsing the LOAD request
when load?
  if @input.length > 1
    load_file(@input.last)
  else
    load_file
  end
  [...]
end
```

Here we already know that the second (or last) part of the command is going to be the filename, so we use `length` to determine that condition, and then pass the corresponding string out of the `@input` array to the `load()` method.  The `load` method has it's own logic to check if a file exists, and handle malformed file names, since that is not a command parsing responsiblity.

When evaluating the `find` method there are three components to the command, `find`, `criteria`, i.e. state, city, etc., and the `term` to search against, 'CO' or 'Denver'.  However the `term` may be consist of multiple words, such as 'New Orleans'.  Because we decompose the command into an array, split on spaces, the search term needs to be reconsituted.  One way to approach that might look like this:

``` ruby Parsing the FIND request with multi-word term
when find
  if @input.length > 2
    value = input[2..input.length].join(" ")
    @repository.find_by(@input[1], value)
    printer.results(@repository.results_count)
  else
    printer.not_a_valid_command
  end
  [...]
end
```

In a similar strategy to the `load` method, we evaluate how long the input request is.  Then we go after the `criteria` and the `term`.  Criteria is the second element in the array after the command, so that is handled with `input[1]` (remembering that Ruby is zero-indexed).

The search term or criteria is all the remaining string elements of the array, so we get that value on line 3 by re-combining all the array elements from `[input[2]..input.length]` and then `.join(" ")` them with a space between.  The resulting terms are then handled off to the `repository` on line 4 to the `find(criteria, search_term)` method.

As part of the evaluation of our project, all of this parsing logic should eventually be refactored into a separate class of it's own, but that's a topic for another post.
