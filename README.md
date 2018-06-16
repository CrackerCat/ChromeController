### Chrome Remote Debug Protocol interface layer and toolkit.

Interface for communicating/controlling a remote chrome instance via the Chrome 
Remote Debugger protocol.

#### Quickstart:

```python
import ChromeController

with ChromeController.ChromeContext(binary="google-chrome") as cr:
    
    # Do a blocking navigate to a URL, and get the page content as served by the remote
    # server, with no modification by local javascript (if applicable)
    raw_source = cr.blocking_navigate_and_get_source("http://www.google.com")
    
    # Since the page is now rendered by the blocking navigate, we can
    # get the page source after any javascript has modified it.
    rendered_source = cr.get_rendered_page_source()
    
    # We can get the current browser URL, after any redirects.
    current_url = cr.get_current_url()
    
    # We can get the page title as the browser sees it.
    page_title, page_url = cr.get_page_url_title()
    
    # Or take a screenshot
    # The screenshot is the size of the remote browser's configured viewport,
    # which by default is set to 1024 * 1366. This size can be changed via the
    # Emulation_setVisibleSize(width, height) function if needed.
    png_bytestring = cr.take_screeshot()
    
    
    # We can spoof user-agent headers:
    new_headers = {
                'User-Agent'      : 'Mozilla/5.0 (Windows NT 6.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/61.0.3163.79 Safari/537.36,gzip(gfe)', 
                'Accept-Language' : 'en-us, en;q=1.0,fr-ca, fr;q=0.5,pt-br, pt;q=0.5,es;q=0.5', 
                'Accept'          : 'image/png,  text/plain;q=0.8, text/html;q=0.9, application/xhtml+xml, application/xml, */*;q=0.1', 
                'Accept-Encoding' : 'gzip,deflate',
            }
    cr.update_headers(new_headers)
    
    
    # We can extract the cookies from the remote browser.
    # This call returns a list of python http.cookiejar.Cookie cookie
    # objects (the Chrome cookies are converted to python cookies).
    cookie_list = cr.get_cookies()
    
    # We can also set cookies in the remote browser.
    # Again, this interacts with http.cookiejar.Cookie() objects
    # directly.
    cook = http.cookiejar.Cookie(<params>)
    cr.set_cookie(cook)

    # We can create more tabs in the current browser context.
    # Note that these additional tabs are scoped to the same lifetime as the original 
    # chromium object (`cr`), so they will become invalid after leaving the 
    # ChromeContext() context manager.
    tab_2 = cr.new_tab()
    tab_3 = cr.new_tab()

    # At this time, multiple tabs are not thread safe, so they *probably* shouldn't 
    # be accessed concurrently. This *is* something that I'd like to change.

```

This library makes extensive use of the python `logging` framework, and logs to 
the `Main.ChromeController.*` log path.

Automatic wrapper class creation for the remote interface by parsing
the chrome `protocol.json` file, and dynamic code generation through dynamic 
AST building. While this is not the most maintainable design, I chose it mostly
because I wanted an excuse to learn/experiment with python AST manipulation.

A higher level automation layer is implemented on top of the autogenerated 
wrapper. Both the higher-level interface, and it's associated documentation are 
very much a work in process at the moment.

Interface documentation is here:  
https://fake-name.github.io/ChromeController/ChromeController.ChromeRemoteDebugInterface.html

All remote methods are wrapped in named functions, with (partial) validation 
of passed parameter types and return types.
Right now, simple parameter type validation is done (e.g. text arguments must be
of type string, numeric arguments must be either an int or a float, etc..). 
However, the compound type arguments (bascally, anything that takes an array 
or object) are not validated, due to the complexity of properly constructing 
type validators for their semantics given the architecture (read: writing the
validator in raw AST broke my brain).

Tested mostly on python 3.5, lightly on 3.4 and 3.6, all on linux. If you are 
using python 2, please stahp. It works with normal chromium and on windows, 
but that has only been very lightly used. My test-target is the 
google-provided `chrome` binary.

Note that this tool generates and manipulates the AST directly, so it is 
EXTREMELY sensitive to implementation details. It is *probably* broken on 
python > 3.6 or < 3.4.

Transport layer (originally) from https://github.com/minektur/chrome_remote_shell

The library also has a relatively useful CLI interface, principally useful for 
doing things like fetching pages which have jerberscript-rendered content:

```
 python3 -m ChromeController --help
Usage: __main__.py [OPTIONS] COMMAND [ARGS]...

  ChromeController

  Usage: python3 -m ChromeController [-s | --silent] [-v | --verbose]
    python3 -m ChromeController fetch <url> [--binary <bin_name>] [--outfile <out_file_name>]
    python3 -m ChromeController update
    python3 -m ChromeController (-h | --help)
    python3 -m ChromeController --version

  Options:
    -s --silent   Suppress all output aside from the fetched content
                  This basically makes ChromeController act like a alternative to curl
    -v --verbose  The opposite of silent. Causes the internal logging to output
                  all traffic over the chromium control interface. VERY noisy.
    --version     Show version.
    fetch         Fetch a specified URL's content, and output it to the console.


Options:
  -v, --verbose  The opposite of silent. Causes the internal logging to output
                 all traffic over the chromium control interface. VERY noisy.
  -s, --silent   Suppress all output aside from the fetched content.
                 This can be used to make ChromeController act like
                 an alternative to curl with proper JS rendering.
  --help         Show this message and exit.

Commands:
  fetch    Fetch a specified URL's content, and output...
  update   Update the generated class
  version  Print the ChromeController Version

```



License:
BSD




------

Current Usage so far has been basically to find bugs or strangeness in the 
chromium remote debug interface:

 - Strange Behaviour is `network.getCookies` (fixed)  
     https://bugs.chromium.org/p/chromium/issues/detail?id=668932
 - `network.clearBrowserCookies` appears to have no effect  
     https://bugs.chromium.org/p/chromium/issues/detail?id=672744
 - Overriding accept header fails  
     https://bugs.chromium.org/p/chromium/issues/detail?id=849972

