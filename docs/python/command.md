# SR-Linux Command plugin

## Introduction

The CLI Plugin is built based on command nodes which are modeled as a nested hierarchy. After entering a given command, you can then enter any of its child commands by calling 'add_command' or 'add_custom_commands_hook' on the command

## add command

The add_command allows to extend the command tree. The arguments are:

* syntax: Instance of srlinux.syntax.Syntax. Represents the syntax of how the user can execute the command (i.e. the name plus all the arguments)
* callback: The hook that will be called when the command is executed, with the following arguments:
	* state (type srlinux.mgmt.cli.CliState) which gives you access to the current state of the CLI engine, as well as access to the server
	* input (type Input) which allows you to request user input
	* output (type CliOutput) which allows you to print output/errors
	* arguments (type CommandNodeWithArguments) All the arguments that the user entered when invoking this command.
* update_location: If true, this command is added to the current location when executed.

```python
git = state.command_tree.tools_mode.root.add_command(
	syntax_git,
	update_location=False)
	
branch = git.add_command(
	syntax_branch, 
	update_location=False, 
	callback=git_process)
```

## get command

The get_command provide a command node based on a name. E.g. this is used to get the context of an existing command

```python
cpmfilter_ipv4 = acl.get_command('cpm-filter').get_command('ipv4-filter')
```
                    
## add custom commands hook

When extending the CLI hierarchy we have to identify the root of the tree where the command node is to be added.

E.g. when you want to extend the tools command at the root

```python
syntax_git = Syntax('git', 
    help='`Git interaction as a client`')
git = state.command_tree.tools_mode.root.add_command(syntax_git,
    update_location=False)
```

E.g. When you want to extend a given cli hierarchy you can lookup the node using the get_command as per example below:

```python
acl = state.command_tree.tools_mode.root.get_command('acl')
cpmfilter_ipv4 = acl.get_command('cpm-filter').get_command('ipv4-filter')
cpmfilter_ipv6 = acl.get_command('cpm-filter').get_command('ipv6-filter')
ipv4filter = acl.get_command('ipv4-filter')
ipv6filter = acl.get_command('ipv6-filter')
cpmfilter_ipv4.add_command(self._get_syntax(),
                           update_location=False,
                           callback=acl_resequence_process)
```

