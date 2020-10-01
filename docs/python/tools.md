# Tools Python Plugin framework

## Introduction

SR Linux is an open networking NOS that can be extended in various ways. In this section we focus on the python plugin framework for extending the tools commands.

## Tools command Plugin

A tools plugin is defined by redefining the ToolsPlugin/CliPlugin Class with its own methods, that define the dependencies, the CLI syntax and its behavior that the plugins uses to extend the system.

### class inheritance

The plugin inherits the ToolsPlugin class and extends it.

```python
class Plugin(ToolsPlugin):
```

### Dependencies

The following statement defines the dependency on the tools_mode. The **get required plugins** method returns a list with the dependencies this plugin needs.

```python
class Plugin(ToolsPlugin):
    def get_required_plugins(self):
        return [
            RequiredPlugin('tools_mode')
        ]
```

### Extending CLI hierarchy

The **on tools load** method provides the ability to extend the cli hierarchy.

```python
class Plugin(ToolsPlugin):
	def on_tools_load(self, state):
```

In the **on tools load** you can define the commands with syntax and callback functions that provide logic when executing the command.
The command and syntax extensions are explained in more detail here.

First, through the **state** variable, you get access to the current state of the CLI engine. This allows to 

```python
git = state.command_tree.tools_mode.root.add_command(syntax_git,
            update_location=False)

```
After, through the **add command**, you can extend the CLI hierarchy. The add_command also allow to extend the command **syntax** and optionaly allows to execute a **callback** function which can provide some logic that the command executes.

```python
class Plugin(ToolsPlugin):
    '''
        git
    '''

    def get_required_plugins(self):
        return [
            RequiredPlugin('tools_mode')
        ]
    def on_tools_load(self, state):
        syntax_git = Syntax('git', 
            help='`Git interaction as a client`')
        git = state.command_tree.tools_mode.root.add_command(syntax_git,
            update_location=False)

        syntax_branch = Syntax('branch', 
            help='git branch creates a branch in github')
        branch = git.add_command(syntax_branch, 
            update_location=False, 
            callback=git_branch_process)
            
def git_branch_process(state, output, arguments, **_kwargs):
    action_arg = {}

    yang_val, result_string = yang_validation(state)
    if yang_val:
        git_rpc_cal('Server.Branch', action_arg, output)
    else:
        output.print_error_line(result_string)
```

## Callback function

In the callback function we can define logic that is used by the plugin to execute various tasks. E.g. in this example we use a yang validation and call a json rpc call to execute a task.

```python
def yang_validation(state):
    path = build_path('/git-client')
    result = state.server_data_store.get_json(
        path, 
        recursive=True,
        include_field_defaults=True)

    result_json_obj = json.loads(result)
    #print(result_json_obj)

    org_exists = 'organization' in result_json_obj
```

In the yang validation we call the state that is defined in the **git-client** tree and validate the data.

In the git-rpc-call we call a json rpc server to execute a task, and we update the **output** based on the returned result.

```python
def git_rpc_call(method, action_args, output):
    url = "http://localhost:7777"

    s_exists = 'subject' in action_args
    if s_exists:
        subject = action_args['subject']
    else:
        subject = 'dummy'

    c_exists = 'comment' in action_args
    if s_exists:
        comment = action_args['comment']
    else:
        comment = 'dummy'

    payload = {
        "method": method,
        "params": [{"Subject": subject, "Comment": comment}],
        "jsonrpc": "2.0",
        "id": 0,
    }
    response = requests.post(url, json=payload).json()

    if response["result"] != 'success':
        output.print_error_line(response["result"])
    assert response["id"] == 0
```