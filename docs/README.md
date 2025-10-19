---
title: Oglama SDK
description: Welcome to the Oglama SDK Documentation.
---

### Founding principles

Defined by the Roman architect Vitruvius, the principles of Strength, Utility, and Delight (firmitas, utilitas, venustas) form the foundation of good architecture, and they are central to Oglama's philosophy.

#### Firmitas

Oglama opens the door to automations that are simply not possible with other software. It maintains a good balance between AI flexibility and JavaScript programming determinism, all while protecting your privacy.

#### Utilitas

Effortlessly install, run, and edit modules directly from the built-in source code editor. The editor supports all the methods and properties listed below, and comes with autocomplete, auto-formatting, and linting.

#### Venustas

Oglama is designed with simplicity at its core. You don't need to be a programmer to get value from this software. Just search our repository for a module that solves your problem, install it, run it, and modify it to your liking.

* * *

### Getting started

Oglama is a desktop application built on top of Chromium that allows you to automate web actions.

[Create an account] and download the latest version to get started. Alternatively, clone [oglama/oglama] and run `npm start` to use the latest pre-release version.

#### 1\. Agents

Agents are autonomous browser windows, each operating in its own isolated session. For security reasons, agents do not have direct access to the file system or to each other's data. You must manually specify input files as needed.

#### 2\. Modules

Agents execute small JavaScript programs called modules. Each module defines inputs, outputs, functions, and a finite-state machine. For security reasons, modules do not have direct access to the browser window. Instead, the dollar sign object (`$`) is used to interact with the web page, pause execution, chat with a locally running large language model, fetch user input, save results, and more.

##### 2.1. Finite-state machine

Complex browser interactions are best modeled as a finite-state machine.

*   When a user starts an agent, the state machine begins execution from the entry state.
    
*   It then jumps from state to state with `return { next: "state-key" }`
    
*   The state machine stops if an error occurs or if no next state is specified.
    

##### 2.2. Functions

Functions are used to avoid code duplication within state machine states. Unlike states, a function's return value has no special significance. Functions can be called from either a state or another function using [$.fn("function-key")]

##### 2.3. Inputs and outputs

Each module can define up to nine inputs and nine outputs. This limit is intended to reduce cognitive load.

Supported input and output types are: `integer`, `string`, `boolean`,`table` and `files`

*   Inputs for each run are specified in the Settings tab. Fetch inputs in states and functions with [$.ioInput\*()]
    
*   Outputs are collected in the Results tab. Store outputs in states and functions with [$.ioOutput\*()]
    

#### 3\. Collaboration

You can publish your modules to the module repository, allowing others to install and use your work. You can also import and export your automations as `.oglama.yaml` files, giving you complete privacy and control over your work.

* * *

### Example

Here is an example of a small Oglama module (`search.oglama.yaml`) that performs a Google search.

Please note that the module is exported as a YAML file, but each function and finite-state machine state is written in pure JavaScript and relies on the dollar sign object (`$`) for operations.

**search.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      await $.fn("visit-google");
      await $.fn("search");
srcFunctions:
  - key: visit-google
    code: |
      // Navigate to Google and wait for page to load
      await $.navLoad("https://google.com/");
  - key: search
    code: |
      // Find the input field
      const inputKey = await $.doQuery("[name='q']");
      if ("string" !== typeof inputKey) {
        throw new Error("Could not find input field");
      }

      // Get search term from user settings
      const searchTerm = $.ioInputString("search-term");

      // Replace previous string and send enter key
      await $.doType(inputKey, searchTerm, { replace: true, submit: true });
srcInputs:
  - key: search-term
    type: string
    name: Search term
    desc: A search term for Google
    max: 1024
    options: []
    default: oglama
srcOutputs: []
```

Import this and other modules by following these steps:

*   Launch Oglama, select an agent, and click on Options
*   Click Source Code, go to Source Control, and select 'Import from YAML'
*   Choose the module file ( `search.oglama.yaml` )

* * *

### The SDK

> All available Oglama SDK methods and properties are described below. The SDK has a flat structure, with every method and property attached to the dollar sign object (`$`).

* * *

#### $.args

> {array}<br/>
> 
> - When used inside a <b>state</b>:<br/>
>  array passed by previous state as { next: 'state-key', args }<br/>
> 
> - When used inside a <b>function</b>:<br/>
>  array passed as the second argument to $.fn( 'function-key', args )<br/>

Although you can use [$.global\*()] methods to store and retrieve data from a global store, it is sometimes better to simply pass arguments from one state to another.

**example-args.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Generate a random number locally (or use $.osRand())
      const randomNumber = await $.fn("random", [2, 10]);

      // Pass it to the next state
      return { next: "surprise", args: [randomNumber] };
  - key: surprise
    code: |
      // Passed with { next: "surprise", args: [randomNumber] };
      const randomNumber = $.args[0];
      $.log(`The number I was thinking of was ${randomNumber}`);

      // Put the LLM to work
      const prompt = `Multiply ${randomNumber} by itself.`;
      const response = await $.llm(prompt);
srcFunctions:
  - key: random
    code: |
      if (2 !== $.args.length) {
        throw new Error("Expecting 2 arguments");
      }

      // Destructure the function arguments
      const [from, to] = $.args;

      // Generate a random number
      return Math.floor(Math.random() * (to - from + 1)) + from;
srcInputs: []
srcOutputs: []
```

* * *

#### $.current

> {string}<br/>
> 
> Current state key.<br/>

At the core of every Oglama module is a finite-state machine where each state is uniquely identified by its key.  
You may need to reference the current or the [$.previous] state key inside functions.

**example-current.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      await $.fn("write-haiku");

      return { next: "middle" };
  - key: middle
    code: |
      await $.fn("write-haiku");

      return { next: "end" };
  - key: end
    code: |
      await $.fn("write-haiku");
srcFunctions:
  - key: write-haiku
    code: |
      // The Old Pond by Matsuo Bashō (1644-1694)
      switch ($.current) {
        case "start":
          $.log("An old silent pond");

          break;

        case "middle":
          $.log("A frog jumps into the pond");

          break;

        case "end":
          $.log("Splash! Silence again.");
          break;
      }

      await $.sleep(1000);
