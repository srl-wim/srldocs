# SR-Linux DataChildrenOfType class

## Introduction

Use this class when you need to access children through a [Data](data.md) object. It allows you to retrieve, create, and iterate all children

## Usage

```python
Class srlinux.data.data.DataChildrenOfType(schema, parent)
```

The children of a Data object are returned by Data.get_children() or by accessing the attribute with the child name (for example, Data.interface). Most methods require you to pass a value for each key defined in the schema.

*schema without keys*

```python
data.node.get() 
data.node.create() 
data.node.exists()
```

*schema with a single key*

```python
data.node.get('abc') 
data.node.create('abc') 
data.node.exists('abc')
```

or

```python
data.node.get(name='abc') 
data.node.create(name='abc') 
data.node.exists(name='abc')
```

*schema with a multiple keys*

**Note: specify them in the correct order**

```python
data.node.get('abc', 1) 
data.node.create('abc', 1) 
data.node.exists('abc', 1)
```

or

```python
data.node.get(name='abc', id=1) 
data.node.create(name='abc', id=1) 
data.node.exists(name='abc', id=1)
```

The DataOfChildrenType Attributes are defined in the following table

| Attribute     | Description    |
| :------------ | :------------- |
| get(*args, **kwargs) | Returns an existing child with the specified keys. Generates an AttributeError if a wrong number of keys is given, and KeyError if there is the child does not exist. |
| exists(*args, **kwargs) | Returns True if a child with the specified keys exists. Generates an AttributeError if a wrong number of keys is given. |
| create(*args, **kwargs) | Creates and returns a child with the specified keys. If this child already exists, the existing child is returned (and no changes are made). Generates an AttributeError if a wrong number of keys is given. |
| count() | Counts the number of children. |
| is_empty | Returns True if there are no children of this type. |
| items() | Iterates over all children of this type and are sorted based on their keys. |
| clear() | Removes all children of this type. |
| formatter | Returns the Formatter that can be used to generate the show report for the Data object. |
| iter_format(max_width) | Invokes the Formatter.iter_format_type() of these Data objects. Returns an iterator over the formatted output lines. |