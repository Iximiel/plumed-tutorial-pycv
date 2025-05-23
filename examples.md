# Examples

All the example are run within the same environment of the installation procedure
```bash
source ./pycvenv/bin/activate
```
Things to know:
 - I am always using "path/to" for indication the directory that contain a file
 - Python will communicate with plumed using a module called "plumedCommunications" that is embedded with `PythonCVInterface.so`
 - The string `path/to/PythonCVInterface.so` can be obtained by running `python -m pycv`
 - The string `path/to/PythonCVInterface.so` can be obtained within a python script with 
 ```python
 from pycv import getPythonCVInterface;print(getPythonCVInterface())
 ```


### Getting the manual

This is a bonus, but can be helpful:

###### plumed.dat

```plumed
LOAD FILE=/path/to/PythonCVInterface.so

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
    with open('PythonFunction.help.txt', 'w') as f:
        h = pydoc.Helper(output=f)
        h(plumedCommunications.PythonFunction)
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

This thing will not have any results on the trajectory, but you will get the manual of the interface in a few txt files.

Or alternatively you can run:
###### pyhelp_html.py
```python
import plumedCommunications
import pydoc

def plumedInit(_):
    pydoc.writedoc(plumedCommunications)
    pydoc.writedoc(plumedCommunications.defaults)
    return {"Value":plumedCommunications.defaults.COMPONENT_NODEV, "ATOMS":"1"}

def plumedCalculate(_):
    return 0
```
To get an extreme bare-bones HTML documentation, that you can browse 
by running `python -m http.server` and then going to the address that appears on the terminal, that usually is 0.0.0.0:8000, with your internet browser.

### Distance (with PBC)

In this example we show how to use the PBCs in a calculation:

###### plumed.dat
```plumed
LOAD GLOBAL FILE=path/to/PythonCVInterface.so

cvPY: PYCVINTERFACE ATOMS=1,4 IMPORT=pydistancePBCs CALCULATE=pydist

cvCPP: DISTANCE ATOMS=1,4

PRINT FILE=colvar.out ARG=*
```
###### pydistancePBCs.py
```python
import numpy as np
import plumedCommunications

plumedInit={"Value":plumedCommunications.defaults.COMPONENT_NODEV}

def pydist(action: plumedCommunications.PythonCVInterface):
    at: np.ndarray = action.getPositions()

    d = at[0] - at[1]
    d = action.getPbc().apply([d])
    assert d.shape[1] == 3, "d is not a (*,3) array"
    d = np.linalg.norm(d[0])

    return d

```
Here we are getting the pbc calculator from plumed and "patching" the distances with it by using the method `apply` like in C++ plumed, but on a `list/np.ndarray` that must be shaped `[nat,3]`

### Periodicity

This is just an example on how the user can work with the periodicity of the retuned variables
###### plumed.dat
```plumed
LOAD GLOBAL FILE=path/to/PythonCVInterface.so

cvPY: PYCVINTERFACE IMPORT=periodicity

PRINT FILE=colvar.out ARG=*
```

###### periodicity.py
```python
import plumedCommunications as PLMD
import numpy

plumedInit = dict(
    COMPONENTS=dict(
        nonPeriodic=dict(period=None),
        Periodic={"period": ["0", "1.3"]},
        PeriodicPI={"period": ["0", "pi"]},
    ),
    ATOMS="1,2",
)

def plumedCalculate(action: PLMD.PythonCVInterface):
    ret = {
        "nonPeriodic": action.getStep(),
        "Periodic": action.getStep(),
        "PeriodicPI": action.getStep(),
    }

    return ret
```

### Getting simulation information

###### plumed.dat
```plumed
LOAD GLOBAL FILE=path/to/PythonCVInterface.so

cvPY: PYCVINTERFACE ATOMS=2,4 IMPORT=mdInfo PREPARE=myPrepare

PRINT FILE=colvar.out ARG=*
```

###### mdInfo.py
```python
import plumedCommunications as PLMD

def plumedInit(action: PLMD.PythonCVInterface):
    myPrint(f"action label: {action.label}")
    return{"Value": PLMD.defaults.COMPONENT_NODEV,}

def myPrepare(action: PLMD.PythonCVInterface):
    print(f"@step {action.getStep()}")
    print(f"{action.getTime()=}")
    print(f"{action.getTimeStep()=}")
    print(f"{action.isRestart()=}")
    print(f"{action.isExchangeStep()=}")
    return {}
    
def plumedCalculate(action: PLMD.PythonCVInterface):
    return 0.0
```

### Using PyCV when in a plumed-python script

```python
from plumed import Plumed
def preparePlumedWithPyCV(num_atoms: int):
   from pycv import getPythonCVInterface

   # Create PLUMED object and read input
   plmd = Plumed()

   # not really needed, used to check https://github.com/plumed/plumed2/issues/916
   plumed_version = np.zeros(1, dtype=np.intc)
   plmd.cmd("getApiVersion", plumed_version)
   plmd.cmd("setMDEngine", "python")
   plmd.cmd("setTimestep", 1.0)
   plmd.cmd("setKbT", 1.0)
   plmd.cmd("setNatoms", num_atoms)
   plmd.cmd("setLogFile", "test.log")
   plmd.cmd("init")
   plmd.cmd("readInputLine", f"LOAD FILE={getPythonCVInterface()}")
   return plmd
```
