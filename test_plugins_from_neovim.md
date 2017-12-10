# Overview

[Neovim](https://github.com/neovim/neovim) contains a testing infrastructure based on 
[busted](http://olivinelabs.com/busted/). It's not only used to do unit test, but also integration
tests (called "functional tests" from now on). It's so nice to work with that I'm using it to write
tests for my own [plugin](https://github.com/KillTheMule/nvimpam/), and I want to point out how to
do this. Here's a quick overview over the pros and cons of this approach, after which we'll dive
right in:

### Pros

* Write tests in lua, a very pleasant language
* Directly assert screen state, i.e. what your users will see
  * This includes a lot of parameters you could still check independently, but probably won't:
    Windows arrangement (including exact sizes), status line, cursor position, highlighting
    information
### Cons

 * It's a quite "heavy handed" approach, both by the additional code used (the neovim source), and
   the work needed to be done by the system running the tests (need to compile neovim)
   * Caching and git submodules help. It does not mean you have to do work yourself, it's the 
     computer sweating, and I personally don't mind that, considering the advantages.
     
# How to run a test

Once you've created a test file, say `myplug_spec.lua`, you can simply run
`TEST_FILE=/path/to/myplug_spec.lua make functionaltest` in the `neovim` directory. It will compile
neovim and all dependencies if neccessary, and then run the test. If successfull, the output will
look like

```
[----------] Global test environment setup.
[----------] Running tests from tmp_spec.lua
[ RUN      ] myplug basically works: 379.14 ms OK
[----------] 1 test from tmp_spec.lua (387.03 ms total)
[----------] Global test environment teardown.
[==========] 1 test from 1 test file ran. (387.13 ms total)
[  PASSED  ] 1 test.
```

If a test fails, the output might look like this

```
[----------] Global test environment setup.
[----------] Running tests from tmp_spec.lua
[ RUN      ] myplug basically works: ERR
./test/functional/ui/screen.lua:306: Row 1 did not match.
Expected:
  |*Ths is a {1:lin^e}                          |
  |{2:~                                       }|
  |{2:~                                       }|
  |{2:~                                       }|
  |                                        |
Actual:
  |*This is a {1:lin^e}                          |
  |{2:~                                       }|
  |{2:~                                       }|
  |{2:~                                       }|
  |                                        |

To print the expect() call that would assert the current screen state, use
screen:snapshot_util(). In case of non-deterministic failures, use
screen:redraw_debug() to show all intermediate screen states.  

stack traceback:
        ./test/functional/ui/screen.lua:306: in function 'wait'
        ./test/functional/ui/screen.lua:220: in function 'expect'
        tmp_spec.lua:27: in function <tmp_spec.lua:23>
```

I've snipped a bit of the test failure output, and there are colors on the command line, but you
can see the most important things:

 * The traceback tells us where exactly an assertion (in this case called `expect`) fails:
   `tmp_spec.lua:27`
 * We're told which line was the first not to match: `Row 1 did not match`
 * We get a full printout of what we expected the screen to look like, and how it actually DID look
 * We get a hint to use `snapshot_util`, which I will tell you about later. It's very usefull!
 
 
 # How to write a test file
 
 First of all, Lua is a very easy and nice language, if you're not familiar with it, you might
 want to check out http://tylerneylon.com/a/learn-lua/. It's very easy to read, though, so
 you can probably follow along without it.
 
 Here's a template to start with:
 
 ```lua
local helpers = require('test.functional.helpers')(after_each)
local Screen = require('test.functional.ui.screen')
local clear, command = helpers.clear, helpers.command
local feed, alter_slashes = helpers.feed, helpers.alter_slashes
local insert = helpers.insert

describe('myplug', function()
  local screen

  before_each(function()
    clear()
    screen = Screen.new(81, 15)
    screen:attach()

    command('set rtp+=' .. alter_slashes('../'))
    command('source ' .. alter_slashes('../plugin/nvimpam.vim'))
  end)

  after_each(function()
    screen:detach()
  end)

  it('basically works', function()
    insert("This is a line")
    command("MyPluginFunction")
    
    screen:expect([[
     This is a {1:lin^e}                          |
     {2:~                                       }|
     {2:~                                       }|
     {2:~                                       }|
                                          |
]], {[1] = {foreground = Screen.colors.Grey100, background = Screen.colors.Red}, [2] = {bold = true, foreground = Screen.colors.Blue1}})

  end)

end)
```
 
 The first 5 lines contain some general setup and some shorthands. The `describe` block contains
 the first test group. You can have several of those if you want, and easy one contains several
 `it` blocks, which are the individual tests. In the `describe` block, the `before_each` and
 `after_each` blocks do some setup/teardown common to all tests in this block. Since we want
 to do screen tests, we make a new screen for all the tests and attach it to our neovim instance
 before each test, and detach it afterwards.
 
 Since we're testing a plugin, we need some setup. `command` calls a neovim command, just the
 same if you typed `:` in normal mode. In this case, we set up the runtime path (might or
 might not be needed), and we source the plugin file. This saves us from doing the installation
 step of our plugin. Note the use of the `alter_slashes` function and the relative paths, which
 will make the whole thing work on windows as well as unixies.
 
 Let's assume our plugin provides "MyPluginFunction" which will highlight the word "line" as an
 error (that is, it's just `syn keyword Error line`), which we want to test. We insert some text
 containing the word "line", and call our function. After that, we assert our screen state
 using `screen:expect`. The part `{1:lin^e}` tells you this word was colored from what is
 given in color `[1] = {foreground = Screen.colors.Grey100, background = Screen.colors.Red}`, and
 the cursor is on the letter "e", which is what we were after. There's still more to see - the
 width and height of the window is asserted, the color of the tildes, and the rest of the text.
 If anything doesn't fully match, we will get a test failure.
 
 *Note*: See [my test file](https://github.com/KillTheMule/nvimpam/blob/master/test/nvimpam_spec.lua#L14)
 for how to shorten the output by predefining a color <-> number association beforehand.
 
 ### How to write an `expect` call
 
 Your first reaction was probably the right one: No one want to write those `expect` calls by
 hand. So here's how to do it: Write the test up to this call, but then put
 `screen:snapshot_util()` in its place. Run the test, and the exact `expect` call will be printed
 to the terminal, ready for you to copy & paste. *Of course* you will need to verify that it
 looks exactly the way you want it to, otherwise your test will be quite useless.
 
 ### Advanced usage
 
 By now you're hopefully convinced that you can quite easily test complex circumstances. If you
 need more inspiration, have a look at the
 [ui tests](https://github.com/neovim/neovim/tree/master/test/functional/ui) from
 neovim. If you get stuck, the [neovim community](https://neovim.io/community/) can surely
 help you out.
 
 # How to do it on CI
 
 CI can be quite daunting, so I just want to leave some hints how to do it on the most popular
 CI services. Note that I'm carrying around the neovim sources a part of my tree, since I need a
 patched version of neovim. You'll probably want to make use of a git submodule here.
 
 * Appveyor:
     * See my [appveyor.yaml](https://github.com/KillTheMule/nvimpam/blob/master/appveyor.yml#L56)
       I'm using an environment variable to only run those test for a subset of the testing
       matrix, since it's quite expensive. On windows, you will need to run `build.bat` as shown.
       Note the use of [caching](https://github.com/KillTheMule/nvimpam/blob/master/appveyor.yml#L67),
       so the deps won't need to be build every time.
 * Travis
     * Again, see my [.travis.yml](https://github.com/KillTheMule/nvimpam/blob/master/.travis.yml#L31),
       this is somewhat easier than on appveyor still, just run the commands in the right directory.
       Never mind the call to `cargo`, this is just building my plugin because I'm writing it in Rust.
 
 
