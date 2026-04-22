# Definition
The name of every git repository should follow this concept:
```
<topic>[-<sub>].<type>[-<sub>].<name>[-<sub>].<language>
```
Parts in square brackets are optional and allow to define multiple terms.

# Options

Possible options and choices.

## Topic
- TCxxxx - relation to TwinCAT Base
- TFxxxx - relation to TwinCAT Function
- TExxxx - relation to TwinCAT Engineering
- tcbsd - TwinCAT/BSD
- linux - Linux
- win10 - Windows 10 (IoT Enterprise 2019/2021/...)

### Sub-Topic
- PLC - TwinCAT PLC
- HMI - TwinCAT HMI
- Motion - TwinCAT Motion

## Type
- `sample` - a sample
- `internal` - not intended to be shared as a whole project with externals
- `library` - a library ready to use
- `utility` - external tool for development and programming support

## Name
any kind of name is allowed
## Language
- `tsproj` - TwinCAT Project
- `plc` - TwinCAT PLC
- `cpp` - C/C++, TwinCAT C++, ...
- `splc` - TwinCAT Safety
- `sh` - Unix Shell Scripts
- `bat`- Windows Batch Scripts
- `ps1` - PowerShell Scripts
- `slx` - MATLAB/Simulink models
- `js` - JavaScript
- `ts` - TypeScript
- `py` - Python
