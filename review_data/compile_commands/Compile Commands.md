# Compile Commands
Compile command files are .json files containing compilation information. They are used by static analysis tools to reproduce builds, e.g., to set C/C++ define flags and include paths.

The file itself is a JSON file with the following format:

```JSON
[
	{
		"command": "<build_command>",
		"directory": "<build_directory",
		"file": "<file being built>"
	},
	
	{
		"command": "<build_command>",
		"directory": "<build_directory",
		"file": "<file being built>"
	},
	...
	,
	{
		"command": "<build_command>",
		"directory": "<build_directory",
		"file": "<file being built>"
	}
]
```