srcInputs: []
srcOutputs: []
```

* * *

#### $.previous

> {string|null}<br/>
> 
> Previous state key or <i>null</i> if this is the entry state.<br/>

Following the example for the [$.current] property, here's how one would use the previous state key.

**example-previous.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      await $.fn("write-haiku");

      return { next: "middle" };
  - key: middle
    code: |
      await $.fn("write-haiku");

      return { next: "end" };
  - key: end
    code: |
      await $.fn("write-haiku");
srcFunctions:
  - key: write-haiku
    code: |
      // The Old Pond by Matsuo Bashō (1644-1694)
      switch ($.previous) {
        case null:
          $.log("An old silent pond");

          break;

        case "start":
          $.log("A frog jumps into the pond");

          break;

        case "middle":
          $.log("Splash! Silence again.");
          break;
      }

      await $.sleep(1000);
srcInputs: []
srcOutputs: []
```

* * *

#### async $.fn( fnKey, fnArgs = \[\] )

> Call a function (see the <i>Functions</i> tab).<br/>
> 
> <i>@param</i> {string} <b>fnKey</b> Function key<br/>
> <i>@param</i> {array} <b>fnArgs</b> (optional) Function arguments; accessed with <i>$.args</i><br/>

Functions allow you to organize your module better and prevent code duplication.  
Here's a simple example for counting down using functions - recursively.

**example-fn.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      await $.fn("countdown", [3]);

      $.log("That was easy");
srcFunctions:
  - key: countdown
    code: |
      const number = $.args[0];

      if (number <= 0) {
        return;
      }

      $.log(number, "success");
      await $.sleep(1000);

      // Recursion is fun!
      await $.fn("countdown", [number - 1]);
srcInputs: []
srcOutputs: []
```

* * *

#### async $.llm( prompt )

> Prompt the locally running large language model.<br/>
> 
> <i>@param</i> {string} <b>prompt</b> Prompt - up to 4096 characters long<br/>
> <i>@return</i> {string} LLM response<br/>
> <i>@throws</i> {Error} Throws an error if the LLM is not ready<br/>

Oglama provides easy access to a locally running large language model for complex tasks such as text summarization and sentiment analysis. Please note that LLMs are neither accurate nor deterministic.  
The example below shows when not to use a large language model: mathematical operations are much faster and more accurate in pure JavaScript.

**example-llm.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      const startTime = new Date().getTime();

      // Ask the magic box
      const response = await $.llm("Answer with one number: 2 + 3");

      // Log the execution time
      $.log(`Finished in ${(new Date().getTime() - startTime) / 1000} seconds`);
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.log( message, status = "info" )

> Append a message to the agent logs.<br/>
> 
> <i>@param</i> {any} <b>message</b> Message<br/>
> <i>@param</i> {"info"|"success"|"warning"|"error"} <b>status</b> (optional) Status; default <i>info</i><br/>

A maximum of 250 logs are retained in the logs panel for each agent. Logs are stored in session and are removed when the app is closed. There are four log types, each with their own color:`info`,`success`.`warning`, and`error`.

**example-log.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      $.log("🍰 A new cake recipe was added.", "info");
      $.log("🧁 The cake has baked successfully!", "success");
      $.log("🔥 The oven temperature is far too high.", "warning");
      $.log("💀 Cake failed to rise, check your ingredients.", "error");
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### async $.sleep( ms )

> Pause the execution of the current thread for a specified number of milliseconds.<br/>
> 
> <i>@param</i> {number} <b>ms</b> Sleep time in milliseconds<br/>

Sometimes you may need to slow down your script to prevent overloading a website's resources, and other times you might just want a bit of showmanship.

**example-sleep.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      $.log("It's the final countdown:", "warning");
      await $.sleep(1000);

      let counter = 5;
      while (counter-- > 0) {
        $.log(`⌛ ${counter + 1}...`);

        // Wait for it...
        await $.sleep(1000);
      }

      $.log("🚀 Liftoff!", "success");
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.pause( message = "" )

> Pause the execution of the current state indefinitely.<br/>
> When resumed, the current state will be re-executed from the start, not from the current line!<br/>
> 
> <i>@param</i> {string} <b>message</b> (optional) Message displayed in dialog when agent is (re-)selected<br/>

Oglama modules do not access or store any personal data, such as passwords or cookies.  
If a user needs to log into a website or verify they are human, simply pause the script at the current finite-state machine state and kindly request their input.

**example-pause.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Check if we already asked the user to log in
      if ($.globalRunGet("asked-user")) {
        return;
      }

      // Mark this so we don't enter an infinite loop
      $.globalRunSet("asked-user", true);

      // Look for button that is only available after login
      const elements = await $.doAwaitPresent(".dashboard-button", { timeout: 1 });

      // Ooops! (reCaptcha, login wall etc.)
      if (!elements) {
        $.pause("'Dubito, ergo cogito; cogito, ergo sum' to continue.");
      }
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.stop( message = "" )

> Stop the execution of the current state.<br/>
> When resumed, the finite-state machine will start from the first state (the Entry Point 🏁).<br/>
> 
> <i>@param</i> {string} <b>message</b> (optional) Message displayed in dialog when agent is (re-)selected<br/>

Unlike [$.pause()], the current run is **abandoned** so all values stored with [$.globalRunSet()] are discarded. The next time you start the agent, it will execute normally from the entry state.

**example-stop.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Check if we already asked the user to log in
      if (await $.globalEnvGet("asked-user")) {
        return;
      }

      // Mark this so we don't enter an infinite loop
      // Use the Environment store instead of the Run store
      await $.globalEnvSet("asked-user", true);

      // Look for button that is only available after login
      const elements = await $.doAwaitPresent(".dashboard-button", { timeout: 1 });

      // Ooops! (reCaptcha, login wall etc.)
      if (!elements) {
        // Abandon the current run (and clear values in the Run store)
        $.stop("State the meaning of life to continue.");
      }
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.setTimeout( callback, ms )

> Delays execution of a function by a specified number of milliseconds.<br/>
> 
> <i>@param</i> {function} <b>callback</b> JavaScript function<br/>
> <i>@param</i> {number} <b>ms</b> The number of milliseconds to wait before executing the callback<br/>
> <i>@return</i> {int} Timeout ID<br/>

Setting a timer is useful for a wide range of algorithms, but you will likely find it most valuable when setting up a listener for an event that has not occurred yet.

**example-setTimeout.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Go home
      await $.navLoad("about:home/");
      await $.doAwaitPresent(".mascot");

      // Click on mascot in the future
      $.setTimeout(async () => {
        const mascotKey = await $.doQuery(".mascot");

        await $.doClick(mascotKey);
      }, 1000);

      // Wait for mascot click to generate new quote
      await $.doAwaitPresent(".quote");

      // Wait for mouse to move away after click
      await $.sleep(500);
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.clearTimeout( timeoutId )

> Cancels a timeout previously established by <i>$.setTimeout</i>.<br/>
> 
> <i>@param</i> {function} <b>timeoutId</b> The identifier of the timeout to cancel, as returned by <i>$.setTimeout</i><br/>

**example-clearTimeout.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Set the bomb
      const timerId = $.setTimeout(async () => {
        $.log("💥 Boom!", "error");
      }, 500);

      // Clear the bomb
      $.clearTimeout(timerId);
      $.log("All clear", "success");
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### async $.osRequest( url, options = {} )

> OS: Make a request directly from the computer.<br/>
> 
> Useful for bypassing CORS (Cross-Origin Resource Sharing) constraints set by the browser.<br/>
> If you need to make an authenticated request from the currently loaded page, use <i>$.doRequest</i> instead.<br/>
> Note that these requests do not have access to your browser session's cookies.<br/>
> 
> <i>@param</i> {string} <b>url</b> Request URL<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Request options<br/>
> <i>@param</i> {string} <b>options.method</b> (optional) Request method; default <i>GET</i><br/>
> <i>@param</i> {object} <b>options.data</b> (optional) Request data; default <i>{}</i><br/>
> <i>@param</i> {object} <b>options.headers</b> (optional) Request headers; default <i>{}</i><br/>
> <i>@param</i> {boolean} <b>options.json</b> (optional) JSON request; default <i>true</i><br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Request timeout in seconds; default <i>60</i><br/>
> <i>@param</i> {boolean} <b>options.resData</b> (optional) Parse and return the response data; default <i>true</i><br/>
> <i>@return</i> {{ ok:boolean, status:number, headers:object, data:mixed}} Response object<br/>
> <i>@throws</i> {Error} Throws an error if request failed or aborted<br/>

This method acts like a proxy, bypassing any CORS restrictions.  
If you need to pass along cookies with your request, first nagivate to the target domain using [$.navLoad()] then issue the request with [$.doRequest()] or [$.ioSaveRequest()].

**example-osRequest.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      const url = "http://localhost:7199/manifest.json";

      // JSON request (CORS is bypassed)
      $.log(`Fetching ${url} from OS`, "success");
      const response = await $.osRequest(url);
      $.log([response?.status, response?.headers, response?.data]);

      // Fetch headers only
      $.log(`Fetching ${url} without data from OS`, "success");
      const responseNoData = await $.osRequest(url, { resData: false });
      $.log([responseNoData?.status, responseNoData?.headers, responseNoData?.data]);
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### async $.osShowFile( filePath )

> OS: Show file in folder.<br/>
> 
> <i>@param</i> {string} <b>filePath</b> File path<br/>
> <i>@return</i> {boolean}<br/>

It is recommended that you use this method with caution and only with the explicit permission of users, as opening folders may interfere with their activities.

**example-osShowFile.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      const url = "http://localhost:7199/manifest.json";

      // Save the file to disk
      const filePath = await $.ioSaveUrl("json-files", url);

      // Open containing folder (if the user allows it)
      if ($.ioInputBoolean("show-file-after-download")) {
        await $.osShowFile(filePath);
      }
srcFunctions: []
srcInputs:
  - key: show-file-after-download
    type: boolean
    name: Show files
    desc: Show downloaded files when the script finishes
srcOutputs:
  - key: json-files
    type: files
    name: JSON files
    desc: ""
    max: 1024
    extensions:
      - json
```

