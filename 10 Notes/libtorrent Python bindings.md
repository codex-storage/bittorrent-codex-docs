
> If you are here because you are trying to build Deluge client, before proceeding, please check build instructions in [[Deluge]] regarding the location of the python virtual environment folder (`.venv` in this case).

To build a debug version with python bindings, first (re)build and (re)install the debug target:

```bash
b2 crypto=openssl cxxstd=14 debug
sudo b2 install --prefix=/usr/local
```

Then check [[Modern Python Tools|Python Tools]], and install `uv`:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
```

Then create a virtual env indicating the python version you want to use:

```bash
uv venv --python 3.12.3
```

It will create a folder called `.venv` in your current folder. 

Activate the environment:

```bash
source .venv/bin/activate
```

> Make sure you have the correct path to the `.venv` folder, If you are not in the folder directly containing `.venv` folder.

Then, install `setuptools`:

```bash
uv pip install setuptools
```

Also make sure that you have:

```bash
sudo apt install libboost-python-dev
```

as specified in [libtorrent python binding](http://www.libtorrent.org/python_binding.html) in *prerequisites*.

Finally, from `bindings/python` run:

```bash
python setup.py build_ext --b2-args=variant=debug install
```

Python bindings should be ready to use. You should be able to test it with:

```bash
python
Python 3.12.3 (main, Sep 11 2024, 14:17:37) [GCC 13.2.0] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import libtorrent
>>>
```
