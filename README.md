# shdotenv

dotenv for shells with support for POSIX-compliant and multiple .env file syntax

![GitHub Workflow Status](https://img.shields.io/github/actions/workflow/status/ko1nksm/shdotenv/ubuntu.yml?branch=main&logo=github)

**Project Status**: Almost complete. Major features have been implemented and v1.0.0 will be released in the near future.

Quoting [bkeepers/dotenv][dotenv]:

> Storing [configuration in the environment](http://12factor.net/config) is one of the tenets of a [twelve-factor app](http://12factor.net). Anything that is likely to change between deployment environments–such as resource handles for databases or credentials for external services–should be extracted from the code into environment variables.

[dotenv]: https://github.com/bkeepers/dotenv

## Why not use `source` or `export`?

It is not safe. There is no formal specification for the .env file syntax, and different languages, libraries, and tools use different syntaxes. If you load a .env file syntax that is incompatible with the POSIX shell syntax, you will get unexpected results and may even result in the execution of scripts.

shdotenv safely loads the syntax of .env files that are compatible with POSIX shell syntax. There is no possibility that the script will be executed. And also, for interoperability, .env files with other syntaxes are supported whenever possible.

## The goals of this project

1. Provide language-independent CLI utilities
2. Provide a library that can safely load .env file from shell scripts
3. Define POSIX shell compatible .env file syntax specification
4. Support for .env file syntax dialects for interoperation

## Requirements

`shdotenv` is a single file shell script with embedded awk script. It uses only the following commands which can be found anywhere.

- POSIX shell (dash, bash, ksh, zsh, etc)
- awk (gawk, nawk, mawk, busybox awk)

## Install

Download `shdotenv` (shell script) from [releases](https://github.com/ko1nksm/shdotenv/releases).

```console
$ mkdir -p "$HOME/bin"
$ wget https://github.com/ko1nksm/shdotenv/releases/latest/download/shdotenv -O "$HOME/bin/shdotenv"
$ chmod +x "$HOME/bin/shdotenv"
```

If you prefer the XDG Base Directory Specification, you can install it under the `$HOME/.local/bin` directory.

```console
$ mkdir -p "$HOME/.local/bin"
$ wget https://github.com/ko1nksm/shdotenv/releases/latest/download/shdotenv -O "$HOME/.local/bin/shdotenv"
$ chmod +x "$HOME/.local/bin/shdotenv"
```

Do not forget to add the installed directory to the `PATH` environment variable.

### Build your own

**Build and install only**

```console
$ git clone https://github.com/ko1nksm/shdotenv.git
$ cd shdotenv
$ make
$ make install PREFIX=$HOME
```

**Full build (test and generate a small build)**

A full build requires requires [shfmt](https://github.com/mvdan/sh), [shellcheck](https://github.com/koalaman/shellcheck) and [shellspec](https://github.com/shellspec/shellspec).

```console
$ git clone https://github.com/ko1nksm/shdotenv.git
$ cd shdotenv
$ make all MINIFY=true
$ make install PREFIX=$HOME
```

**Note for developers**: `shdotenv` can be run in source code without building. Please run `src/shdotenv`.

## Usage

```
Usage: shdotenv [OPTION]... [--] [[COMMAND | export] [ARG]...]

  If the COMMAND is specified, it will load .env files and run the command.
  If the COMMAND is omitted, it will output the result of interpreting .env
  files. It can be safely loaded into the shell (For example, using eval).

Options:
  -d, --dialect DIALECT     Specify the .env dialect [default: posix]
                                posix, ruby, node, python,
                                php, go, rust, docker
  -f, --format FORMAT       Output in the specified format [default: sh]
                                sh, csh, fish, json, jsonl, yaml
  -e, --env ENV_PATH        Location of the .env file [default: .env]
                              Multiple -e options are allowed
                              If the ENV_PATH is "-", read from stdin
  -i, --ignore-environment  Ignore the current environment variables
      --overload            Overload predefined variables
      --no-allexport        Disable all variable export
      --no-nounset          Allow references to undefined variables
      --grep PATTERN        Output only names that match the regexp pattern
  -s, --sort                Sort variable names
  -q, --quiet               Suppress all output (useful for test .env files)
      --version             Show the version and exit
      --help                Show this message and exit

Usage: shdotenv [OPTION]... export [-0ps] [-n | -v] [--] [NAME]...
  Exports environment variables. Default output is POSIX-compliant .env format.

  -0  end each output line with NUL, not newline
  -p  Append "export" prefix to environment variable names
  -s  Empty string instead of error if name is missing
  -n  List environment variable names only
  -v  List environment variable values only

  This will be output after the .env files is loaded. If you do not want
  to load it, specify "-e /dev/null". This is similar to "export", "env"
  and "printenv" commands, but quoting correctly and exports only portable
  environment variable name that are valid as identifier for posix shell.
```

## How to use

### Use as a CLI utility

Set environment variables and execute the specified command.

```sh
shdotenv [OPTION]... <COMMAND> [ARGUMENTS]...
```

#### Test the .env file syntax

```sh
shdotenv --quiet --env .env
```

### Use as a library

Load the .env file into the shell script. When run on the shell, it exports to the current shell.

#### sh, bash, ksh, zsh, etc. (POSIX-compliant shells)

```sh
eval "$(shdotenv [OPTION]...)"
```

You may want to abort the program when the `.env` file fails to parse. In that case, do the following

```sh
eval "$(shdotenv [OPTION]... || echo "exit $?")"
```

#### csh, tcsh

```tcsh
set newline='\
'
eval "`shdotenv -f csh [OPTION]...`"
```

#### fish

```fish
eval (shdotenv -f fish [OPTION]...)
```

### Export environment variables safely

This is similar to `export`, `env` and `printenv` commands, but quoting correctly and exports only portable environment variable name that are valid as identifier for POSIX shell.

```text
shdotenv [OPTION]... export [OPTION]... [NAME]...
```

## How to work with docker

The `docker` command has the `--env-file` option, but it only supports setting simple values without newlines.

- [docker cannot pass newlines from variables in --env-file files](https://github.com/moby/moby/issues/12997)

shdotenv provides a simple solution to this problem.

```sh
shdotenv docker run $(shdotenv -n | sed s/^/-e/) debian sh -c export
```

## .env file syntax

```sh
# dotenv posix
# This line is a comment, The above line is a directive
COMMENT=This-#-is-a-character # This is a comment

UNQUOTED=value1 # Spaces and some special characters cannot be used
SINGLE_QUOTED='value 2' # Cannot use single quote
DOUBLE_QUOTED="value 3" # Some special characters need to be escaped

MULTILINE="line1
line2: \n is not a newline
line3"
LONGLINE="https://github.com/ko1nksm\
/shdotenv/blob/main/README.md"

ENDPOINT="http://${HOST}/api" # Variable expansion requires braces

export EXPORT1="value"
export EXPORT2 # Equivalent to: export EXPORT2="${EXPORT2:-}"
```

- The syntax is a subset of the POSIX shell
- The first line is an optional directive that specifies the dialect of the .env syntax
- No spaces are allowed before or after the `=` separating the name and value
- ANSI-C style escapes are not available (i.e., `\n` is not a newline)
- **Unquoted value**
  - The special characters that can be used are `#` `%` `+` `,` `-` `.` `/` `:` `=` `@` `^` `_`
  - Unquoted '=' not allowed for first character (Add in 0.14.0)
- **Single-quoted value**
  - The disallowed character is: `'`
  - It can contain newline characters
- **Double-quoted value**
  - Variable expansion is available (only `${VAR}` style is supported)
  - The following values should be escaped with a backslash (`\`): `$` <code>\`</code> `"` `\`
  - The `\` at the end of a line value means line continuation
  - It can contain newline characters
- An optional `export` prefix can be added to the name
- Comments at the end of a line need to be preceded by spaces before the `#`

Detailed [POSIX-compliant .env syntax specification](docs/specification.md)

### Directive

Specifies the dotenv syntanx dialect that this `.env` file.

```sh
# dotenv <DIALECT>
```

Example:

```sh
# dotenv ruby
```

### Supported dialects

The formal `.env` syntax for this project is `posix` only. The `posix` is a subset of the POSIX shell and is compatible with shell scripts. Support for other .env syntax dialects is for interoperability purposes. Compatibility will be improved gradually, but is not fully compatible. Reports of problems are welcome.

- docker: [docker](https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e---env---env-file)
- ruby: [dotenv](https://github.com/bkeepers/dotenv)
- node: [dotenv](https://github.com/motdotla/dotenv) + [dotenv-expand](https://github.com/motdotla/dotenv-expand)
- python: [python-dotenv](https://github.com/theskumar/python-dotenv)
- php: [phpdotenv](https://github.com/vlucas/phpdotenv)
- go: [godotenv](https://github.com/joho/godotenv)
- rust: [dotenv](https://github.com/dotenv-rs/dotenv)

[Comparing Dialects](docs/dialects.md)

## .shdotenv

Specifies options for shdotenv. Currently, only `dialect` is supported. It is recommended that the dotenv dialect be specified with the `dotenv` directive. The `.shdotenv` setting is for personal use in projects where it is not allowed.

```
dialect: <DIALECT>
```

Example:

```
dialect: ruby
```

## Environment Variables

| name               | description                             | default |
| ------------------ | --------------------------------------- | ------- |
| SHDOTENV_FORMAT    | Output format (`sh`, `fish`, etc.)      | `sh`    |
| SHDOTENV_AWK       | Path of the `awk` command               | `awk`   |

## FAQ

**Note and reference**: The FAQs present on [motdotla's dotenv](https://github.com/motdotla/dotenv#faq) node project page and [cdimascio's dotenv-java](https://github.com/cdimascio/dotenv-java#faq) project page are so well done that I've included those that are relevant in the FAQs above.

### Q: Should I deploy a .env to e.g. production?

A: Tenant III of the [12 factor app methodology](https://12factor.net/config) states "The twelve-factor app stores config in environment variables". Thus, it is not recommended to provide the .env file to such environments. dotenv, however, is super useful in e.g a local development environment as it enables a developer to manage the environment via a file which is more convenient.

Using dotenv in production would be cheating. This type of usage, however is an anti-pattern.

### Q: Should I commit my .env file?

No. We **strongly** recommend against committing your `.env` file to version control. It should only include environment-specific values such as database passwords or API keys. Your production database should have a different password than your development database.

### Q: What happens to environment variables that were already set?

By default, we will never modify any environment variables that have already been set. In particular, if there is a variable in your `.env` file which collides with one that already exists in your environment, then that variable will be skipped.

If instead, you want to override environment variables use the `--overload` option.

```sh
shdotenv --overload
```

### Q: Why can't I define an environment variable with the same name in the .env file?

We allows multiple .env files for convenience and interoperability with other dotenv tools. However, we believe that being able to use the same name in different .env files will lead to environment variables that are not "fully orthogonal" as [The Twelve-Factor App outlines](https://12factor.net/config).

> In a twelve-factor app, env vars are granular controls, each fully orthogonal to other env vars. They are never grouped together as “environments”, but instead are independently managed for each deploy. This is a model that scales up smoothly as the app naturally expands into more deploys over its lifetime.
>
> – The Twelve-Factor App

If you want to override a previous definition, use the `--overload` option.
