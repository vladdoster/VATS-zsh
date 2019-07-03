# Valgrind Automatic Tests Suite (VATS)

Runs sequence of tests, preceded by `valgrind` call, with a command and arguments taken from file `vtest.conf`.
The arguments are dynamic because the `vtest.conf` entry can look like this:

```SystemVerilog
# Arguments passed to $test_bin, evaluated at use (i.e. for each test separately)
test_bin_args='+Z -f $ZTST_srcdir/ztst.zsh $file'   # runs ztst.zsh on given $file
test_bin="local-zsh"                                # expands to ../Src/zsh
```

The variable `$file` is set to current test-file for each test-run. Example test file:

**Zshell**: For Zsh-integrated VATS, `$file` is just passed as argument to `ztst.zsh`. So it is
`A01grammar.ztst`, for example.

## Error Definitions

You can define errors so that they are skipped from test result (i.e. from Valgrind output). This is
the main feature of VATS because it allows to quickly check if changes broke anything. Usage is very
much like of unit tests – run `make TESTNUM=A01`, look for any red color, done. Zero parsing with
eyes. A typical definition for Zshell can look like this:

```zsh
errors1+=( "* / zsh_main / setupvals / gettimeofday / *" )
```

and is placed in `__error1.def` or other such file with index.

## Integrating With Project

Following patch applied to the Test/Makefile solves the integration of VATS to Zsh:
```diff
──────────────────────────────────────────────────────────────────────────────────────────
modified: Test/Makefile.in
──────────────────────────────────────────────────────────────────────────────────────────
@@ -35,6 +35,7 @@ VPATH           = @srcdir@
 sdir            = @srcdir@
 sdir_top        = @top_srcdir@
 INSTALL         = @INSTALL@
+VLGRND                 = $(VALGRIND:1=v)

 @DEFS_MK@

@@ -49,7 +50,7 @@ check test:
            do echo $$f; done`" \
         ZTST_srcdir="$(sdir)" \
         ZTST_exe=$(dir_top)/Src/zsh@EXEEXT@ \
-        $(dir_top)/Src/zsh@EXEEXT@ +Z -f $(sdir)/runtests.zsh; then \
+        $(dir_top)/Src/zsh@EXEEXT@ +Z -f $(sdir)/$(VLGRND)runtests.zsh; then \
         stat=0; \
        else \
         stat=1; \
```

## Fundamental Test-Configuration

The configuration-file of tests is `vtest.conf`. It defines two settings:

```zsh
test_bin="../Src/cgiturl"   # Binary that runs any test, or is the tested program itself
zsh_control_bin="zsh"       # Binary used when scheduling tests & interpreting Valgrind output
```

Variable `zsh_control_bin` is used to implement special `#!` behavior: `runtests.zsh`
starts with `#!/bin/sh`, reads `vtest.conf`, and restarts with `zsh_control_bin`. This way
user can define shebang interpreter via separate configuration file (`vtest.conf`).

The second script that uses `zsh_control_bin` is `zsh-valgrind-parse.cmd`. It also restarts
via `#!/bin/sh` and `exec /usr/bin/env "$zsh_control_bin"`.

The setting `test_bin` specifies the test-program used to run tests. **This binary is examined by Valgrind**.

## Remaining Test-Configuration

The setting `tkind` is used to set a **test-kind**. These are modes of Valgrind operation.
Allowed values are: `error` (only detect read/write errors), `leak` (also detect memory leaks),
`nopossiblylost` (detect memory leaks, but not _possibly_ lost blocks).

```zsh
tkind="leak"                # Test kind: error, leak, nopossiblylost
```

`Valgrind` messages of type `summary` and `info` are muted via lines:

```zsh
summaries="no"               # show valgrind summaries?
info="no"                    # show info messages?
```
