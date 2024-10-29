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

### The basics

When using pycv the user will need to create a module[^1] that must at least contain an initialization dictionary and a calculate function

[^1]: a module is a `.py` file or a directory that contains an `__init__.py`, here we will only show `py` files

#### What to write in the plumed.dat
  - `IMPORT` this is the only mandatory keyword, this indicates the python module to load
  - `INIT` indicates the function to call at initialization. Defaults to `plumedInit`
    - `COMPONENTS` can be specified either in python or in the plumed.dat 
    - `ATOMS`, `GROUPA`,`GROUPB`  can be specified either in python or in the plumed.dat
  - `NOPBC`, `PAIR`,`NLIST`,`NL_CUTOFF`,`NL_STRIDE` can only specified int the plumed.dat
  - `CALCULATE` indicates the function to call at calculate time. Defaults to `plumedCalculate`
  - `PREPARE` indicates the function to call at prepare time. Ignored if not specified
  - `UPDATE` indicates the function to call at update time. Ignored if not specified

#### Initialization

If in the plumed.dat `INIT` is not specified, plumed will search for an object "plumedInit",
that can be either a function that returns a dict or a dict
This dict MUST contain at least the informations about the presence of the
derivatives and on the periodicity of the variable.
We will refer to this dict as "the _init dict_" from now.

The _init dict_ will tell plumed how many components the calculate function will
return and how they shall behave.
Along this the dict can contain all the keyword that are compatible with
PYCVINTERFACE.
Mind that if the same keyword is specified both in the _init dict_ and in the
plumed file the calculation will be aborted to avoid unwated settings confict.
In case of flags the dict entry must be a bool, differently from the standard
plumed input.



The only keyword that can only be specified in python is `COMPONENTS`.
The `COMPONENTS` key must point to a dict that has as keys the names of the
componensts.
Each component dictionary must have two keys:
 - `"period"`: `None` of a list of two values, min and max (like `[0,1]` or also
 strings like `["0.5*pi","2*pi"]`)
 - `"derivative"`: `True` or `False`
If you want to use a single component you can create the `"COMPONENTS"` dict
with as single key, the name will be ignored.
In the previous example the key `"Value"` is used instead of `"COMPONENTS"`:
it is a shorter form for `"COMPONENTS":{"any":{...}}`.
To avoid confusion you cannot specify both `"COMPONENTS"` and `"Value"` in the
 same dict.

To speed up the declarations of the components the `plumedCommunications` module
contains a submodule `defaults` with the default dictionaries already set up:
 - plumedCommunications.defaults.COMPONENT={"period":None, "derivative":True}
 - plumedCommunications.defaults.COMPONENT_NODEV={"period":None, "derivative":False}

#### Calculate

If `CALCULATE` is not specified, plumed will search for a function named
"plumedCalculate" plumed will read the variable returned accordingly to what it
was specified in the initialization dict.

The calculate funtion must, as all the other functions accept a
PLMD.PythonCVInterface object as only input.

The calculate function must either return a float or a tuple or, in the case of
multiple components, a dict whose keys are the name of the components, whose
elements are either float or tuple.

Plumed will assign automatically the result to the CV (to the key named
element), if the name of the component is missing the calculation will be
interrupted with an error message.
If derivatives are disabled it will expect a float(or a double).
In case of activated derivatives it will interrupt the calculation if the
return value would not be a tuple.
The tuple should be (float, ndArray(nat,3),ndArray(3,3)) with the first
elements the value, the second the atomic derivatives and the third the box
derivative (that can also have shape(9), with format (x_x, x_y, x_z, y_x, y_y,
y_z, z_x, z_y, z_z)), if the box derivative are not present a WARNING will be
raised, but the calculation won't be interrupted.

#### Prepate

If the `PREPARE` keyword is used, the defined function will be called at
prepare time, before calculate.
The prepare dictionary can contain a `"setAtomRequest"` key with a parseable
ATOM string, like in the input (or a list of indexes, 0 based).
```python
#this , with "PREPARE=changeAtom" in the plumed file will select a new atom at each new step
def changeAtom(plmdAction: plumedCommunications.PythonCVInterface):
    toret = {"setAtomRequest": f"1, {int(plmdAction.getStep()) + 2}"}
    if plmdAction.getStep() == 3:
        toret["setAtomRequest"] = "1,2"
    return toret
```
#### Update

If the `UPDATE` keyword is used, the defined function will be called at update
time, after calculate. As now plumed will ignore the return of this function
(but it stills need to return a dict) and it is intended to accumulate things
or post process data afer calculate

In the example `plmdAction.data["pycv"]=0` is intialized in `pyinit` and its
value is updated in calculate.


#### The `data` attribute
The plumedCommunications.PythonCVInterface has a `data` attribute that is a
dictionary and can be used to store data during the calculations

```plumed
cv1: PYCVINTERFACE  ...
  ATOMS=@mdatoms
  IMPORT=pycvPersistentData
  CALCULATE=pydist
  INIT=pyinit
...

PRINT FILE=colvar.out ARG=*
```

```python
import plumedCommunications as PLMD
from plumedCommunications.defaults import COMPONENT_NODEV

def pyinit(plmdAction: PLMD.PythonCVInterface):
    plmdAction.data["pycv"]=0
    print(f"{plmdAction.data=}", file=log)
    return {"Value":COMPONENT_NODEV}

def pydist(plmdAction: PLMD.PythonCVInterface):
    plmdAction.data["pycv"]+=plmdAction.getStep()
    d=plmdAction.data["pycv"]
    return d
```