* * *

#### $.osRand( min, max, options = {} )

> OS: Generate a random signed integer between the specified minimum and maximum values (inclusive), or a random alphanumeric string with a length between the specified minimum and maximum values.<br/>
> 
> <i>@param</i> {int} <b>min</b> Minimum signed integer value OR minimum string length<br/>
> <i>@param</i> {int} <b>max</b> Maximum signed integer value OR maximum string length<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Random generator options<br/>
> <i>@param</i> {boolean} <b>options.string</b> (optional) Return a random string instead; default <i>false</i>' if true, <i>min</i> and <i>max</i> define the length of the returned string<br/>
> <i>@return</i> {int|string} A random signed integer between <i>min</i> and <i>max</i> (inclusive) OR a random string between <i>min</i> and <i>max</i> characters long, but not longer than 512 characters<br/>

Introducing randomness into the behavior of modules is so useful that we decided to dedicate a helper function to it.  
You could use `Math.floor(Math.random() * (max - min + 1)) + min` instead, but this is cleaner.

**example-osRand.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      const minTemp = -15;
      const maxTemp = 25;

      const predictedTemp = $.osRand(minTemp, maxTemp);

      $.log(`🌤️ Forecast says: ${predictedTemp}°C`, "success");

      switch (true) {
        case predictedTemp < 0:
          const randomString = $.osRand(3, 4, { string: true });
          $.log(`Brrr${randomString}... this is cold! 🥶`);
          break;

        case predictedTemp < 20:
          $.log("Perfect for a walk. 😄");
          break;

        default:
          $.log("Time for some ice cream. 😎");
          break;
      }
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### async $.globalEnvGet( envKey = null )

