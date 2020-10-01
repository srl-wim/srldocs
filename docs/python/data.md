# SR-Linux Data class

## Introduction

The data class represent all data; both the data retrieved from the server and the data displayed in the show report. The data class allows easy access to a configuration/state instance. When creating a top-level Data object, you must specify an instance of a SchemaNode. The code analyzes the schema, and makes all fields, keys, leaf-lists, and children accessible as attributes.

## Usage

```python
Class srlinux.data.data.Data(schema, parent=None, **keys)
```

For example, assume that this is the data model:

```python
list interface { 
    key 'name';
    field 'admin-state'; 
    leaflist 'values'; 
    list subinterface {
        key 'id'; }
    }
```

*Get the keys*

```python
value = data.name
```

*Get and set the fields*

```python
value = data.admin_state # returns the value or None if unset 
data.admin_state = 'enabled'
```

*Get and set leaflists*

```python
value = data.values # returns a list 
data.values = ['a', 'b']
```
*Access children*

```python
child = data.subinterface.create(42) # Creates the subinterface with id '42' 
child = data.subinterface.get(42)
if data.subinterface.exists(42):
    # Returns True/False
for si in data.subinterface.items():
# Walks all subinterfaces, ordered by their key
```

**Note**

Names are changed so that any character that is not a to z, A to Z, or 0 to 9 is replaced by an “_”.

## Children

When accessing a child (data.subinterface in the preceding example), an instance of *DataChildrenOfType* is returned. Follow the link to see all the accessors it provides.

## Values

All key/field/leaf-list values are of the following types:

- bool
- integer 
- string

## Formatters

To generate the show report, Formatter objects are used. These can be tied to a
specific Data object using two methods:

- By assigning a formatter to the Data.formatter property.
- By using Data.set_formatter().

Both methods assign the formatter to all sibling Data objects. For example, calling the following sets the formatter for all interfaces and not just for subinterface 42.

```python
data.subinterface.get(42).formatter = ColumnFormatter()
```

The table below lists the different types of formatters.

| Formatter     | Description    |
| :------------ | :------------- |
| parent | Returns the parent Data object, or None if this is the root. |
| schema | Returns the SchemaNode. |
| key_names | Returns an iterator over the names of the keys. |
| key_values | Returns an iterator over the key values. Iterators are returned in the order specified by self.key_names, which is the same as the order that the keys were added to the SchemaNode. |
| get_key(name) | Returns the value of the key with the given name. |
| keys_dict | Returns a “name: value” dictionary of the keys. |
| type_name | Returns the type-name, which is self.schema.name. |
| field_names | Returns an iterator over the names of the fields. |
| field_values | Returns an iterator over the field values. Iterators are returned in the order specified by self.field_names, which is the same as the order that the fields were added to the SchemaNode. |
| set_field(name, value) | Assigns the value to the field with the given name.|
| get_field(name, default=None) | Returns the value of the field with the specified name, or the default if the field is unset. |
| is_field_set(name) | Returns True if the field is set. |
| leaflist_names | Returns an iterator over the names of the leaf-lists. |
| leaflist_values | Returns an iterator over the leaf-list values. Iterators are returned in the order specified by self.leaflist_names, which is the same as the order that the leaf-lists were added to the SchemaNode. |
| get_leaflist(name) | Returns a list containing the values of the leaf-list with the specified name. This list is empty if the leaf- list is unset. |
| set_leaflist(name, value) | Assigns the value to the leaf-list with the specified name. The value must be a list. |
| is_leaflist_set(name) | Returns True if the leaf-list is set. |
| child_names | Returns an iterator over the names of the children. |
| get_children(name) | Returns the DataChildrenOfType instance that contains all the children of the specified type. This returned object is mutable and can be used to walk/ retrieve/add children. |
| iter_children_by_type(predicate=<f unction Data.<lambda>>) | Iterates over all DataChildrenOfType instances for which the predicate is True. By default, this returns all children. |
| iter_children() | Iterates over all child instances. |
| get(name) | Returns the value of the specified key, field, leaf-list, or child. |
| get_annotations(name=None) | Returns a list containing the annotations of this node (if called with no arguments) or the field with the specified name. |
| add_annotation(annotation, name=None) | Adds the specified annotation to this node (when called with a single argument) or to the field with the specified name (when called with two arguments). The annotation must be an instance of Annotation. |
| formatter | Returns the Formatter that can be used to generate the show report for this Data object. |
| iter_format(max_width) | Invokes the Formatter of this Data object. Returns an iterator over the formatted output lines. |
| iter_format_children(max_width) | Invokes the Formatter of all children of this Data object. Returns an iterator over the formatted output lines. |
| set_formatter(schema, formatter) | Adds a Formatter to the Data object with the specified schema. The schema can be specified as an XPath string (without keys). For example, “/ interface/subinterface”. |
| get_schema(path) | Get the SchemaNode of the Data object with the specified path. The path must be an XPath string, for example, “/interface/subinterface”. |

for more detail on DataChildrenOfType click [here](datachildrenoftype.md)

