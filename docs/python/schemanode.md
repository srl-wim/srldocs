# SR-Linux SchemaNode

## Introduction

A schemaNode is a base building block that describe a data model. Similar to the output of the tree command or the content of a YANG file, they indicate what lists, containers, keys, fields, and leaf- lists can be created.
Show commands use schemaNode as a base building block.

## Import

To build a SchemaNode, start with a FixedSchemaRoot(). 

```python
from srlinux.schema.fixed_schema import FixedSchemaRoot
```

Initializing a variable with the FixedSchemaRoot()

```python
root = FixedSchemaRoot()
```

## Method: add_child

The add_child method allows to add a list/container to the current node, which allows you to specify the keys, fields, and leaf-lists of the new child. Childs can be nested and can use a mix of key, fields and leaf-lists

## Example

Example of a simple datamodel with field

```python
def _get_schema(self):
	root = FixedSchemaRoot()
	root.add_child(
		'<child-name>', 
		fields=[
			'<field1>',
			'<field2>',
			'<field3>'])
	return root
```

Example of a datamodel with keys

```python
def _get_schema(self):
	root = FixedSchemaRoot()
	root.add_child(
		'<child-name>', 
		keys=[
			'<key1>', 
			'<key2>'], 
		fields=[
			'<field1>',
			'<field2>'])
	return root
```

Example of a nested datamodel with keys and fields

```python
def _get_schema(self):
    root = FixedSchemaRoot()
    <var> = root.add_child(
    	'<child1-name>',
		keys=[
			'<key1>', 
			'<key2>'])
    <var>.add_child(
        '<child2-name>',
        fields=[
            '<fielda>', 
            '<fieldb>']
    )
    return root
```