> Environment globals: Get environment variable(s). Values are JSON serializable.<br/>
> These values persit between runs but are reset on module install, fork or release.<br/>
> 
> <i>@param</i> {string|null} <b>envKey</b> (optional) Environment variable key or <i>null</i> for all values as a key-value object; default <i>null</i><br/>
> <i>@return</i> {object|any|null}<br/>

In this example, we're using the environment cache to perform an action only once per day.

**example-globalEnvGet.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Prepare date in YYYY-MM-DD format
      const dateToday = new Date().toISOString().split("T")[0];

      // Fetch the date stored in environment storage (persistent between runs)
      const dateStored = await $.globalEnvGet("date");

      if (dateToday !== dateStored) {
        // Do something new!
        $.log("New day, new possibilities ☀️");

        // Store today in environment storage
        await $.globalEnvSet("date", dateToday);
      }
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### async $.globalEnvSet( envKey, envValue )

> Environment globals: Set environment variable. Values must be JSON serializable.<br/>
> These values persit between runs but are reset on module install, fork or release.<br/>
> 
> <i>@param</i> {string} <b>envKey</b> Environment variable key<br/>
> <i>@param</i> {any|null} <b>envValue</b> Environment variable value; if <i>null</i>, the key is removed<br/>
> <i>@throws</i> {Error} If total environment storage size exceeded 512kB for this agent<br/>
> <i>@return</i> {boolean}<br/>

In this example, we're listing and removing all values from the environment storage.

**example-globalEnvSet.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      // Set some random values to environment storage (persistent between runs)
      for (let i = 1; i <= 3; i++) {
        const randomInt = $.osRand(10, 99);
        const randomString = $.osRand(10, 15, { string: true });
        await $.globalEnvSet(`key-${randomInt}`, randomString);
      }

      // Get all values stored in environment storage
      const envValues = await $.globalEnvGet();

      // Log them as an object
      $.log(envValues);

      // Log them individually
      for (const envKey of Object.keys(envValues)) {
        $.log(`✨ ${envKey} = ${envValues[envKey]}`, "success");
      }

      // Clean the cache
      for (const envKey of Object.keys(envValues)) {
        // Setting value to null deletes it from the environment storage
        await $.globalEnvSet(envKey, null);
        $.log(`🗑️ ${envKey} environment value removed`, "warning");
      }
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.globalRunGet( runKey = null )

> Run globals: Get global variable(s) for this run.<br/>
> These values are reset before each run.<br/>
> 
> <i>@param</i> {string|null} <b>runKey</b> (optional) Run variable key or <i>null</i> for all values as a key-value object; default <i>null</i><br/>
> <i>@return</i> {object|any|null}<br/>

* * *

#### $.globalRunSet( runKey, runValue )

> Run globals: Set global variable for this run.<br/>
> These values are reset before each run.<br/>
> 
> <i>@param</i> {string} <b>runKey</b> Run variable key<br/>
> <i>@param</i> {any|null} <b>runValue</b> Run variable value; if <i>null</i>, the key is removed<br/>
> <i>@return</i> {boolean}<br/>

* * *

#### $.ioInputInt( ioKey )

> IO: Get input integer(s).<br/>
> This method returns an integer or an array of integers based on the selected input format.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Integer input key<br/>
> <i>@return</i> {int | int[] | null} Integer(s) supplied by user or <i>null</i> if input is not of type <i>integer</i><br/>

* * *

#### $.ioInputString( ioKey )

> IO: Get input string(s).<br/>
> This method returns a string or an array of strings based on the selected input format.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> String input key<br/>
> <i>@return</i> {string | string[] | null} String(s) supplied by user or <i>null</i> if input is not of type <i>string</i><br/>

* * *

#### $.ioInputBoolean( ioKey )

> IO: Get input boolean.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Boolean input key<br/>
> <i>@return</i> {boolean | null} Boolean supplied by user or <i>null</i> if input is not of type boolean<br/>

* * *

#### async $.ioInputRow( ioKey, index = null )

> IO: Get the next available table row and increment index internally.<br/>
> Alternatively, get the row at the specified index.<br/>
> Row keys are declared in the <i>Inputs</i> tab for this table input.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Table input key<br/>
> <i>@param</i> {int} <b>index</b> (optional) Table index; default <i>null</i><br/>
> <i>@return</i> {Object&lt;string,string&gt; | null} Current row or <i>null</i> if reached the end of the table or if input is not of type <i>table</i><br/>

* * *

#### $.ioInputFiles( ioKey )

> IO: Get input file paths.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Files input key<br/>
> <i>@return</i> {string[] | null} Input file paths or <i>null</i> if input is not of type <i>files</i><br/>

* * *

#### async $.ioOutputInt( ioKey, int )

> IO: Set output integer.<br/>
> Subsequent calls override previous values.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Integer output key<br/>
> <i>@param</i> {int} <b>int</b> Integer<br/>
> <i>@return</i> {boolean} True on success, false for invalid integer or if output is not of type <i>integer</i><br/>

* * *

#### async $.ioOutputString( ioKey, string )

> IO: Set output string.<br/>
> Subsequent calls override previous values.<br/>
> 
> Warning: string will be truncated to <b>1024</b> characters.<br/>
> If you need to store longer strings, use <i>$.ioSaveText</i> instead.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> String output key<br/>
> <i>@param</i> {string} <b>string</b> String<br/>
> <i>@return</i> {boolean} True on success, false for invalid string or if output is not of type <i>string</i><br/>

* * *

#### async $.ioOutputBoolean( ioKey, boolean )

> IO: Set output boolean.<br/>
> Subsequent calls override previous values.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Boolean output key<br/>
> <i>@param</i> {boolean} <b>boolean</b> Boolean value<br/>
> <i>@return</i> {boolean} True on success, false for invalid boolean or if output is not of type <i>boolean</i><br/>

* * *

#### async $.ioOutputRow( ioKey, row )

> IO: Append a row to output table.<br/>
> Row keys are declared in the <i>Outputs</i> tab for this table output.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Table output key<br/>
> <i>@param</i> {Object&lt;string,string&gt;} <b>row</b> Row object<br/>
> <i>@return</i> {boolean} True on success, false for invalid row object or if output is not of type <i>table</i><br/>

* * *

