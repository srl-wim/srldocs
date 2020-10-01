# SR-Linux Syntax plugin

## Introduction

The SR-Linux syntax plugin defines the syntax of how the user can execute the command. The syntax is based on a name-tag with arguments with attributes.

## Argument

A command can have arguments, SR-Linux defines the following argument types

- named-arguments: an argument with a name-tag and a value
- unnamed-arguments: an argument with only a value
- boolean-arguments: an argument with a name-tag, but without a value

## Attributes

An argument in SR-Linux can be defined with the following attributes:

- names: The name used as tag for the named-arguments.
- default [optional]: The default-value used if the argument is not specified in the CLI.
- min_count [default 1]: The minimum number of values that can be entered. Must be > 1
- max_count [default 1]: The maximum number of values that can be entered. Must be > 1 or * for a list
- count [default 1]: The exact number of values that must be entered. Is equivalent to setting both 'min_count' and 'max_count' to this same value.
- array_type [default ArgumentArrayType.none]:
	- If ArgumentArrayType.none, then the value is not an array
	- If ArgumentArrayType.leaflist, then '[' and ']' are required before/after the values on the command line.
	- If ArgumentArrayType.leaflist_optional_value_or_brackets, then '[' and ']' are optional before/after the values on the command line, single value is also optional
	- If ArgumentArrayType.leaflist_mandatory_value_or_brackets, then '[' and ']' are optional before/after the values on the command line, but at least single value is is_mandatory
- choices [optional]: A list of accepted values. An error will be raised if the value is not one of the given choices.
- suggestions [optional]: Either a list of suggested values, or a callable that returns such a list. No error will be raised if the value is not one of the suggested values. 
- callback: will be invoked with the following arguments:
	- The Argument describing the argument of which we're retrieving the values
	- state: The CliState
	- The CommandNodeWithArguments describing the input node and the current context.
- value_checker [optional]: A callable that can be used to select which values are accepted
- help: defines a help description for the argument

## Examples

Example named + boolean arguments:

```python
def _get_syntax(self):
    syntax = (Syntax(
            'action',
            help='test action')
        .add_boolean_argument(
            'trigger',
            help='trigger action')
        .add_named_argument(
            'start',
            value_checker=IntegerValueInRangeChecker(min_value=1),
            default='1',
            help='The starting sequence-id that will be assigned to the first entry')
        .add_named_argument(
            'increment',
            value_checker=IntegerValueInRangeChecker(min_value=1),
            default='10',
            help='The constant sequence-id increment between adjacent entries'))
    return syntax
```

Example unamed argument:

```python
syntax_pull_request = (Syntax(
    'pull-request', 
    help='git pull-request creates a pull request based on the commits in github')
    .add_unnamed_argument('prSubject',
        help='pull request subject fir the pull-request')
    .add_unnamed_argument('prDescription',
        help='pull request Description for the pull-request'))
pull_request = git.add_command(syntax_pull_request, 
    update_location=False,
    callback=git_pullrequest_process)
```