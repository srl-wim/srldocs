# SR-Linux Output plugin

## Introduction

The Output Plugin allows you to print output and errors

## Output Methods

The folowing output methods are defined

```
* output_format(self):
* set_output_format(self, output_format):
* screen_dimensions(self):
* print_data(self, data):
* print(self, text):
* print_line(self, text=''):
* print_error(self, text):
* print_error_line(self, text=''):
* print_warning(self, text):
* print_warning_line(self, text=''):
* print_info(self, text):
* print_info_line(self, text=''):
```

## Example

```
output.print_error_line(result_string)
```