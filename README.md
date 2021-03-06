# IPython Kernel for ELM


This is a Python kernel that has hooks to inspect and modify user input before it's processed (for example, to replace
certain terms, exclude from the command history or make more complex transformations using the AST module)


## Installation


Run

```bash
pip install .
```

to fetch the needed dependencies and install the kernel.


## Running

After launching Jupyer (for example with jupyer notebook) under the New... dropdown you should see an entry "ELM (Python
3)"

Without configuring this works exactly the same as the original kernel.


## Configuring

The file ipython_kernel_elm_config.py defines a simple set of filters as example.
In order for it to be loaded you need to do one of this:

  - Copy it to ~/.ipython/profile_default
  - Copy it to /etc/ipython (if running from a virtual env it's under VIRTUALENV_DIR/etc/ipython)
  - Set the environment variable IPYTHONDIR (see IPython docs)
  - Run jupyter from the directory where that file is

The filter classes do not need to be all in this file, you can make a custom package and import it from there.



### Filter definition

Filters are classes that extend from `elm_kernel.filters.BaseFilter`


```python
class BaseFilter:
    """
    This is the reference implementation for all filters/hooks.
    Just passes the data as-is without changing it.
    """
    def register(self, kernel, shell):
        self.kernel = kernel
        self.shell = shell

        shell.events.register('post_run_cell', self.post_run_cell)
        shell.input_transformers_cleanup.append(self.process_text_input)

        # You can also perform more advanced modifications, see:
        # https://ipython.readthedocs.io/en/stable/config/inputtransforms.html#ast-transformations

    def process_text_input(self, lines):
        return lines

    def process_text_output(self, text):
        """
        This is called from the kernel when displaying the results of a command back to the User
        """
        pass

    # This is called from the kernel before feeding input into the IPython Shell
    def process_run_cell(self, code, options):
        """
        Modifies the arguments and code passed to shell.run_cell()
        options is a dict like
        {
            'silent': False,
            'store_history': True,
            'user_expressions': None

        }
        that can be modified in place to change behaviour.
        Returns: the new code to run
        """
        return code

    # This is called from the kernel before returning completion data
    def process_completion(self, code, cursor_pos, completion_data):
        """
        This is called from the kernel before returning completion data
        completion_data is a dict like
        {
            'matches' : matches,
            'cursor_end' : cursor_pos,
            'cursor_start' : cursor_pos - len(txt),
            'metadata' : {},
            'status' : 'ok'
        }
        """
        return completion_data

    def post_run_cell(self, result):
        """
        This is called after executing a cell with the result of that
        """
        pass
```

We added a configurable trait to ElmPythonKernel, code_filters. It is a list of filter instances.
When a new kernel is launched first the register() method is called with the kernel and IPython shell as arguments.

After that the corresponding methods are called when the User enters commands and Python gives a result back.

The config file included under `examples/` has a sample filter that logs user input to a file on /tmp and makes the following
changes:

  - Replaces all occurrences of 'FORBIDDEN_WORD' with 'SAFE_WORD'
  - If anywhere in the code there's the text 'no-history', that block will not be stored on the internal command history
    (%hist)
