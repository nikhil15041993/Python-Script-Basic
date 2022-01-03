To find few details we will use python module, platform. We will be running this script using python3 interpreter 

```

import platform

# Architecture
print("Architecture: " + platform.architecture()[0])

# machine
print("Machine: " + platform.machine())

# node
print("Node: " + platform.node())

# system
print("System: " + platform.system())

```

## Memory Usage:
Memory details are stored in /proc/meminfo file. The first line is the Total memory in the system and the second line is the free memory available at the moment.

```

# Memory
print("Memory Info: ")
with open("/proc/meminfo", "r") as f:
    lines = f.readlines()

print("     " + lines[0].strip())
print("     " + lines[1].strip())

```
