There are some [[Modern Python Tools]], but I tend to rely on the good old [pyenv](https://github.com/pyenv/pyenv) and native python environments.

### Python Version

I use `pyenv` to switch between python versions.
I installed it with:

```bash
brew install pyenv
```

Using `pyenv` is convenient.

You can check existing versions with:

```bash
# checks the most recent available version starting with 3
pyenv latest -k 3

# checks the most recent available version starting with 3.12
pyenv latest -k 3.12
```

Now to install the chosen python version, just run:

```bash
pyenv install 3.13.1
```

When you run a Python command, `pyenv` will look for a `.python-version` file in the current directory and each parent directory. If no such file is found in the tree, `pyenv` will use the global Python version specified with `pyenv global`. A version specified with the `PYENV_VERSION` environment variable takes precedence over local and global versions.

Run `pyenv versions` for a list of available Python versions. To read more, run `pyenv local --help`.

Running `pyenv local 3.13.1` will create the `.python-version` file in the current directory.

### Python Environment

On my Mac I keep python environments in `~/python-venvs`. Now to create a new environment, run:

```bash
python -m venv ~/python-venvs/<name-of-the-environment>
```

To activate and deactivate given python environment, run:

```bash
# to activate
source ~/python-venvs/<name-of-the-environment>/bin/activate

# to deactivate 
deactivate
```
