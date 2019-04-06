# Curie
A modular Node.js http/1.0 framework

## <div id="config">Config</div>
Curie-server accepts a number of configuration options of the type:
```typescript
interface ServerParams {
  routes: string,
  routeParser: ClassConstructor<RouteParser>
  public: string
  port: number
  listeners: [string, string | RegExp]
  middleware: [string, string | RegExp]
  database: string,
  root: string
}
```
**Keep in mind that all the paths are relative to the main file**
But if you don't create a config file, `InitApp({...})` will either take the config from the default options or it's parameters. 

## <div id="file_structure">File structure</div>
curie-server supports a multi file structure. An example file structure:
```
|public/
|--css/
|----main.css
|--js/
|----main.js
|routes/
|--index.pug
|listeners/
|--index.list.ts
|middleware/
|--logger.mdw.ts
|index.ts
|database.ts
```

### <div id="main_file">Main file</div>
The main file is the file, which is responsible for starting your application.
```typescript
import { initApp } from "curie-server/dist/@core"
initApp({
  port: 8000,
  public: "../public",
  routes: "../routes",
  listeners: ["./listeners", "list.[jt]s"],
  middleware: ["./middleware", "mdw.[jt]s"],
  database: ""
})
```

### Database file
The database file is responcible for connecting with your database. It should export a class which extends the [DBridge](#dbridge) class. You may create your own class or use the so-called out of the box PotgreSQL DBridge.
It should look something like it:
```typescript
import { PostDBridge } from "curie-server"
import { database } from "curie-server/dist/@core"

@database("postgres://postgres:postgres@127.0.0.1:5432/postgres")
export default class extends PostDBridge {}
```

### Listeners
Listeners are responsible for responding to incoming http requests. Both their location and extension are specified in the `Server` constructor parameters in the [main](#main_file) file. A listener should extend [the Listener class](#listener), implement `onGET` and/or `onPOST` method(s), such that they return [CallbackReturnType](#callback_return_type). Example:
```typescript
import c, { Listener } from "curie-server";
import { hookup } from "curie-server/dist/@core"

@hookup("/")
export default class extends Listener {
  async onGET(req: c.Request, res: c.Response): c.CallbackReturnType {
    this.server.routeParser.render(res, "index")
    return [null, false]
  }
}
``` 

### Middleware
Middleware is responcible for interrupting incoming requests and can even reject them. Middleware should extends [the Middleware](#middleware) class and return [the CallbackReturnType](#callback_return_type). It should look something like this:
```typescript
import { CallbackReturnType, Middleware, withTime, Request, Response, c_log } from "curie-server";
import { use } from "curie-server/dist/@core";

@use()
export default class extends Middleware {
  async intercept(req: Request, res: Response) {
    c_log(withTime(`[LOGGER]> ${req.method}: ${req.url || ""}`))
    return [null, true] as CallbackReturnType
  }
}
```

### <div id="route_parser">RouteParser</div>
The `RouteParser` is responsible for parsing and rendering template files. If you want to use the lamplating language of your choice, you should provide a class that extends the `RouteParser`.
```typescript
abstract class RouteParser<RouteType = any> {
  path: string
  routes: LooseObject<RouteType>
  server: Server
  public constructor(path: string, server: Server) {/*...*/}
  abstract compile(route: string): CallbackReturnType
  abstract async compileAll(): Promise<CallbackReturnType>
  abstract async render(res: Response, route: string, locals?: LooseObject): Promise<CallbackReturnType>
}
```
Out of the box curie-server is providing support for the [pug](https://pugjs.org/api/getting-started.html) templating lang.

### Routes
Routes are themplates rendered by [the RouteParser](#route_parser). Out of the box you get the `PugParser`, which compiles `.pug` files and allows you to query item from the database (template: `//# $<variable_name>: <query>`).
```pug
//# $posts: SELECT * FROM posts

<!DOCTYPE html>
html(lang="en")
  head
  body
    ul.posts
      for post in posts
        li.posts__post
          h2.posts__post__title= post.title
          p.posts__post__body= post.body
```

### Single file approach
While I highly advise you to take the advatnage of the multi file structure, around which the `curie-server` was built, you can fit everything into a single file.
```typescript
import { initApp, database, hookup, use } from "curie-server/dist/@core";
import c, { Server, PostDBridge, Listener, Middleware, c_log, withTime } from "curie-server";

(async () => {
  await initApp({
   port: 8000,
   public: "../public",
   routes: "../routes",
   listeners: ["./listeners", "list.[jt]s"],
   middleware: ["./middleware", "mdw.[jt]s"],
   database: ""
  })

  @database("postgres://postgres:postgres@127.0.0.1:5432/postgres")
  class Db extends PostDBridge {}

  @hookup("/")
  class IndexListener extends Listener {
    async onGET(req: c.Request, res: c.Response) {
      await this.render(res, "index")
      return [null, false] as c.CallbackReturnType
    }
  }
  @use()
  class Logger extends Middleware {
    async intercept(req: Request, res: Response) {
      c_log(withTime(`[LOGGER]> ${req.method}: ${req.url || ""}`))
      return [null, true] as c.CallbackReturnType
    }
  }
})()
```

## Classes
### <div id="dbridge">DBridge</div>
```typescript
@database("<connection_uri>")
class MyDBridge extends DBridge<DBInterface, QueryType> {
  async get(query: QueryType): Promise<LooseObject[] | any> {
    // Fetch the query! Good boy!
    return getResponse
  }
  async update(query: QueryType): Promise<UpdateResponse | any> {
    // Update something...
    return updateResponse
  }
  async delete(query: QueryType): Promise<DeleteResponse | any> {
    // Delete something...
    return deletionResponse  
  }
  async create<T extends ClassConstructor>(model: T, data: ConstructorParameters<T>): Promise<CreateResponse | any> {
    // Create something...
    return creationResponse
  }
}
```
### <div id="listener">Listener</div>
```typescript
@hookup("/")
class IndexListener extends Listener {  
  async onGET(
    req: http.IncomingMessage, 
    res: http.ServerResponse
    ): Promise<CallbackReturnType | undefined> {
      return [null, true]
    }
  async onPOST(
    req: http.IncomingMessage, 
    res: http.ServerResponse
    ): Promise<CallbackReturnType | undefined> {
      return [null, true]
    }
}
```
### <div id="middleware">Middleware</div>
```typescript
@use()
export default class Intercepter extends Middleware {
  async intercept(req: Request, res: Response) {
    // Do something
    return [null, true] as CallbackReturnType
  }
}
```

## Interfaces and types
### Request
```typescript
interface Request extends http.IncomingMessage {
  query: LooseObject<string>
  cookies: cookies
  body: LooseObject
}
```
### Response
```typescript
interface Response extends http.ServerResponse {
  cookies: cookies
}
```

### ServerParams
It is a configuration object passed to the Server constructor. 
```typescript
interface ServerParams {
  routes: string,
  routeParser: ClassConstructor<RouteParser>
  public: string
  port: number
  listeners: [string, string | RegExp]
  middleware: [string, string | RegExp]
  database: string,
  root: string
  [key: string]: any | any[]
}

Server.DEFAULT_SERVER_OPTIONS: ServerParams = {
    routes: "./routes",
    routeParser: PugParser,
    public: "./public",
    port: 8000,
    listeners: ["./", "list.[jt]s"],
    middleware: ["./", "mdw.[jt]s"],
    database: '',
    root: path.dirname((require.main as NodeModule).filename)
  }
```

### <div id="callback_return_type">CallbackReturnType</div>
CallbackReturnType is a value returned by many `curie-server` functions, but is used especially in [the Listener](#listener) and [the Middleware](#middleware) classes. The first part of the tuple is the `Error`, and the 2nd one is the `ShouldContinue` boolean, which tells the inner loop whether is should send the Response to the client or continue.
```typescript
type CallbackReturnType = [Error | null, boolean]
```