#### async $.ioSaveText( ioKey, text, options = {} )

> IO: Save text to disk as new file.<br/>
> 
> Useful for saving arbitrary strings in custom formats like JSON, YAML, INI etc.<br/>
> For strings that are shorter than or equal to <b>1024</b> characters, you can use <i>$.ioOutputString</i>.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Files output key<br/>
> <i>@param</i> {string} <b>text</b> Text to save<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Save options<br/>
> <i>@param</i> {string} <b>options.extension</b> (optional) File extension; default <i>null</i>; must match one of the extensions declared in output; falls back to first file extension declared in output<br/>
> <i>@return</i> {string | null} File path on success, null if download failed or if output is not of type <i>files</i><br/>

* * *

#### async $.ioSaveDownload( ioKey, options = {} )

> IO: Capture the next downloaded file and save it to disk.<br/>
> 
> Useful for saving any file download, regardless of how it was trigerred.<br/>
> Defer the download event with <i>$.setTimeout()</i> before calling <i>$.ioSaveDownload</i>.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Files output key<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Save options<br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Download timeout in seconds; default <i>600</i><br/>
> <i>@param</i> {string} <b>options.extension</b> (optional) File extension; default <i>null</i>; must match one of the extensions declared in output; falls back to first file extension declared in output<br/>
> <i>@return</i> {string | null} File path on success, <i>null</i> if download failed or if output is not of type <i>files</i><br/>

* * *

#### async $.ioSaveScreenshot( ioKey, options = {} )

> IO: Capture a screenshot of the current page and save it to disk.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Files output key<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Save options<br/>
> <i>@param</i> {string} <b>options.extension</b> (optional) File extension; default <i>null</i>; must match one of the extensions declared in output; falls back to first file extension declared in output<br/>
> <i>@return</i> {string | null} File path on success, <i>null</i> if download failed or if output is not of type <i>files</i><br/>

* * *

#### async $.ioSaveUrl( ioKey, url, options = {} )

> IO: Capture the file stored at this URL and save it to disk.<br/>
> 
> Useful for saving files that are not available as links on page.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Files output key<br/>
> <i>@param</i> {string} <b>url</b> URL to download<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Save options<br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Download timeout in seconds; default <i>600</i><br/>
> <i>@param</i> {string} <b>options.extension</b> (optional) File extension; default <i>null</i>; must match one of the extensions declared in output; falls back to first file extension declared in output<br/>
> <i>@return</i> {string | null} File path on success, <i>null</i> if download failed or if output is not of type <i>files</i><br/>

* * *

#### async $.ioSaveRequest( ioKey, url, options = {} )

> IO: Capture the result of this fetch request and save it to disk.<br/>
> 
> Useful for saving the result of fetch requests made from the current domain/page.<br/>
> For direct access to JSON or text responses, use <i>$.doRequest</i> instead.<br/>
> 
> <i>@param</i> {string} <b>ioKey</b> Files output key<br/>
> <i>@param</i> {string} <b>url</b> URL to save locally<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Request options<br/>
> <i>@param</i> {string} <b>options.method</b> (optional) Request method; default <i>GET</i><br/>
> <i>@param</i> {object} <b>options.data</b> (optional) Request data; default <i>{}</i><br/>
> <i>@param</i> {object} <b>options.headers</b> (optional) Request headers; default <i>{}</i><br/>
> <i>@param</i> {boolean} <b>options.json</b> (optional) JSON request; default <i>true</i><br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Request timeout in seconds; default <i>60</i><br/>
> <i>@param</i> {string} <b>options.extension</b> (optional) File extension; default <i>null</i>; must match one of the extensions declared in output; falls back to first file extension declared in output<br/>
> <i>@return</i> {string | null} File path on success, <i>null</i> if download failed or if output is not of type <i>files</i><br/>

This example demonstrates how to save a file with a custom extension. Note that the file extension must first be declared in the output configuration. If the specified extension is not included in the declared list, the first listed extension, "json" in this case, will be used instead.  
If you don't specify a file extension, the script will attempt to deduce it from the URL.

**example-ioSaveRequest.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      const url = "http://localhost:7199/manifest.json";

      // Navigate to origin so the browser allows the request (CORS)
      await $.navLoad(new URL(url).origin);

      $.log("Saving JSON as simple text file...");
      const jsonPath = await $.ioSaveRequest("manifest", url, { extension: "txt" });
      if ("string" !== typeof jsonPath) {
        throw new Error("$.ioSaveRequest failed");
      }
srcFunctions: []
srcInputs: []
srcOutputs:
  - key: manifest
    type: files
    name: JSON files
    desc: ""
    max: 1024
    extensions:
      - json
      - txt
