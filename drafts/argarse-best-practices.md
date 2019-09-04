# Argparse best practices

Building command-line tools in Python the right way.

## Introduction

It's hard to imagine a professional developer who doesn't have to write a command line tool at some point.

In python, the in-built library provides [argparse](https://docs.python.org/3/library/argparse.html).

After building, reading and maintaining a number of these command line tools, here is HotCode's best practice for writing a command line tool using argparse.

### Note

* This is not an introduction to argparse or a tutorial. The stand libraries tutorial is [here](https://docs.python.org/3/howto/argparse.html)
* The code here is python 2/3 independent.

## Code overview

```python
# stdlib
import argparse
import time
# 3rd party
from argsupport import act_if, handler

### WORKER FUNCTIONS ###
def bake_for(minutes=None):
    if minutes:
        print "Baking for %dmins" % minutes

def fancy_decor():
    """Stupidly simple"""
    print "Fancified goods"


def box_decor():
    """Stupidly simple"""
    print "Boxed goods"


### SUBCOMMAND FUNCTIONS ###
def bake_executor(minutes=0):
    bake_for(minutes=minutes)


def decor_executor(fancy=False, box=False):
    act_if(fancy, fancy_decor,
           descript="Adding fancy decoration")
    act_if(box, box_decor,
           descript="Boxing goods.")


# A handler extracts the properties on the namespace into
# kwargs passed to the executor.
def bake_handler(args_namespace):
    return handler(args_namespace, ["minutes"], bake_executor)


def decor_handler(args_namespace):
    return handler(args_namespace, ["fancy", "box"], decor_executor)


### COMMAND LINE TOOLING ###
def setup_cmd_parser():
    """Setup commandline options and returns the parser."""
    parser = argparse.ArgumentParser()
    # Global options
    parser.add_argument("--no-logging", action="store_true",
                        help="Disable logging.")

    # Subcommands
    subparsers = parser.add_subparsers(title="subcommands")

    # Baking subcommand
    bake_parser = subparsers.add_parser("bake",
                                        help="Start baking biscuits.")
    bake_parser.add_argument("--minutes",
                             help="Number of minutes to bake for.""")
    bake_parser.set_defaults(func=bake_handler)

    # Decorate subcommand
    decor_parser = subparsers.add_parser("decorate",
                                         help="Decorate baked goods.")
    decor_parser.add_argument("--fancy", action="store_true",
                              help="Apply fancy decoration")
    decor_parser.add_argument("--box", action="store_true",
                              help="Box the goods.")
    decor_parser.set_defaults(func=decor_handler)

    return parser

def execute_cmdline():
    """Parses commandline and executes the commands."""
    parser = setup_cmd_parser()
    args = parser.parse_args()

    # Throw help if nothing specified.
    if len(sys.argv) < 2:
        parser.print_help()
        sys.exit(0)

    # Handle globals
    if args.no_logging:
        # Disable any message at info level or below.
        LOGGER.setLevel(logging.INFO)
    else:
        logging.basicConfig(level=logging.DEBUG)

    # Whatever subcommand was specified, call it's function.
    args.func(args)

if __name__ == "__main__":
    execute_cmdline()

```

Full sample

## Main risks

### Argparse is procedural/imperative

The library isn't friendly to functional programmers. The interfaces of the library encourage a procedural implementation. This means that the code to create and setup the ArgumentParser often is written as one long function with a lot of complex state interactions. These can become beastly to maintain.

#### Interface problem

Perhaps you want to write some functions that will maniuplate the argument parser? It becomes difficult because there are no methods to get the things setup on the argument parser object. Once the parser or argument is created on the ArgumentParser object, it's hard to find it. Normally, coder just hangs on to the returned value as a local variable.

For example, when you create a new subparser, there's no function that let's you get back the subparser. (Unless you go poking in the internals.)

Poking in the internals example:

```python
import argparse

# Horrid global but this is just quick-and-easy example code.
PARSER = None

def get_parser():
    global PARSER
    if not PARSER:
        PARSER = argparse.ArgumentParser(description="Biscuit factory")
    return PARSER

def get_subparsers(parser):
    """Provides a way to get the subparser"""
    if not parser._subparsers:
        parser.add_subparsers()
    return parser._subparsers._actions[1]

def add_bake():
    """Add the bake subcommand to the parser"""
    bake = get_subparsers(parser=get_parser())).add_parser('bake')

add_bake()
```

Normally, coders just write it in one long function:

```python
import argparse

def main():
    parser = argparse.ArgumentParser(description='Biscuit factory')
    subparsers = parser.add_subparsers()
    bake = subparsers.add_parser('bake')

```

This looks fine to begin with, but it's hard to digest main() into small functions, and this becomes potentially problematic as the command line tool grows.

#### Limited code re-use; harder test-coverage

Command line tools are useful. That's why we write them. But the procedural inclinations of argparse encourages programmers to forget that the code within the command line tool may be useful to other python code. It's easy to write a command-line tool that contains lots of useful functionality, but that functionality is _only_ accessible through the command line. While other code can issue subprocesses to re-use the functionality, this is often a poor subsitute for being able to call a native python function.
