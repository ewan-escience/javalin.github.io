---
# layout: tutorial
layout: default
title: Clean Vue frontends without the hassle
author: <a href="https://www.linkedin.com/in/davidaase" target="_blank">David Åse</a>
date: 2019-07-29
# permalink: /tutorials/simple-frontends-with-javalin-and-vue
permalink: /drafts/simple-frontends-with-javalin-and-vue
github: https://github.com/tipsy/javalinvue-example
summarytitle: Simple frontends with Javalin and Vue
summary: The tutorial shows how to use the JavalinVue plugin for simplified frontend development
language: kotlin
---

In this tutorial you'll learn how to create simple frontends with Javalin and Vue.
The tutorial is quite extensive, and covers single-file components, routing, error handling,
application layouts, access management (plus authentication), and state sharing between server and client.

The tutorial is a bit different from what I normally write, as it will be pretty opinionated.
The background section gives a bit of insight as to how I've come up with the approach
used in the tutorial, but if you're just here to learn how to write simple frontends,
you can skip ahead to [setup](#setup) section.

## Background

I've been doing web development for roughly ten years. For server side rendering,
I started with pure HTML (framesets!), then PHP, JSP, JSTL, Twirl (Scala) and finally, Velocity templates (Java).
On the frontend I've used jQuery/Zepto, then Knockout, Angular, Meteor, and finally, Vue.
For styling I've used Less, SASS (SCSS), and finally CSS variables.
For building I've used Bower, NPM and Yarn, Grunt and Gulp, and I've configure a total of one Webpack project.

Frontend has come a long way in the last ten years, and declarative/reactive view libraries
are much nicer to work with than the DOM manipulation mess of the past.

But modern frontend architecture is also often hugely (and unnecessarily) complex.
I won't go into a rant about this, as I won't move any people in either camp, but this
tutorial will show an alternative approach for creating clean and modern frontends in pure Vue,
without most of the complexity that would normally follow along with it.
There will be no Node, NPM, or Webpack, no state management, reducers, or sagas. No rehydration, no tree shaking,
no code splitting, or even transitive dependencies. No boilerplate or plumbing of any kind.

Just business logic.

## Setup
Our backend will be Kotlin, and we'll be using Maven to build.
We need to bring in Javalin (web library), Jackson (JSON serializer), and slf4j-simple (logger).\\
We'll also add Vue (view library) and Axios (http client) for our frontend:

```markup
<dependency>
    <groupId>io.javalin</groupId>
    <artifactId>javalin</artifactId>
    <version>3.3.0</version>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.module</groupId>
    <artifactId>jackson-module-kotlin</artifactId>
    <version>2.9.9</version>
</dependency>
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-simple</artifactId>
    <version>1.7.26</version>
</dependency>
<dependency>
    <groupId>org.webjars.npm</groupId>
    <artifactId>vue</artifactId>
    <version>2.6.10</version>
</dependency>
<dependency>
    <groupId>org.webjars.npm</groupId>
    <artifactId>axios</artifactId>
    <version>0.19.0</version>
</dependency>
```

The frontend dependencies are contained in [Webjars](https://www.webjars.org/).
Webjars can be built directly from NPM, so if something is available on NPM, it's also available as a Webjar.
To view the rest of the POM, please go to [GitHub](URL).

Now that we have all our dependencies in order, we need to configure our web server.\\
Let's create `/src/main/kotlin/javalinvue/Main.kt:`:

```kotlin
import io.javalin.Javalin

fun main() {
    Javalin.create { config ->
        config.enableWebjars()
    }.start(7000)
}
```

We also need an HTML file to load our frontend dependencies and to initialize Vue. \\
Let's create `/src/main/resources/vue/layout.html`:

```markup
<html>
    <head>
        <meta charset="utf8">
        <script src="/webjars/axios/0.19.0/dist/axios.min.js"></script>
        <script src="/webjars/vue/2.6.10/dist/vue.min.js"></script>
        @componentRegistration
    </head>
    <body>
        <main id="main-vue" v-cloak>
            @routeComponent
        </main>
        <script>
            new Vue({el: "#main-vue"});
        </script>
    </body>
</html>
```

There are two Javalin specific things here: `@componentRegistration` and `@routeComponent`.
Javalin's Vue plugin will scan your `/resources/vue` folder and put all your Vue components
into `@componentRegistration`, similar to how libraries are loaded via `<script>` tags.
Javalin will also let you choose one component to mount based on the current URL,
this is the `@routeComponent`.

## Hello World

Now that we have a layout file, let's create `/resources/vue/views/hello-world.vue`:

```html
<template id="hello-world">
    <h1 class="hello-world">Hello, World!</h1>
</template>
<script>
    Vue.component("hello-world", {template: "#hello-world"});
</script>
<style>
    .hello-world {
        color: goldenrod;
    }
</style>
```

We're telling Vue that we want to register a `hello-world` component, and to use the template with the id `hello-world`.
We're also giving the `Hello, World!` message a nice goldenrod color. Notice how we can put HTML, JavaScript and CSS all in the same file.

To display our component to the user, we need to tell Javalin when to show it. Let's expand our web server with a new route:

```kotlin
import io.javalin.Javalin
import io.javalin.plugin.rendering.vue.VueComponent

fun main() {

    val app = Javalin.create { config ->
        config.enableWebjars()
    }.start(7000)

    app.get("/", VueComponent("<hello-world></hello-world>"))
}

```

The `@routeComponent` that we added in `layout.html` earlier will be replaced by the String inside of `VueComponent`.
This means a call to `/` will load the layout and display our `<hello-world></hello-world>` component.

Restart the server, go to `http://localhost:7000/`, and you'll see "Hello, World!" in a nice goldenrod color.

## Routing and error handling

Now that we know how to create components, let's look at a very common scenario:
creating an admin interface. Our admin interface should be able to display an overview
of users, and also additional details for one specific user.
This will required two views, but we should probably also include a 404 page.

Let's change the server by adding the following lines:

```kotlin
app.get("/users", VueComponent("<user-overview></user-overview>"))
app.get("/users/:user-id", VueComponent("<user-profile></user-profile>"))
app.error(404, "html", VueComponent("<not-found></not-found>"))

app.get("/api/users", UserController::getAll)
app.get("/api/users/:user-id", UserController::getOne)
```

We've referenced `UserController` in the previous snippet, but that doesn't exist yet.\\
So, let's create `/src/main/kotlin/javalinvue/UserController.kt`:

```kotlin
import io.javalin.http.Context
import io.javalin.http.NotFoundResponse

data class User(val id: String, val name: String, val email: String, val userDetails: UserDetails?)
data class UserDetails(val dateOfBirth: String, val salary: String)

val users = setOf<User>(
    User(id = "1", name = "John", email = "john@fake.co", userDetails = UserDetails("21.02.1964", "2773 JB")),
    User(id = "2", name = "Mary", email = "mary@fake.co", userDetails = UserDetails("12.05.1994", "1222 JB")),
    User(id = "3", name = "Dave", email = "dave@fake.co", userDetails = UserDetails("01.05.1984", "1833 JB")),
    User(id = "4", name = "Jane", email = "jane@fake.co", userDetails = UserDetails("30.12.1989", "1532 JB")),
    User(id = "5", name = "Eric", email = "eric@fake.co", userDetails = UserDetails("14.09.1973", "2131 JB")),
    User(id = "6", name = "Gina", email = "gina@fake.co", userDetails = UserDetails("16.08.1977", "1982 JB")),
    User(id = "7", name = "Ryan", email = "ryan@fake.co", userDetails = UserDetails("07.11.1988", "1638 JB")),
    User(id = "8", name = "Judy", email = "judy@fake.co", userDetails = UserDetails("05.01.1959", "2983 JB"))
)

object UserController {

    fun getAll(ctx: Context) {
        ctx.json(users.map { it.copy(userDetails = null) }) // remove sensitive information
    }

    fun getOne(ctx: Context) {
        val user = users.find { it.id == ctx.pathParam("user-id") } ?: throw NotFoundResponse()
        ctx.json(user)
    }

}
```

This completes our backend, let's move on to the frontend. We want three views (user-overview, user-profile, and not-found), so
let's create three separate files in `/src/main/resources/vue/views`. We'll start with the user-overview:

{% raw %}```html
<template id="user-overview">
    <div>
        <ul class="user-overview-list">
            <li v-for="user in users">
                <a :href="`/users/${user.id}`">{{user.name}} ({{user.email}})</a>
            </li>
        </ul>
    </div>
</template>
<script>
    Vue.component("user-overview", {
        template: "#user-overview",
        data: () => ({
            users: [],
        }),
        created() {
            axios.get("/api/users")
                .then(res => this.users = res.data)
                .catch(() => alert("Error while fetching users"))
        }
    });
</script>
<style>
    ul.user-overview-list {
        padding: 0;
        list-style: none;
    }
    ul.user-overview-list a {
        display: block;
        padding: 16px;
        border-bottom: 1px solid #ddd;
    }
    ul.user-overview-list a:hover {
        background: #00000010;
    }
</style>
```{% endraw %}

It's a simple component which performs one GET request to the server to fetch the list of users,
then sets the component state. Vue loops through the users and creates a list of links that we can
click to view additional information for one user. We've also included a few CSS rules to pretty things up.

Open `http://localhost:7000/users/` to view the list of users. If you click on one, a blank page will show.\\
Let's fix this by creating the user-profile component:

{% raw %}```html
<template id="user-profile">
    <div>
        <dl v-if="user">
            <dt>User ID</dt>
            <dd>{{user.id}}</dd>
            <dt>Name</dt>
            <dd>{{user.name}}</dd>
            <dt>Email</dt>
            <dd>{{user.email}}</dd>
            <dt>Birthday</dt>
            <dd>{{user.userDetails.dateOfBirth}}</dd>
            <dt>Salary</dt>
            <dd>{{user.userDetails.salary}}</dd>
        </ul>
    </div>
</template>
<script>
    Vue.component("user-profile", {
        template: "#user-profile",
        data: () => ({
            user: null,
        }),
        created() {
            const userId = this.$javalin.pathParams["user-id"];
            axios.get(`/api/users/${userId}`)
                .then(res => this.user = res.data)
                .catch(() => alert("Error while fetching user"))
        }
    });
</script>
```{% endraw %}

This is pretty similar to our user-overview, but since this is a dynamic route,
we have to ask our router what the current user-id is.
JavalinVue includes path parameters and query parameters on `$javalin` by default,
and also has an optional state parameter we will look at later.\\
For now, let's finish up our views with a 404 page:

{% raw %}```html
<template id="not-found">
    <h1>Page not found (error 404)</h1>
</template>
<script>
    Vue.component("not-found", {template: "#not-found"});
</script>
```{% endraw %}

Great, we have all our views ready! ..but they're don't look very consistent.
While not strictly related to JavalinVue, let's add an application frame:

{% raw %}```html
<template id="app-frame">
    <div class="app-frame">
        <header>
            <span>JavalinVue demo app</span>
        </header>
        <slot></slot>
    </div>
</template>
<script>
    Vue.component("app-frame", {template: "#app-frame"});
</script>
<style>
    .app-frame > header {
        padding: 20px;
        background: #b6e2ff;
        font-size: 20px;
        display: flex;
        justify-content: space-between;
        align-items: center;
    }
</style>
```{% endraw %}

The `<slot></slot>` element will be replaced by the content of the current page. Here is how the 404 page can use the new app-frame:

{% raw %}```html
<template id="not-found">
    <app-frame>
        <h1>Page not found (error 404)</h1>
    </app-frame>
</template>
<script>
    Vue.component("not-found", {template: "#not-found"});
</script>
```{% endraw %}

Now that both the frontend and backend are done, it's time to make things more complicated by adding access management to the mix.

## Access Management
Access management in Javalin is handled by the aptly named `AccessManager`. This is a functional interface which takes a HTTP context,
a handler function, and a set of roles. It's up to the developer to determine if a request is valid. We will be securing our application using
basic-auth for simplicity, but you can use any technique and identity provider you want with the `AccessManager` interface.

First of all we need to define roles. Some parts of the app should be accessible to everyone (the user-overview and the 404 page), while
other parts should only be available if you log in. Two roles should be enough: `ANYONE` and `LOGGED_IN`. We add those roles to the endpoints
(both for the views and the APIs, and create an `AccessManager`:

```kotlin
import io.javalin.Javalin
import io.javalin.core.security.Role
import io.javalin.core.security.SecurityUtil.roles
import io.javalin.core.util.Header
import io.javalin.http.Context
import io.javalin.plugin.rendering.vue.JavalinVue
import io.javalin.plugin.rendering.vue.VueComponent

enum class AppRole : Role { ANYONE, LOGGED_IN }

fun main() {

    val app = Javalin.create { config ->
        config.enableWebjars()
        config.accessManager { handler, ctx, permittedRoles ->
            when {
                AppRole.ANYONE in permittedRoles -> handler.handle(ctx)
                AppRole.LOGGED_IN in permittedRoles && anyUsernameProvided(ctx) -> handler.handle(ctx)
                else -> ctx.status(401).header(Header.WWW_AUTHENTICATE, "Basic")
            }
        }
    }.start(7000)

    app.get("/", VueComponent("<hello-world></hello-world>"), roles(AppRole.ANYONE))
    app.get("/users", VueComponent("<user-overview></user-overview>"), roles(AppRole.ANYONE))
    app.get("/users/:user-id", VueComponent("<user-profile></user-profile>"), roles(AppRole.LOGGED_IN))
    app.error(404, "html", VueComponent("<not-found></not-found>"))

    app.get("/api/users", UserController::getAll, roles(AppRole.ANYONE))
    app.get("/api/users/:user-id", UserController::getOne, roles(AppRole.LOGGED_IN))

}

fun anyUsernameProvided(ctx: Context) = ctx.basicAuthCredentials()?.username?.isNotBlank() == true
```

This is just an example, our authentication isn't exactly secure. As long as the user enters
**anything** in the basic-auth username field, we log the user in. We completely ignore the password.

## Server Side State
Now that we can log users in, it would be nice if the client knew the current user.
Our server knows, so we need to transfers this knowledge somehow.
This can be solved by setting the JavalinVue state:

```kotlin
JavalinVue.stateFunction = { ctx -> mapOf("currentUser" to ctx.basicAuthCredentials()?.username) }
```

This line of code sets a state-function that will run for every `VueComponent`, so all components will now
have access to the current user (if there is one). Since basic-auth works per directory, the frame will only
show the user for `http://localhost:7000/users/` (the user-overview) and its subpaths (the individual profiles).
Let's add it to the app-frame:

{% raw %}```html
<template id="app-frame">
    <div class="app-frame">
        <header>
            <span>JavalinVue demo app</span>
            <span v-if="$javalin.state.currentUser">Current user: '{{$javalin.state.currentUser}}'</span>
        </header>
        <slot></slot>
    </div>
</template>
```{% endraw %}

## Conclusion
We've created a fully working (but pretty limited) admin interfaces with only a few files.
* `Main.kt` has the server config (routes, error handlers, access management)
* `UserController.kt` has the list of fake users, and methods to get them (getAll, getOne)
* `layout.html` has the frontend dependencies and initializes Vue
* `app-frame.vue` has a header and some global styling which is included in each other component
* `user-overview.vue` has a list of users
* `user-profile.vue` has additional details for one user (requires login)
* `not-found.vue` has a 404 error page
* `pom.xml` has all our dependencies

Since our frontend dependencies are prepacked WebJars, we don't need NPM, and we don't need to
check in any frontend libraries manually. The project structure is very clean, and everything is where
it's supposed to be:

<div class="compressed-code" markdown="1">
```text
javalinvue-example
├───src
│   └─── main
│       └───kotlin
│           ├───javalinvue
│           │   ├───UserController.kt
│           │   └───Main.kt
│           └───resources
│               ├───components
│               │   └───app-frame.vue
│               ├───components
│               │   ├───not-found.vue
│               │   ├───user-overview.vue
│               │   └───user-profile.vue
│               └───layout.html
└───pom.xml
```
<style>.compressed-code .highlighter-rouge pre code { line-height: 1.2; }</style>
</div>

That's about it. The Epilogue section contains a bit more discussion about this technique,
but similarly to the Background section you can skip it if you're not interested.

### Epilogue
The architecture described in this tutorial has a lot of the benefits of a modern frontend.
We have full client side rendering, no DOM manipulation, and we're able to use Vue fully. We even have single-file components.
We don't have any client side routing, we actually have one one app per page (per server side route),
which makes state management a lot easier. Each view is responsible for its own state, and that's it.
Shared state (signed-in-user, current-theme, etc) can be set on the session (server side), and is included in all views.

We're missing out on a few nice things. We can't hot reload the content of a component or inject new styles, we actually
have to refresh the page to see changes. Changes are picked up instantly though, so a refresh typically takes 5ms.

### Pros and Cons

### When to use this approach