```

* * *

#### async $.navLoad( url )

> Navigation: Open URL and wait for the page to load.<br/>
> 
> <i>@param</i> {string} <b>url</b> URL; only <i>http</i> and <i>https</i> protocols are allowed<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>
> <i>@throws</i> {Error} Throws an error if page failed to load<br/>

* * *

#### async $.navReload( skipCache = false )

> Navigation: Reload the current page.<br/>
> 
> <i>@param</i> {boolean} <b>skipCache</b> (optional) Reload skipping the cache; default <i>false</i><br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>
> <i>@throws</i> {Error} Throws an error if page failed to load<br/>

* * *

#### async $.navGoBack()

> Navigation: Go backwards.<br/>
> 
> <i>@return</i> {boolean} True on success, false on failure<br/>
> <i>@throws</i> {Error} Throws an error if page failed to load<br/>

* * *

#### async $.navGoForward()

> Navigation: Go forwards.<br/>
> 
> <i>@return</i> {boolean} True on success, false on failure<br/>
> <i>@throws</i> {Error} Throws an error if page failed to load<br/>

* * *

#### async $.navGetUrl()

> Navigation: Get the URL of the current page.<br/>
> 
> <i>@return</i> {string}<br/>

* * *

#### async $.navGetTitle()

> Navigation: Get the title of the current page.<br/>
> 
> <i>@return</i> {string}<br/>

* * *

#### async $.handleAlert()

> Browser: Prevent the next <i>window.alert()</i> from bubbling.<br/>
> 
> Unhandled alert dialogs are automatically closed,<br/>
> and their message is passed as a toast notification.<br/>

* * *

#### async $.handleConfirm( accept = true )

> Browser: Prevent the next <i>window.confirm()</i> from bubbling, and either accept or reject it.<br/>
> 
> Unhandled confirmation dialogs are automatically accepted,<br/>
> and their message is passed as a toast notification.<br/>
> 
> <i>@param</i> {boolean} <b>accept</b> (optional) Accept or reject the next confirmation message; default <i>true</i><br/>

* * *

#### async $.handlePrompt( response = "" )

> Browser: Prevent the next <i>window.prompt()</i> from bubbling, and answer it.<br/>
> 
> Unhandled prompts automatically return with their default value or an empty string,<br/>
> and their message is passed as a toast notification.<br/>
> 
> <i>@param</i> {string} <b>response</b> (optional) Prompt response text<br/>

* * *

#### async $.doQuery( selector, options = {} )

> Document: Find the first HTML element that matches the CSS selector and return its corresponding <i>Element key</i>.<br/>
> 
> <i>@param</i> {string} <b>selector</b> Standard CSS selector<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {string} <b>options.parent</b> (optional) Parent Element key; default <i>null</i> to search the entire Document<br/>
> <i>@param</i> {string} <b>options.contains</b> (optional) Text contained by Element (case insensitive); default <i>null</i> for no restrictions<br/>
> <i>@param</i> {boolean} <b>options.fromScreenView</b> (optional) Restrict results to elements placed from the current scroll position down; default <i>false</i><br/>
> <i>@return</i> {string|null} <i>Element key</i> on null on error<br/>

* * *

#### async $.doQueryAll( selector, options = {} )

> Document: Find all HTML elements that match the CSS selector and return their corresponding <i>Element keys</i>.<br/>
> 
> <i>@param</i> {string} <b>selector</b> Standard CSS selector<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {string} <b>options.parent</b> (optional) Parent Element key; default <i>null</i> to search the entire Document<br/>
> <i>@param</i> {string} <b>options.contains</b> (optional) Text contained by Element (case insensitive); default <i>null</i> for no restrictions<br/>
> <i>@param</i> {boolean} <b>options.fromScreenView</b> (optional) Restrict results to elements placed from the current scroll position down; default <i>false</i><br/>
> <i>@return</i> {string[]} Array of <i>Element keys</i><br/>

* * *

#### async $.doQueryParent( elKey, options = {} )

> Document: Find the parent of this HTML element that matches the CSS selector and return its corresponding <i>Element key</i>.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {string} <b>options.selector</b> (optional) CSS selector for parent element; default <i>null</i> to stop at first ancestor<br/>
> <i>@param</i> {string} <b>options.contains</b> (optional) Text contained by Element (case insensitive); default <i>null</i> for no restrictions<br/>
> <i>@return</i> {string|null} <i>Element key</i> on null on error<br/>

* * *

#### async $.doQuerySiblings( elKey, options = {} )

> Document: Find the siblings of this HTML element that match the CSS selector and return their corresponding <i>Element keys</i>.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {string} <b>options.selector</b> (optional) CSS selector for parent element; default <i>null</i> to stop at first ancestor<br/>
> <i>@param</i> {string} <b>options.contains</b> (optional) Text contained by Element (case insensitive); default <i>null</i> for no restrictions<br/>
> <i>@return</i> {string[]} <i>Element keys</i><br/>

* * *

#### async $.doQueryAt( left, top, options = {} )

> Document: Find the first HTML element that matches the CSS selector at the specified coordinates,<br/>
> and return its corresponding <i>Element key</i>.<br/>
> 
> <i>@param</i> {int} <b>left</b> Left coordinate in pixels<br/>
> <i>@param</i> {int} <b>top</b> Top coordinate in pixels<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {string} <b>options.selector</b> (optional) CSS selector for parent element; default <i>null</i> to stop at first ancestor<br/>
> <i>@param</i> {string} <b>options.contains</b> (optional) Text contained by Element (case insensitive); default <i>null</i> for no restrictions<br/>
> <i>@return</i> {string|null} <i>Element key</i> on null on error<br/>

* * *

#### async $.doRequest( url, options = {} )

> Document: Make a request from the current page.<br/>
> 
> Useful for JSON and simple text responses. For large files or binary data use <i>$.ioSaveRequest</i> instead.<br/>
> If you need to bypass CORS and send the requests directly from the computer (outside of the browser session) use <i>$.osRequest</i> instead.<br/>
> 
> <i>@param</i> {string} <b>url</b> Request URL<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Request options<br/>
> <i>@param</i> {string} <b>options.method</b> (optional) Request method; default <i>GET</i><br/>
> <i>@param</i> {object} <b>options.data</b> (optional) Request data; default <i>{}</i><br/>
> <i>@param</i> {object} <b>options.headers</b> (optional) Request headers; default <i>{}</i><br/>
> <i>@param</i> {boolean} <b>options.json</b> (optional) JSON request; default <i>true</i><br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Request timeout in seconds; default <i>60</i><br/>
> <i>@param</i> {boolean} <b>options.resData</b> (optional) Parse and return the response data; default <i>true</i><br/>
> <i>@return</i> {{ ok:boolean, status:number, headers:object, data:mixed}} Response object<br/>
> <i>@throws</i> {Error} Throws an error if request failed or aborted<br/>

This example describes how to fetch data in the browser when CORS is an issue.  
If you don't care about cookies you can use [$.osRequest()] to bypass CORS instead.

**example-doRequest.oglama.yaml**
```yaml
srcStateMachine:
  - key: start
    code: |
      const url = "http://localhost:7199/manifest.json";

      // Navigate to origin so the browser allows the request (CORS)
      await $.navLoad(new URL(url).origin);

      // JSON request
      $.log(`Fetching ${url} from browser`, "success");
      const response = await $.doRequest(url);
      $.log([response?.status, response?.headers, response?.data]);

      // Fetch headers only
      $.log(`Fetching ${url} without data from browser`, "success");
      const responseNoData = await $.doRequest(url, { resData: false });
      $.log([responseNoData?.status, responseNoData?.headers, responseNoData?.data]);
