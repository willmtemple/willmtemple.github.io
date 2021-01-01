---
title: Lifecycle of a BlockyTalky 3 Program
subtitle: Getting to know the experimental BlockyTalky 3 runtime, the lifecycle of a user's program, and how the source code interacts with the Raspberry Pi hardware.

# Summary for listings and search engines
summary: Getting to know the experimental BlockyTalky 3 runtime, the lifecycle of a user's program, and how the source code interacts with the Raspberry Pi hardware.

# Link this post with a project

projects: ["blockytalky3"]

# Date published
date: "2019-06-13"

# Date updated
lastmod: "2020-12-30"

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

# Featured image
# Place an image named `featured.jpg/png` in this page's folder and customize its options here.
image:
  caption: 'Lifecycle Diagram of a BlockyTalky 3 Program'
  focal_point: ""
  placement: 2
  preview_only: false

authors:
- wtemple

tags:
- Education
- Software
---

In February 2018, I started working seriously on the [BlockyTalky 3][bt3]
project, a web-based physical computing editor for linux systems. We initially
chose to target the Raspberry Pi, and we chose to use [Microsoft
MakeCode][makecode] as the base of the web IDE. MakeCode is a relatively mature
blocks-based code editor (one of the official platforms of the BBC micro:bit),
and, for a while, it made a lot of sense to build off of it instead of creating
a new editor from scratch.

However, MakeCode was designed to compile users’ Static TypeScript code to
native ARM machine code assemblies to be flashed to the device over USB or a
programming interface. In our case, the Raspberry Pi is a fully-functional
Linux device, so we are able to run whatever Linux software we can manage to
compile for it as long as it is within the Pi’s ability to run it. So, we
decided to simply transmit the user’s code from the web editor to the Raspberry
Pi for execution. This approach had a few major benefits:

1. Running the code in a full Linux environment would allow us to rapidly
   develop integrations with the rich ecosystem of Linux and Node software.
2. We wouldn’t have to build complex C/C++ libraries to leverage the hardware
   of the Raspberry Pi.
3. We would be able to use the Pi itself to apply transformations to user code
   and archive it for research purposes.

It took a few months of planning, trial, and error to nail down exactly how
this program lifecycle would work. We finally settled on a procedure with three
major steps. This is how a user’s program in the browser eventually executes on
the BlockyTalky Unit (Raspberry Pi).

{{< figure numbered="true" title="BlockyTalky Document Flow" src="./featured.png" >}}

## Editor

When the user is writing the program in the editor within their browser, it
continuously runs automated analysis (such as typechecking) against the user’s
code. In our derivative of MakeCode, the in-browser virtual machine is
disabled, but the compiler still runs and generates diagnostics. When the user
clicks the button labeled “Compile and Run” in the bottom left (pictured
below), the editor gathers the entire user code bundle in source format (see
the next section), serializes it to JSON, and transmits it to the BTU (the Pi).
It uses an HTTP POST message to the endpoint at /api/save using the same server
and port that originally served the editor itself. Conceptually, this is the
simplest component of BlockyTalky 3. In actual practice, it is the hardest
component to modify and extend.

{{< figure numbered="true" title="BlockyTalky 3 Editor Interface" src="./editor.png" >}}

### Anatomy of a User Bundle

User code files are organized into a virtual file-system within the web editor.
In each bundle, there is:

- One pxt.json file that encodes information about the files in the bundle and
  their dependencies
- One main.ts file that contains the main TypeScript code, generated from the
  blocks workspace. This is the entry-point of the program.
- Optionally, one or more extension TypeScript (.ts) files. Exported functions
  and classes in this file are available in main.ts without needing to be
  imported.

On the wire between the editor and the orchestrator, these files are serialized
together as a single JSON-encoded object. That object has filenames as keys and
file contents as values.

The orchestrator manages executing child processes on the BTU. Conceptually,
this is probably the most complicated part of the process, though it involves
the least actual code. When a new user code bundle is POST’d to /api/save, the
orchestrator attempts to decode it, and it then transforms each TypeScript file
in the user’s code (non-TypeScript resources in the user code are not currently
supported) according to certain rules that ensure a standard TypeScript
compiler will be able to resolve dependecies which are abstracted away by the
novice-friendly PXT editor. Those rules are (see this file on github):

- For BOTH main.ts and all extension files: the core library is always inserted
  first, and the loops and console libraries are always inserted next, as all
  packages implicitly import them, so each file begins with:

