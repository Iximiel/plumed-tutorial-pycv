# Using PyCV
## Installation

It should be sufficient to run, ideally in a virtual or conda environment.
From the plumed source directory run:
```sh
cd plugins/pycv
pip install .
```
You will need to have plumed avaiable in your path or through `pkg-config`

Required dependencies `numpy` and `pybind11` are installed as part of the installation process.

>[!NOTE]
>Note that an in-place installation, `pip install -e .`, won't work. 

>[!NOTE]
>To get the position of the shared object to load with the LOAD action, call `python -m pycv`, see [the examples](examples.md)

### Known runtime problems

On some platforms, *embedded* Python interpreters  (such as the one used in PYCV) appear to behave 
differently than the plain ones, raising surprising errors.  For example:

>[!NOTE]
>Some Python configurations (e.g. Conda under Linux) require the
 Python shared library to be found in the LD_LIBRARY_PATH, 
 ignoring the activated environment.  This manifests itself with an
 error like:

```
      libpython3.13.so.1.0: cannot open shared object file: No such file or directory
```

>[!NOTE]
>Similarly, some Python configurations (e.g. MacOS) ignore the current
 environment when searching for packages (e.g. `numpy`). Hence,
 one should set PYTHONPATH manually.  This manifests itself with an
 error like:

```
      No module named numpy
```

## The basics

When using PyCV the user will need to create a module[^1] that must at least contain an initialization dictionary and a calculate function

[^1]: a module is a `.py` file or a directory that contains a `__init__.py`, here we will only show `py` files

### What to write in the plumed.dat (PYCVINTERFACE)

```plumed
cvPY: PYCVINTERFACE ATOMS=1,4 IMPORT=pydistancePBCs CALCULATE=pydist
```

  - `IMPORT` this is the only mandatory keyword, this indicates the python module to load
  - `INIT` indicates the function to call at initialization. Defaults to `plumedInit`
  - `COMPONENTS` can be specified either in python or in the plumed.dat 
  - `NOPBC` can be specified either in python or in the plumed.dat
  - `ATOMS`, `GROUPA`,`GROUPB`, using `GROUP*` will make plumed set up a neighbor list between the groups, or group A, whereas with ATOMS all the atom will be passed 
  - `PAIR`,`NLIST`,`NL_CUTOFF`,`NL_STRIDE` can only be specified in the plumed.dat ~to be verified~
  - `CALCULATE` indicates the function to call at calculate time. Defaults to `plumedCalculate`
  - `PREPARE` indicates the function to call at prepare time. Ignored if not specified
  - `UPDATE` indicates the function to call at update time. Ignored if not specified

### What to write in the plumed.dat (PYFUNCTION)
```plumed
d1: DISTANCE ATOMS=1,2
d2: DISTANCE ATOMS=1,3 

fPY: PYFUNCTION IMPORT=pycvfunc CALCULATE=plumedCalculate ARG=d1,d2
```

  - `IMPORT` this is the only mandatory keyword, this indicates the python module to load
  - `INIT` indicates the function to call at initialization. Defaults to `plumedInit`
    - `COMPONENTS` can be specified either in python or in the plumed.dat 
  - `CALCULATE` indicates the function to call at calculate time. Defaults to `plumedCalculate`
  - `ARG` the list of arguments, to be specified in the plumed.dat

PYFUNCTION is simpler: but needs to get the arguments from the plumed object

```python
import plumedCommunications as PLMD
import numpy

plumedInit={"Value": PLMD.defaults.COMPONENT_NODEV}

def plumedCalculate(action: PLMD.PythonFunction):
    arg = [action.argument(0),action.argument(1)]
    return arg[0]*arg[0]*arg[1]
```
### The functions

For both PYFUNCTION and PYCVINTERFACE the function called by plumed must accept a specific object.
    - PYCVINTERFACE functions will expect a `plumedCommunications.PythonCVInterface` object
    - PYFUNCTION functions will expect a `plumedCommunications.PythonFunction` object

These objects are used to retrieve data and settings from plumed. Plumed will get data back with the `data` attribute (see [below](#the-data-attribute)) but mainly with the use of returned dictionaries.

In the [examples](examples.md#getting-the-manual) I show how to retrieve the manual for those objects


### Initialization

If in the plumed.dat `INIT` is not specified, plumed will search for an object "plumedInit",
that can be either a function that returns a dict or a dict
This dict MUST contain at least the information about the presence of the
derivatives and on the periodicity of the variable.
We will refer to this dict as "the _init dict_" from now.

The _init dict_ will tell plumed how many components the calculate function will
return and how they shall behave.
Along this the dict can contain all the keyword that are compatible with
PYCVINTERFACE.
Mind that if the same keyword is specified both in the _init dict_ and in the
plumed file the calculation will be aborted to avoid unwanted settings conflicts.
In case of flags the dict entry must be a boolean, differently from the standard
plumed input.


The only keyword that can only be specified in python is `COMPONENTS`.
The `COMPONENTS` key must point to a dict that has as keys the names of the
components.
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
 - `plumedCommunications.defaults.COMPONENT={"period":None, "derivative":True}`
 - `plumedCommunications.defaults.COMPONENT_NODEV={"period":None, "derivative":False}`

### Calculate

If `CALCULATE` is not specified, plumed will search for a function named
"plumedCalculate" plumed will read the variable returned accordingly to what it
was specified in the initialization dict.

The calculate function must, as all the other functions accept a
`plumedCommunications.PythonCVInterface` (or a `plumedCommunications.PythonFunction` in case of a pyfunction) object as only input.

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

### Prepare

If the `PREPARE` keyword is used, the defined function will be called at
prepare time, before calculate.
The returned "prepare" dictionary can contain a `"setAtomRequest"` key with a parseable
ATOM string, like in the standard plumed input (or a list of indexes, 0 based).
```python
#this , with "PREPARE=changeAtom" in the plumed file will select a new atom at each new step
def changeAtom(plmdAction: plumedCommunications.PythonCVInterface):
    toret = {"setAtomRequest": f"1, {int(plmdAction.getStep()) + 2}"}
    if plmdAction.getStep() == 3:
        toret["setAtomRequest"] = "1,2"
    return toret
```
### Update

If the `UPDATE` keyword is used, the defined function will be called at update
time, after calculate. As now plumed will ignore the return of this function
(but it stills need to return a dict) and it is intended to accumulate things
or post process data afer calculate

In the example `plmdAction.data["pycv"]=0` is initialized in `pyinit` and its
value is updated in calculate.


### The `data` attribute
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