srcFunctions: []
srcInputs: []
srcOutputs: []
```

* * *

#### $.doTick( name, amount = 1 )

> Document: Increment a named counter in the Status bar.<br/>
> Up to 5 counters can be displayed at a time.<br/>
> 
> <i>@param</i> {string} <b>name</b> Counter name. The following strings are displayed as icons:<br/>
>  "contact", "view", "like", "post", "repost", "comment",<br/>
>  "upload", "download", "screenshot", "collect",<br/>
>  "success", "warning"<br/>
> <i>@param</i> {int} <b>amount</b> (optional) Strictly positive number; <i>[0,1000]</i>; default <i>1</i>; <i>0</i> won't increment the counter<br/>

* * *

#### async $.doHighlight( elKey, scrollIntoView = false )

> Document: Highlight an HTML Element in the viewport for 1 second.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {boolean} <b>scrollIntoView</b> (optional) Scroll element into view before highlighting; default <i>false</i><br/>

* * *

#### async $.doHighlightBox( box )

> Document: Highlight a box in the viewport for 1 second.<br/>
> 
> <i>@typedef</i> {Object} Box<br/>
> <i>@property</i> {int} <b>left</b> Left coordinate in pixels<br/>
> <i>@property</i> {int} <b>top</b> Top coordinate in pixels<br/>
> <i>@property</i> {int} <b>width</b> Width in pixels<br/>
> <i>@property</i> {int} <b>height</b> Height in pixels<br/>
> 
> <i>@param</i> {Box} <b>box</b> Box - obtained with <i>$.doGetBox</i><br/>

* * *

#### async $.doGetBox( elKey )

> Document: Get the box of any HTML Element.<br/>
> 
> <i>@typedef</i> {Object} Box<br/>
> <i>@property</i> {int} <b>left</b> Left coordinate in pixels<br/>
> <i>@property</i> {int} <b>top</b> Top coordinate in pixels<br/>
> <i>@property</i> {int} <b>width</b> Width in pixels<br/>
> <i>@property</i> {int} <b>height</b> Height in pixels<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@return</i> {Box|null} Element box or <i>null</i> on error<br/>

* * *

#### async $.doGetType( elKey )

> Document: Get the type of an HTML Element.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@return</i> {string|null} Element type or <i>null</i> on error<br/>

* * *

#### async $.doGetValue( elKey )

> Document: Get the value of an HTML Element. Supported elements:<br/>
> - <i>input</i> (includes <i>checkbox</i> and <i>radio</i>)<br/>
> - <i>textarea</i><br/>
> - <i>select</i><br/>
> 
> Returns multiple values for checboxes and multi-select.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@return</i> {string|string[]|boolean|null} Value or <i>null</i> on error<br/>

* * *

#### async $.doGetAttribute( elKey, attr )

> Document: Get a single HTML Element attribute.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {string} <b>attr</b> HTML attribute (lowercase)<br/>
> <i>@return</i> {string|null} Attribute value or <i>null</i> on error<br/>

* * *

#### async $.doGetAttributes( elKey, attrs = \[\] )

> Document: Get all or multiple HTML Element attributes.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {string[]} <b>attrs</b> (optional) HTML attributes (lowercase) or empty array for all; default <i>[]</i><br/>
> <i>@return</i> {Object&lt;string,string&gt;} Map of attribute and value<br/>

* * *

#### async $.doGetContent( elKey, asHtml = false )

> Document: Get the content of an HTML Element.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {boolean} <b>asHtml</b> (optional) Use innerHTML instead of innerText; default <i>false</i><br/>
> <i>@return</i> {string|null} Element contents or <i>null</i> on error<br/>

* * *

#### async $.doGetVisible( elKey )

> Document: Get whether HTML Element is visible on page.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@return</i> {boolean}<br/>

* * *

#### async $.doGetInViewport( elKey )

> Document: Get whether HTML Element is even partially located in the viewport.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@return</i> {boolean}<br/>

* * *

#### async $.doGetViewportSize()

> Document: Get the viewport size<br/>
> 
> <i>@return</i> {{width: int, height: int}}<br/>

* * *

#### async $.doClick( elKey, options = {} )

> Document: Click or double-click on HTML Element.<br/>
> Automatically scroll to Element before action.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {Object} <b>options</b> (optional) Click options<br/>
> <i>@param</i> {boolean} <b>options.double</b> (optional) Double-click; default <i>false</i><br/>
> <i>@param</i> {boolean} <b>options.hover</b> (optional) Hover after click; default <i>false</i> to move mouse to the side after clicking<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doClickAt( left, top, options = {} )

> Document: Click or double-click at coordinates in viewport.<br/>
> 
> <i>@param</i> {int} <b>left</b> Left coordinate in pixels<br/>
> <i>@param</i> {int} <b>top</b> Top coordinate in pixels<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Click options<br/>
> <i>@param</i> {boolean} <b>options.double</b> (optional) Double-click; default <i>false</i><br/>
> <i>@param</i> {boolean} <b>options.hover</b> (optional) Hover after click; default <i>false</i> to move mouse to the side after clicking<br/>
> 
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doHover( elKey )

> Document: Move mouse over an HTML Element.<br/>
> Automatically scroll to Element before action.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doHoverAt( left, top )

> Document: Move mouse to coordinates in viewport.<br/>
> 
> <i>@param</i> {int} <b>left</b> Left coordinate in pixels<br/>
> <i>@param</i> {int} <b>top</b> Top coordinate in pixels<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doHoverCenter()

> Document: Move mouse just slightly outside of viewport center.<br/>
> 
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doJiggle( radius = 50 )

> Document: Jiggle the mouse at the current coordinates.<br/>
> 
> <i>@param</i> {int} <b>radius</b> (optional) Jiggle radius in pixels; [10,500]; default <i>50</i><br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doScroll( amount, vertical = true )

> Document: Scroll on page.<br/>
> 
> <i>@param</i> {int} <b>amount</b> Scroll amount in pixels<br/>
> <i>@param</i> {boolean} <b>vertical</b> (optional) Vertical or Horizontal scroll; default <i>true</i> for vertical<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doScrollTo( elKey, marginTop = 0 )

> Document: Scroll to HTML Element.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {int} <b>marginTop</b> (optional) Top margin; default <i>0</i><br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doType( elKey, text, options = {} )

> Document: Type text to specified HTML element.<br/>
> Automatically scroll to Element before action.<br/>
> Skips typing if the Element is not editable.<br/>
> 
> To save time on large texts, the first part of the text is pasted,<br/>
> and only the last 250 characters are typed one character at a time.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {string} <b>text</b> Text to type<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Typing options<br/>
> <i>@param</i> {boolean} <b>options.replace</b> (optional) Replace current text; default <i>false</i> to append text<br/>
> <i>@param</i> {boolean} <b>options.submit</b> (optional) Press the <i>Enter</i> key at the end; default <i>false</i><br/>
> <i>@param</i> {int} <b>options.speed</b> (optional) Typing speed in characters per second; <i>[1,250]</i>; default <i>5</i><br/>
> <i>@return</i> {boolean} Returns <i>false</i> if target element is not editable<br/>

* * *

#### async $.doTypeAt( left, top, text, options = {} )

> Document: Type text at coordinates in viewport.<br/>
> 
> To save time on large texts, the first part of the text is pasted,<br/>
> and only the last 250 characters are typed one character at a time.<br/>
> 
> <i>@param</i> {int} <b>left</b> Left coordinate in pixels<br/>
> <i>@param</i> {int} <b>top</b> Top coordinate in pixels<br/>
> <i>@param</i> {string} <b>text</b> Text to type<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Typing options<br/>
> <i>@param</i> {boolean} <b>options.replace</b> (optional) Replace current text; default <i>false</i> to append text<br/>
> <i>@param</i> {boolean} <b>options.submit</b> (optional) Press the <i>Enter</i> key at the end; default <i>false</i><br/>
> <i>@param</i> {int} <b>options.speed</b> (optional) Typing speed in characters per second; <i>[1,250]</i>; default <i>5</i><br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doSelect( elKey, values )

> Document: Select zero, one or more options, replacing previous selection.<br/>
> Automatically scroll to Element before action.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {string|string[]} <b>values</b> A single value or an array of values for &lt;select multiple/&gt;<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doCheck( elKey, values )

> Document: Check radio or checkbox values. The element's siblings must share the same name attribute.<br/>
> Automatically scroll to Element(s) before action.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {string|string[]} <b>values</b> A single value or an array of values<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doChooseFiles( elKey, filePaths )

> Document: Choose files for <i>&lt;input type="file" /&gt;</i> HTML Element.<br/>
> Automatically scroll to Element before action.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i><br/>
> <i>@param</i> {string|string[]} <b>filePaths</b> File path(s)<br/>
> <i>@return</i> {boolean} True on success, false on failure<br/>

* * *

#### async $.doAwaitPresent( selector, options = {} )

> Document: Wait for an Element to be present in the DOM.<br/>
> 
> <i>@param</i> {string} <b>selector</b> Standard CSS selector<br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {string} <b>options.contains</b> (optional) Text contained by Element (case insensitive); default <i>null</i> for no restrictions<br/>
> <i>@param</i> {string} <b>options.parent</b> (optional) Parent Element key; default <i>null</i> to search the entire Document<br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Timeout in seconds; default <i>60</i><br/>
> <i>@param</i> {boolean} <b>options.first</b> (optional) Return only the first Element's key (<i>string</i> instead of <i>string[]</i>); default <i>false</i><br/>
> <i>@return</i> {string[]|string|null} Array of <i>Element keys</i> OR <i>Element key</i> if <i>options.first</i> OR <i>null</i> on timeout<br/>

* * *

#### async $.doAwaitNotPresent( elKey, options = {} )

> Document: Wait for an Element to be removed from the DOM.<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i> or <i>$.doAwaitPresent</i><br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Timeout in seconds; default <i>60</i><br/>
> <i>@return</i> {boolean} True on success, false on timeout<br/>

* * *

#### async $.doAwaitVisible( elKey, options = {} )

> Document: Wait for an Element to become visible to the user (display, visibility, opacity).<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i> or <i>$.doAwaitPresent</i><br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Timeout in seconds; default <i>60</i><br/>
> <i>@return</i> {boolean} True on success, false on timeout<br/>

* * *

#### async $.doAwaitNotVisible( elKey, options = {} )

> Document: Wait for an Element to become invisible to the user (display, visibility, opacity).<br/>
> 
> <i>@param</i> {string} <b>elKey</b> Element key - obtained with <i>$.doQuery</i> or <i>$.doAwaitPresent</i><br/>
> <i>@param</i> {Object} <b>options</b> (optional) Query options<br/>
> <i>@param</i> {int} <b>options.timeout</b> (optional) Timeout in seconds; default <i>60</i><br/>
> <i>@return</i> {boolean} True on success, false on timeout<br/>

[Create an account]: https://oglama.com/login/
[oglama/oglama]: https://github.com/oglama/oglama/
[$.fn("function-key")]: https://oglama.com/docs/#/doc:fn
[$.ioInput\*()]: https://oglama.com/docs/#/doc:ioInputInt
[$.ioOutput\*()]: https://oglama.com/docs/#/doc:ioOutputInt
[$.global\*()]: https://oglama.com/docs/#/doc:globalEnvGet
[$.previous]: https://oglama.com/docs/#/doc:previous
[$.current]: https://oglama.com/docs/#/doc:current
[$.pause()]: https://oglama.com/docs/#/doc:pause
[$.globalRunSet()]: https://oglama.com/docs/#/doc:globalRunSet
[$.navLoad()]: https://oglama.com/docs/#/doc:navLoad
[$.doRequest()]: https://oglama.com/docs/#/doc:doRequest
[$.ioSaveRequest()]: https://oglama.com/docs/#/doc:ioSaveRequest
[$.osRequest()]: https://oglama.com/docs/#/doc:osRequest