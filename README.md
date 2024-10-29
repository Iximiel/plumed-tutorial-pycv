# Using PyCV
## Installation

_Note that currently we are working on a simpler installation procedure_

For compiling the plugin you just need pybind11 and numpy.
I always recomend to create and ad-hoc environment for your projects:
```bash
python3 -m venv pycvenv
source ./pycvenv/bin/activate
pip install -U pip
pip install -r requirements.txt
```
The requirements.txt file is in the home of the plug in (`plumeddir/plugins/pycv`)

If you have a plumed that supports plumed mklib with multiple files you can simply
```bash
./standaloneCompile.sh
```
This usually goes smooth, expecially if plumed has already found the necessary python configuration and `plumed --no-mpi config makefile_conf | grep canPyCV` returns `canPyCV=yes`.

After running `./standaloneCompile.sh` you should have a `PythonCVInterface.so` file into the pycv directory

## Examples

All the example are run withing the same environment of the installation procedure
```bash
source ./pycvenv/bin/activate
```
Things to know:
 - I am always using "path/to" for indication the directory that contain a file
 - Python will communicate with plumed using a module called "plumedCommunications" that is embedded with `PythonCVInterface.so`


### Getting the manual

This is a bonus, but can be helpful:

###### plumed.dat

```plumed
LOAD=/path/to/PythonCVInterface.so

cvdist: PYCVINTERFACE IMPORT=pyhelp
```
###### pyhelp.py
```python
import plumedCommunications
import pydoc

def plumedInit(_):
    with open('PythonCVInterface.help.txt', 'w') as f:
        h = pydoc.Helper(output=f)
        h(plumedCommunications.PythonCVInterface)
    with open('plumedCommunications.help.txt', 'w') as f:
        h = pydoc.Helper(output=f)
        h(plumedCommunications)
    with open('plumedCommunications.defaults.help.txt', 'w') as f:
        h = pydoc.Helper(output=f)
        h(plumedCommunications.defaults)
    return {"Value":plumedCommunications.defaults.COMPONENT_NODEV, "ATOMS":"1"}

def plumedCalculate(_):
    return 0
```
Then run with
`plumed driver --plumed plumed.dat --ixyz traj.xyz`

This thing won't run, but you will get the manual of the interface in three txt files.
