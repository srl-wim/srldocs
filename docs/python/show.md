# SR-Linux Show Python Plugin framework

## Introduction

SR Linux is an open networking NOS that can be extended in various ways. In this section we focus on the python plugin framework for show commands.

## Show command Plugin

A show Plugin is implemented by redefining the CliPlugin Class with its own methods. The methods define different aspects on how a show command is consumed, like:

*  The cli commands/arguments used as input
*  The data-model the show commands uses
*  How the show commands format the output to the terminal.

To write a CliPlugin the following steps should be followed:

1. Build a schemaNode
2. Upate the CliPlugin and define how to interact with the show command
3. Retrieve the state from the management server
4. Populate the data object 
5. Add Formatter instances to determine how the data will be formatted
6. Implement the callback method to pass the Data structure to the output.print_data command

## Build a schemaNode

Schema nodes describe a data model. Similar to the output of the tree command or the content of a YANG file, they indicate what lists, containers, keys, fields, and leaf- lists can be created.

To build a SchemaNode, start with a FixedSchemaRoot() and then add your top-level list/container using FixedSchemaNode.add_child().

```python
	def _get_my_schema(self):
		root = FixedSchemaRoot() 
		interface = root.add_child(
			'interface',
			key='name',
			fields=[
				'description', 
				'admin-state'])
		child = interface.add_child(
			'child', 
			key='Child-Id', 
			fields=[
				'Is-Cool'])
```

The code above generates a data model for the following YANG model:

```python
list interface {
	key "name";
	leaf "description";
	leaf "admin-state";
	list child {
		key "name";
		leaf "success";
		leaf "failure";
	}
```

More details on the schemaNode can be found [here](schemanode.md)

## Upate the CliPlugin and define how to interact with the show command

Once the data model is defined, the next step is to define the arguments/keys that will be used by the show command. This is done through the add_command which defines:
- The syntax, which defines the command syntax. Command arguments/values
- The callback function that is defining the output, through the next steps.
- The schema that references th  schemaNode that is defined in the previous step 

```python
class Plugin(CliPlugin): 
	'''
		Adds a fancy show report. 
	'''

	def load(self, cli, **_kwargs): 
		cli.show_mode.add_command(
			Syntax('report').add_unnamed_argument('name'), 
			update_location=False,
			callback=self._print, 
			schema=self._get_my_schema(),
		)
```

More details on the add_command can be found [here](command.md).

More details on the syntax plugin can be found [here](syntax.md).

## Retrieve the state from the management server

To retrieve the state, use build_path() to populate a path of the key you need to retrieve, and call get_data.
This returns a Data object pointing to the root of the data returned by the management server:

```python
from srlinux.location import build_path

	def _fetch_state(self, state, arguments):
		path = build_path('/interface[name={name}]/
		subinterface[index=*]', name=arguments.get('name'))
		return state.server_data_store.get_data(path, recursive=True)
```

## Populate the data

With the data from the management server and a data model, populate the Data object:

```python
from srlinux.data import Data from srlinux import strings

	def _populate_data(self, server_data): result = Data(self._get_my_schema())
		for interface in server_data.interface.items(): data = 		result.interface.create(interface.name) data.description = interface.description 		data.admin_state = interface.admin_state
		self._add_children(data, interface.subinterface) return result
	
	def _add_children(self, data, server_data):
		# server_data is an instance of DataChildrenOfType
		for subinterface in server_data.items():
		child = data.child.create(subinterface.index)
		cool_ids = [42, 1337]
		is_cool = subinterface.index in cool_ids child.is_cool = strings.bool_to_yes_no(is_cool)

```

## Add formatter instances

To format the output, assign Formatter instances to the different Data objects. The type of Formatter determines whether the output is formatted using key: value pairs, as a grid-based table, or using a custom format.

```python
from srlinux.data import Border, ColumnFormatter, Data, TagValueFormatter, Borders, Indent
	def _set_formatters(self, data): data.set_formatter(
'/interface', Border(TagValueFormatter(), Border.Above | Border.Below | Border.Between, '='))
data.set_formatter(
'/interface/child',
Indent(ColumnFormatter(ancestor_keys=False, borders=Borders.Header), indenta
tion=2))

```

## Implement the callback method

The following example shows how to implement the callback method which can then be invoked to complete the routine:

```python
	def _print(self, state, arguments, output, **_kwargs): 
		server_data = self._fetch_state(state, arguments) 
		result = self._populate_data(server_data) 		self._set_formatters(result) 
		output.print_data(result)
```