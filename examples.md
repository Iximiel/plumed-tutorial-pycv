# Examples

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