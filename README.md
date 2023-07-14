# clt

The testing tool allows you to execute any commands, repeat them, and validate whether your results are accurate.

## Installation

The installation of the tool is very simple. All you need to do is clone the repository and start using it!

```bash
git clone https://github.com/manticoresoftware/clt.git
cd clt
./clt help
```

We have prebuilt binaries for use in a Linux environment, for both amd64 and arm64. It's automatically detected based on the platform you're using. Binaries for OSX and Windows are currently not available, but you can build them for your purpose if required. Remember, all tests run in a container environment using Docker, so your machine should have Docker installed.

## Usage

1. Begin by recording your test in interactive mode by executing the `record` command as follows:

	```bash
	./clt record centos:7
	```

	Here, centos:7 is the docker image that you want to use for tests. You can incorporate the `--no-refine` option if you wish to omit the refine step and execute the test later.

2. Next, perform various commands in interactive mode. Once you're done, press `^D` to stop and record your test results. By default, the test results will be saved in the `test.rec` file.

3. To validate and replay it, execute the following command:

	```bash
	./clt test -d centos:7
	echo $?
	```

	This will display the exit code and print the diff (if outputs vary while you replay it). If the exit code equals 1, everything is fine. Otherwise, you should examine the diff created by the cmp tool and introduce the necessary changes to the file, provided you executed it with the `--no-refine` option.

	The `-d` flag is used for debug output and displays the diff, not just the exit code.

	You can locate the complete output replayed in the same file name but with a .rep extension. In this case, it's test.rep.

## GitHub Workflow example

CLT provides a prearranged workflow for use in GitHub actions to run all tests located in your `tests` folder. You can find the template in `.github/workflows/clt-template.yml`. Here is a simple `clt.yml` example for use in your project:

```yaml
name: CLT tests

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  clt:
    uses: manticoresoftware/clt/.github/workflows/clt-template.yml@v2
    with:
      docker_images: |
        [
          {"image": "ghcr.io/manticoresoftware/manticoresearch:clt-tmp-47146bd"},
          {"image": "manticoresearch/manticore-executor-kit:0.6.9"}
        ]
      init_code: |
        git clone https://github.com/manticoresoftware/phar_builder.git
        ./phar_builder/bin/build --name="Manticore Buddy" --package="manticore-buddy"
      run_args: |
        -v $PWD/build/share/modules/manticore-buddy:/usr/local/share/manticore/modules/manticore-buddy
        -v $PWD/build/manticore-buddy:/usr/local/share/manticore/modules/manticore-buddy/bin/manticore-buddy
      version: 'v2'
```

The code is self-explanatory and covers almost everything that the current template offers.

## Developers section

### How to build rec and cmp tools

Build aarch and amd64 static cross for Linux:

```bash
./bin/cross-build
git add .
git commit -m '...'
```

### How to make output readable

This one-liner removes control characters and console codes/colors from recorded terminal output:

```bash
cat output.rec | sed -e "s/\x1b\[.\{1,5\}m//g"
```

### Current limitations

- Use `^D` only once when closing your `clt` environment; for other exits, use `exit`.
- Avoid using `^C`, `^V`, `^Z`, and other magic controls as they might not function correctly.
- Reverse search (`^R`) is currently unsupported, so do not use it to record.
- Complete your tests in the `clt> ` shell and press `^D` to terminate the session and finish recordings.
- Use a simple default terminal, like iTerm. Using the VS Code terminal may lead to unusual behavior due to various settings.
- Currently, there is a limitation that requires entering commands only when the prompt is available. This can lead to inaccurate result validation due to the TTY workflow.

Not all keyboard and Bash controls are supported. Here is the list of supported keystrokes:

- Input characters from the keyboard.
- Left and right arrows.
- Backspace and delete.
- CTRL+a and CTRL+e

### The replay flow

To re-run and confirm that tests are successful, we look for .rec files and compile them into a ready-to-use version on the fly. We then utilize these compiled versions to execute each command in sequence, generating a .rep file. Subsequently, we use the cmp tool to compare the results with the compiled version of the .rec files.

### File Extension Description

There are several types of files:
| Extension | Description |
|-|-|
| .rec | Original record of the input commands and their outputs. It may contain links to block files. |
| .recb | Record block file, contains reusable blocks that can be included in .rec files. |
| .rep | Replay file that contains the results of replaying the .recc file. |
