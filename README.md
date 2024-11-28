# Wednesday
[![Go Report Card](https://goreportcard.com/badge/github.com/DazFather/Wednesday)](https://goreportcard.com/report/github.com/DazFather/Wednesday)


**wed**nesday is a front-end web framework that aims to simplify things:

- **No Node.js server**, no Deno, no Bun, no server! Build your own if you need one.
- **No virtual DOM, no re-renders**, no extra calls, no extra bundle size. The DOM is already there.
- **No mixed client-server code**, no headaches.

## 1. Installation
Build Wednesday with:
```shell
go build -o wed
```
> To install it, you can simply follow the official Go (mini) tutorial [here](https://go.dev/doc/tutorial/compile-install), as with any other Go project.

## 2. Initialize a project
To initialize the current working directory as a new Wednesday project. Use the command 
```shell
wed init
```
> It will create all necessary files and directories, which you can customize later via the `wed-settings.json`.

If you already have a settings file, you can initialize the project accordingly by passing the path to the file using the "settings" (or the alias "s") flag.
> For example: `wed init -s=path/to/my-settings-file.json`
> 
> This can be a great way to create a new project with the same structure of another one

To verify everything is in place, simply serve it using `wed serve` and visit `http://localhost:8080`; you should see a welcome page.
> You can specify the port using the "port" (or "p") flag. More on this later.

## 3. Writing a component
You can create a component by creating a file ending with `.wed.html` in any subdirectory of your project. In this file, you can specify the following top-level tags:

### `<style>`
This will host your CSS styles scoped for this component only (`.my-component-name.wed-component`).

### `<html>`
Within the `<html>` tag, you can put the HTML structure of your component. This will be wrapped in a `<div class="my-component-name wed-component">` that will have a "display" of "contents".
> WED relies on the [Go template engine](https://go.dev/pkg/text/template) for this, and the code will be executed only once, when the project is built. Both commands `wed build` and `wed serve` will build the project.
> Thanks to this, the default WED component has zero extra runtime cost. Use them as much as you like; the client won't incur additional load.

Let's see how to use them in some practical examples:

#### Invoking a ~~component~~ template
The first scenario where the template system is useful is when we want to use a component.
To do this, we can simply use `{{ template "my-component-name" }}`, and the content of the HTML for the specified component will be inserted there.

#### Passing values between templates
You can also pass a list of values between templates by using the provided `args` function. For example:
```html
<html>
    <h1>Concept Bucket</h1>
    <strong>List of ideas that came to my mind:</strong>
    {{ template "bucket-list" args "pippo" 3.14 "banana" false }}
</html>
```
> `parent.wed.html`

```html
<html>
    <p>
        First thought: <mark>{{ index . 0 }}</mark><br>
        Full list below:
    </p>
    <ul>
        {{ range . }} <li>{{ . }}</li> {{ end }}
    <ul>
<html>
```
> `bucket-list.wed.html`

#### Defining ~~snippets~~ inner templates
Another useful feature from the templating language is the ability to define additional templates that can be used inside and outside your components.
> Note: When used outside, they will not carry the same style since they will be out of scope.
```html
<style>
    .magic-word { color: indigo; }
</style>
<html>
    <h1>{{ block "magic-word" "please" }}<strong class="magic-word">{{ . }}</strong>{{ end }}</h1>
    <strong>The importance of saying "{{ template "magic-word" }}"</strong>
    {{ template "child" }}
</html>
```
> `parent.wed.html`

```html
<style>
    .magic-word { color: limegreen; }
</style>
<html>
    <p>It's important for good communication to say {{ template "magic-word" }}</p>
<html>
```
> `child.wed.html`

#### Dynamic templates
Sometimes web pages need to add components dynamically, such as when a user click on a button or after an HTTP call.
Wednesday gots you cover, simply add the attribute "_type="dynamic"_" to the html tag
> By default Wednesday will interpret an empty or missing "_type_" attribute as "_static_"
> This is a precise strategy to improve client speed and reduce state management to only when necessary
  
In this way the html code will be wrapped again on a [template tag](https://developer.mozilla.org/en-US/docs/Web/HTML/Element/template) having as the only attribute an "_id_" with the component name as the value such as `<template id="pippo">...</template>`.
This template can be insert anywhere just by using `{{ importDynamic "component-name" }}`
> In praticular this template call accepts a variadic number of template names.
> It's possible to pass only the needed template_s_ or nothing to import them all

At this point by manipulating the DOM in the standard way is possible to insert the template content anywhere
> There is also an utility function [useTemplate](#useTemplate) that might cover many cases and help reduce the boilerplate

A dynamic template can include static templates, share datas and snippets with others and do all other action allowed by the template engine

### `<script>`
Here you can add JavaScript logic that will run once the page is fully loaded (`defer`). 
This script is shared across all components, giving access to helpful utilities that WED provides to enhance component reactivity.
The import order of the components is currently alphabetical based on the file name but it might change in the future.
If a component script require some definitions form another one it's possible to use the _require_ attribute giving as value a space-spareted components name (without the file extension)

#### useDisplay
When you want to update text on the screen:
```html
<html>
    <p class="display">loading...</p><!-- expected: Hello World! -->
<html>
<script>
    const show = useDisplay(".display", t => t + " World!")
    console.log(show("Hello")) // expected: Hello World!
</script>
```
> `app.wed.html`

#### useEffect
Runs a callback whenever the value changes:
```html
<html>
    <p class="display">loading...</p><!-- expected: 24,6 -->
<html>
<script>
    const arr = useEffect([], (() => {
        const show = useDisplay(".display")
        return v => { show(v.map(e => "" + (e * 2)).join(",")); return v }
    })())
    arr.value = [12,3]
</script>
```
> `app.wed.html`

#### useMirror
_A sibling of useDisplay_, it provides controlled access to properties of a DOM element, allowing you to specify one or a list of properties.
This can prevent accident where, especially on large codebases, a person might change the value of something is not supposed to.

#### useBinds
Binds properties between elements and an object, updating them in response to changes.
use the `bind` HTML attribute on the elements you want to bind and then put the following values separated by '`:`'. Only the first is required:
- The property of the element that you need to bind (ex. "innerText")
  > Notice that even though "class" is an HTML attribute, in JS to modify it directly you need to use "className". This is just an examples, there are other exceptions too (like "style") 
- _(optional)_ The corresponding name (by default is the same of the property) of the bind object on the JS side.
  > This can be useful if you want to bind under the same name different elements properties
- _(optional)_ An event name that you want to listen to and cause the value to be recomputed
  > If you are binding the "value" property on an "input" tag you might wanna listen to the "input" event in order to update the value on user input

You can binds multiple properties of the same element by separating them with a (or more) spaces like this: `bind="value:color:input innerText className:theme"`.
A more complete example below:
```html
<style>
    pre { display: inline-block; }
    h5 { color: indianred; }
</style>
<html>
    <form id="sign-up-form" action="./sign-up" method="POST">
        <h3>Welcome <span bind="innerText:user"></span>!</h3>
        <input type="text" autocomplete="off" id="kawa" name="kawa" bind="value:user:input" placeholder="Write your name here">
        <input type="text" autocomplete="off" id="bonga" name="bonga" bind="value:password:input" placeholder="Write your password here">
        <h5 hidden>Your password "<code><pre bind="innerText:password"></pre></code>" is too weak!</h5>
        <input type="submit" value="Sign UP" disabled>
    </form>
</html>
<script>
    const weak = useMirror("#sign-up-form h5", 'hidden')
    const submit = useMirror("#sign-up-form input[type=submit]", 'disabled')
    const form = useBinds("#sign-up-form", (val, key) => {
      if (key === 'password') {
        weak.hidden = val === '' || !(submit.disabled = val.length <= 5)
      }
      return val
    })
    form.user = "Guest"
</script>
```
> `app.wed.html`

#### useTemplate
Retrieves a template by it's id and provides methods to facilitate content insertion into the DOM upon initialization.
```html
<style>
    input[readonly] {
        border: none;
        outline: none;
        background: transparent;
        text-decoration: line-through;
    }
</style>
<html type="dynamic">
    <div>
        <input type="checkbox" bind="checked:done:input">
        <input type="text" bind="value:task:input readOnly:done">
    </div>
</html>
<script>
    const { clone: newItem } = useTemplate("todo-item", templ => {
        templ._binds = useBinds(templ)
        return templ
    })
</script>
```
> `todo-item.wed.html`

```html
<html>
    {{ importDynamic "todo-item" }}
    <div id="todo-list">
        <h1>TO DO LIST</h1>
        <div>
            <input type="text" placeholder="Write your task here">
            <input type="button" value=" + " onclick="handleItemAdd()">
        </div>
    </div>
</html>
<script require="todo-item">
    const holder = select("#todo-list")
    const input = useMirror(holder.querySelector("input[type=text]"), 'value')

    const handleItemAdd = () => {
        const item = newItem()
        item._binds.task = input.value
        holder.appendChild(item)
        input.value = ''
    }
</script>
```
> `app.wed.html`

## 4. Building a webpage
With your project set up, you're ready to build your site. Wednesday compiles your components, scripts, and styles into a static, deployable format.
Use the following command to compile your project:
```shell
wed build
```
> This command processes all `.wed.html` components recursively from the specified directory (or current directory if none is given).

The command will output on terminal the relative path to the directory containing the "index.html"
You can now open this file with your browser and everything should be working! No server needed _(well unless you actually do needed it)_

### Organizing components and assets
Arrange your components files however suits your needs. As stated previously the build process is recursive.
> You might for example store your components in a `/components` folder. Or in any other way, Wednesday doesn't really care

For the assets, you can put them wherever you want on the _HomeDir_ specified in your settings file. Again the choice is yours

### Handle directories and settings
If your project is on another directory you can specify the mount directory as a second arguments
> Example: `wed build path/to/my/project`

But what if you want to edit the way your build is generated you can customize it using the JSON settings file:
- **HomeTempl**: Define the file path of the home template (default: `home.tmpl`) 
- **HomeDir**: Define the output directory where the project will be built _and eventually served_ (default: `build`)
- **ScriptDir**: Define the subdirectory (inside _HomeDir_) for the JavaScript files of your components (default: `scripts`)
- **StyleDir**: Define the subdirectory (inside _HomeDir_) for the CSS files of your components. (default: `styles`)

You can also specify the settings file (default is `wed-settings.json`) using the "**settings**" (or "s") flag
> Example: `wed build --settings=path/to/my/settings.json`
> 
> Flag must be prefixed by one or two '-' and value can be included in '"'

### Adding external resources
Inside the settings there is also the "**Styles**" and "**Scripts**" properties. Unlike the previous properties, these two are lists.
You can add there multiple urls or paths for external resources.
> **Be aware**: Those and other properties are used by the home template. You can change it at your own risk 

## 5. Serving a site
As stated previously there is no need for a server. The site should be accessible simply by opening the index.html from the browser.
This is a design choice and one of the main difference between Wednesday and the majority of the frameworks out there.
> **Why not to:**
> - The majority of sites doesn't really needs a back-end for front-end and just end
> - Often the codebase is confusing where you don't know where the code will be executed on server or client or both
> - Modifying elements on screen is the reason why we have JavaScript on browsers. There is no point to add more complexity with http calls to load and re-render components
> - If your project is small you can include a static server for the site on the same server where your other APIs lives
> - If you receive too many requests then you can decide to scorporate the two by simply having another static server 

**Why there is a command:**
> - To have an easy endpoint available on your browser instead of looking for your project folder
> - Provides an easy way to serve statically without relying on external tools
> - To check your changes "live" when you are doing frequent changes, more of that later

Only for these reasons:
```shell
wed serve
```
> The command will build the site and then serve the _HomeDir_ statically

This command accepts the "settings" flags and the optional second argument "mount" like `wed build`, plus:
- "**port**" flag (or "p") to specify the port you want to serve your site on (default: `:8080`)
   > Example: `wed serve --port=":8081"`
- "**live**" flag (or "l") to rebuild the application each specified time interval
   > Example: `wed serve -live=3s`

## 6. Pipeline Integration
Wednesday can also help with CI/CD pipelines when a project grows on size thanks to some settings property

### Declaring Custom Data
You can declare custom data in the "Var" property of your settings file.
This data can be accessed within your components HTML, allowing for dynamic content generation. Example:

```json
{
  // ...
  "Var": {
    "site": {"Name": "My Awesome Site"},
    "version": 1
  }
}
```
> `wed-settings.json`

In your HTML templates, you can use these variables like this:

```html
<html>
    <h1>Welcome to {{ .Var.site.Name }}</h1>
    <p>Version: {{ .Var.version }}</p>
</html>
```
> `welcome.wed.html`


### Using `wed run`
The `wed run` command allows you to automate workflow steps by executing a list of commands specified in the "Run" property of your settings file.
The property value must be an array that lists each command to be executed by `wed run` in order. Example:

```json
{
  // ...
  "Run": [
    "git stash",
    "git checkout master",
    "wed build",
    "zip -r build.zip ./build"
  ]
}
```

To execute these commands sequentially, simply run:
```bash
wed run
```

This setup lets you automate and customize your project’s workflows with ease.