```typescript
import * as _core from '/path/to/pxexec/lib/core-exec';
import * as loops from '/path/to/pxexec/lib/loops';
import * as console from '/path/to/pxexec/lib/console';
```

- For BOTH main.ts and all extension files: Any packages marked as dependencies
  in pxt.json (from the user bundle) are imported in order in the same manner
  as the loops library. For example, projects import gpio by default (though
  they can unconfingure that dependency), so most projects will have a line
  import * as gpio from '/path/to/pxexec/lib/gpio'.

- For ONLY main.ts: Any extension files in the user bundle will be imported,
  and we abuse prototypical inheritance to “merge” the extensions with their
  parent library object. In essence, we make it so that any access to the
  extension object will then fall back to the parent library object by setting
  it as the extension’s prototype, and then we replace the parent object with
  the extension object. This is a bit complicated to explain without getting
  into the meat of prototypical inheritance, but an example: if dmx.ts exists
  in the user bundle, the orchestrator will generate this code:

```typescript
import * as dmxEX from './dmx';
Object.setPrototypeOf(dmxEX, dmx);
dmx = dmxEX;
```

- For ONLY main.ts: The original text of main.ts is put into a wrapper function
  that is passed to core.main, which defines the operating semantics of the
  runtime environment and performs necessary setup. So, the user’s code is
  transformed into:

```typescript
_core.main(() => {
  // user's code here
});
```

- For ONLY `main.ts`: there is a bug with WebRTC in Node which causes it to
  fail to import in some contexts (including within fibers). In order to avoid
  this bug, we manually import WebRTC and store it in a specified location in
  the core library. The networking module later reads this location and uses it
  when initializing the signaling system. To insert the module, we add the
  following line:

```typescript
_core.hacks.wrtc = require('dss/client/node_modules/wrtc');
```

After transformation, the code is compiled using a thin typescript compiler
that does not perform typechecking or any other kind of analysis. It just
performs TypeScript-to-ES6 transformation (we do this to avoid having to load
the entire runtime into the compiler for every step, which takes 5-6 seconds on
even basic programs). The compiler outputs the compiled ES6 files to a
temporary folder in `/tmp`.

Finally, the executor (see
[pxexec-orchestrator/node\_executor.ts](https://github.com/LaboratoryForPlayfulComputation/pxexec-orchestrator/blob/master/src/node_executor.ts))
spawns a node child process to run the code. If a child process already exists
from a previous lifecycle iteration, it is killed first. If all of the above
steps succeed, then HTTP 200 is returned to the editor (so that it can show
that nifty toast notification informing the user that the new program is
running), otherwise, HTTP 500 is returned and an error message is displayed.

### Runtime

At last, the program is running. The behavior of the runtime is to, first,
resolve all imports and run the body of all of the imported library functions.
Those library bodies are where packages can configure initialization
requirements and teardown functions. The core library will run module
initialization in a predictable way. The import of core itself configures exit
handlers, so that the process can return some information to the orchestrator
when it exits (such as the status code, or an exception trace). Finally, the
node process encounters the `_core.main(() => { ... });` from earlier in our
`main.js` file.

`_core.main` spawns a fiber for the user code body. Fibers are a loose
concurrency model that allow us to neatly abstract over operating system
threads and to avoid unintentionally blocking the Node event loop, even though
we are running synchronous code that may contain diverging constructs such as
`while (1) {}`. Most importantly, they allow Node to interrupt the user’s
synchronous code to service events. From this point, the user’s code executes
apparently synchronously as written, and library service routines will detach
all user-code handlers into their own fibers.

The program will terminate normally when the node event loop has no more
configured awaiters (i.e. all fibers have terminated, all timers have expired,
and all event handlers are unconfigured). Some types of programs, by their
nature, will never terminate, due to having an unlimited wait time for an event
(such as “when I receive” blocks).

## Conclusion

I hope this has been an interesting and instructive (if overly technical) look
into some of the work that I’ve been up to for the past couple of years. While
I am no longer working on the BlockyTalky 3 project, I have begun developing my
own runtime and block-editing framework, [Serendipity](/project/serendipity),
which I hope will solve some of the more difficult design challenges discussed
above.  I view the Serendipity project in many ways as a spiritual successor to
my involvement in the BlockyTalky 3 work, and I hope that anyone who is
interested will continue to follow my progress on that project.

[bt3]: https://github.com/LaboratoryForPlayfulComputation/pxt-pi
[makecode]: https://github.com/Microsoft/pxt
