# The Ultimate MCP Crash Course - Build From Scratch

[![Watch the video](https://img.youtube.com/vi/ZoZxQwp1PiM/hqdefault.jpg)](https://youtu.be/ZoZxQwp1PiM)

**MCP** is incredibly popular right now. Everyone's talking about it, but no one's going in-depth enough on how it works and what it means for you. So, in this video, I'm not only going to explain everything you need to know about **MCP** and how it works, but I'm also going to be showing you how to create your very own **MCP** server that you can hook up to any **MCP** client that you want.

And then on top of that, I'm even going to show you how to create your very own **MCP** client that can hook up to any server out there, whether you created it or someone else created it. This is going to be the complete crash course on everything you need to know about **MCP**.

## Sponsored Segment: Frontend Masters

If you're struggling to learn web development, it may be because the content you're using doesn't resonate with your learning style. Some people really learn best from workshop style events, and that's where our sponsor, Frontend Masters, specializes. They have hundreds of different courses and learning paths that you can check out that follow through with workshop based content. So you get that more collaborative kind of experience you get in a workshop, but for way less cost than an actual workshop.

They also have a brand new feature which are these tutorials. And these are kind of like a curated version of YouTube that has exclusive videos. I actually have some exclusive videos out there right now and I'm continually making more exclusive videos for them. And I even have a workshop that I'm going to be doing which is going to go onto one of those courses. So, if you're interested in checking out some of this additional content that I have on front-end masters or really diving more into this workshop based way of doing things, I'll have a link down in the description for you to use. That'll give you a discount when you check out.

## What is MCP?

Welcome back to WebDev Simplified. My name is Kyle and my job is to simplify the web for you so you can start building your dream project sooner. And I know you're dying to get into the code and see how we create something like this. But we first need to understand what **MCP** is.

There's a really nice site for **MCP**. I'll link it down in the description below. It goes over a lot of really good information on what **MCP** is, how it works, and everything like that. So, I highly recommend checking this out for more in-depth knowledge.

> But really, **MCP** is just a protocol. It stands for **model context protocol**. And all it is is a protocol similar to like a **REST API** or a **GraphQL API**.

And if we look here, it just allows you to communicate between an **MCP** server and an **MCP** client. Essentially, it tells you how the client and server should communicate with each other, so they both know how to send messages back and forth. And one client can hook up to as many **MCP** servers as it wants to. And again, like I said, it's very similar to **REST APIs** or **GraphQL**. Essentially, it's just a language that you can use on your client to communicate with a server and that the server can use to send information back to the client so they know how to work with each other properly to do all these really cool different things.

### Core Components of an MCP Server

Now, inside of an **MCP** server, there are essentially four main things that it's made up of, but two of them are the most important. That is going to be:

*   **Tools**
*   **Resources**
*   **Prompts**
*   **Samplings**

Now, of those four, **tools** and **resources** are by far the most used with **tools** probably being the most used hands down out of all four of them. And really, the entirety of your server is made up of these four things. And the **MCP**, the protocol itself, is what allows you to communicate between all these different things the server has.

#### Tools

Now, the first of these that I want to talk about is going to be **tools**. And **tools** are essentially a way for a client to call code on the server. So, essentially, let's say that I have an **MCP** server for Excel. I can have a tool that says create an Excel document. And then me as a user using some type of AI chat program, I could tell my AI chat program, hey, create me an Excel sheet that has this information inside of it. That AI chat program knows there's an **MCP** server for Excel that has a tool for creating an Excel sheet. So essentially, the AI will call that tool to create a new Excel sheet.

And these **tools** can be as simple or as complex as you want. It could be as simple as creating an Excel sheet or it could be as complicated as doing a bunch of different data computation in an Excel sheet to create charts and so on. Essentially, it's a way for these AI clients to interact with different applications and user interfaces and programs through a specific model context protocol. So, **tools** are just a way for the AI to essentially call functions and do things inside of whatever program this **MCP** wraps around.

#### Resources

A **resource** is quite a bit simpler and really all it is is a set of data. Now, this could be like a database. It could be files on your file system. It can literally be anything that contains data. Images, files, database, doesn't matter what it is. This is just something that you have access to on a particular **MCP** server. So, in the case of Excel, you may have an **MCP** server and the **resources** could be the rows inside of your Excel tables. It could be the Excel files themselves or it could even be like charts inside of your Excel files. There's lots of different things that could be **resources** depending on what your application is. A lot of times if you're doing like a application that's a web application, your **MCP** **resources** are probably going to be things related to database records or maybe files that users have uploaded.

#### Prompts and Samplings

Now **prompts** and **samplings** are quite a bit less used, but they're still important. A **prompt** is essentially a pre-created prompt that you can ask your **MCP** server for and it'll send down a really well-formatted prompt for you to do specific tasks. So this is, like I said, not as useful because you can already have **tools** that do some of this for you or **resources** that give you the information. But if you specifically want to create a prompt that helps a user with something, you can create a prompt on the server, they can ask for that prompt and it'll send them back that prompt so that they can use that inside of their AI client. It's just a really great way to create robust prompts on your server that then the client or user can interact with.

Finally, **samplings** is kind of the opposite of all the stuff we've talked about where normally the AI client is talking to the server and saying, "Hey, do this thing for me. Give me this information. Give me this prompt." Whatever it is. But **samplings** are the opposite. That is when the server wants to get information from an AI and it says, "Hey, I need some additional information for you. Can you run this prompt on your AI and then send the result back to me?" So, a **sampling** is just a way for a server to ask the client for some type of information based on a prompt that they send down to an AI. So, it's just the reverse of everything else that we've talked about.

## Building an MCP Server

And that's pretty much all there is to this model context protocol when it comes to the server client communication. The next thing that we want to do is actually dive into the code and start implementing this ourselves. And the really nice thing about this is we don't have to start entirely from scratch because this same site, this model context protocol site that has all the detailed information on this. They also have SDKs for pretty much every single language that you could want. So in our case, they have a TypeScript SDK that we can open up. And if we scroll all the way down here quite a ways, you'll eventually find the installation step and we can just copy that into our very own project.

So now that we have that copied over, we can go to our actual project. And you'll notice on the side that I have **Copilot** currently open. We're not going to be using that right away, but when we get our **MCP** server set up, I want to be able to show you how you can integrate it with **Copilot** and how you can actually call commands directly from here. And this is where you can integrate it with any application you want. So, if you're using Cloud, you can integrate with Cloud. All the integrations work almost exactly the same.

So, let's go ahead and we're going to install that library. And all it is is a very thin wrapper over the protocol that is created from this model context protocol. But we still have to write a lot of the main code ourselves.

### Initial Project Setup

Now, you'll also notice I have a very small amount of starting code here. I just have a very basic boilerplate `tsconfig` file. I pretty much copied this directly from the model context protocol site, but really all it is is just a simple file for a Node.js TypeScript project.

```json
// tsconfig.json
{
  "compilerOptions": {
    // ...
  }
}
```

Then in my `package.json`, I have a few dev dependencies. I can see I have the types for Node. I have `tsx`, which allows us to run this dev command to restart our server automatically when files change. And then I have TypeScript, obviously, because we're in a TypeScript project. If you were using JavaScript, you wouldn't need any of these dev dependencies or this TS config. It's only because I'm using TypeScript that we have these.

And I have three commands. The `build` command converts my TypeScript to JavaScript. The `build:watch` command just creates those files every single time something changes. So every time I update my files, it rebuilds it. And then the `dev` command allows me to just really quickly work through my process and automatically refresh every time I make changes. Overall, relatively straightforward.

```json
// package.json
{
  "scripts": {
    "build": "tsc",
    "build:watch": "tsc -w",
    "dev": "tsx watch src/server.ts"
  },
  "devDependencies": {
    "@types/node": "^20.12.12",
    "tsx": "^4.9.3",
    "typescript": "^5.4.5"
  }
}
```

### Creating the Basic Server

Now, in order to set up our application inside of our server, we want to create a new **MCP** server.

```typescript
// src/server.ts
import { MCPServer } from "@mcp/sdk/server";

const server = new MCPServer({
    name: "test",
    version: "1.0.0",
    capabilities: {
        resources: {},
        tools: {},
        prompts: {}
    }
});
```

So we'll say `const server is equal to new MCPServer`. Just like that. And let me make sure I get those parenthesis or those brackets in the correct place. Make sure I spell this correctly. And then we need to make sure that we import this. So we're going to say that we're going to import `MCPServer`. That's coming from that `@mcp/sdk`. Let me make sure everything is spelled properly here. And we specifically want to make sure we get this from the server file and we want to get it from the MCP file. Just like that.

So now we have that **MCP** server and we can set up all the files and names and stuff inside here. So really all we need to do is give it a name. I'm just going to call this `test`. You can call it whatever you want. So, if you're creating an Excel-based **MCP** server, you could call it Excel. It doesn't really matter what the name is. And then we need to give it a version. In our case, we'll just call this version `1.0`.

Now, we also have a section down below here that is called `capabilities`. And this just essentially says what is our server capable? What are the different things that we can do? Well, in our case, we have **resources**, we have **tools**, and we have **prompts**. Those are going to be the main things that our server is capable of, which is essentially all three of the things that a server can do because **samplings** are things you send from the server to the client. So we don't have to specify that as a capability because we need the client to be able to support that not ourselves. So we support **resources**, **tools**, and **prompts**. So to be able to set up that capability, all we do is we just pass it an empty object. And that says, hey, we support this particular thing. And we want to do this for **tools** as well as for **prompts**. And we're just saying we support all three of these because in this project, we're going to create a server that does all three of these things. So you can see the differences between them and how they all interact.

### Setting Up the Transport Layer

Now, the next thing we need to do is to run our server. And we'll just create a function called `main`. Just like that. And then we'll call this `main` function down here. And the only reason I'm doing that is because this is going to be an asynchronous function. So I can put `async await` inside of here.

Now to be able to use an **MCP** server and client relationship, you need to specify what the actual transport protocol is going to be. And there are technically three, but really only two main protocols.

1.  The first one is going to be **standard input output**. Essentially, that's just going to be using the standard IO, like when you do `console.log`. Essentially, it's that type of communication through a terminal. This is great when you have an application running locally on the same place that your actual client is running. So, in our case, we have **Copilot** running on my computer and we have our program also running on my computer. So, we can use a standard IO process for this.
2.  The other option is going to be **HTTP streaming**. This is great if you have like a web application where you're streaming this down to another web application that's not connected on the same network. That would be a use case for a streaming version.
3.  And then technically there's another one called server sent events but that's deprecated so we don't really need to worry about it. It was replaced by HTTP streaming.

So in our case we need to set up what our transport is going to be. For the most part all the code stays exactly the same no matter which transport you use. It just depends on how your application runs. Is it local or is it going to be remote? In our case we have a local application. So we're going to be using the standard IO.

```typescript
// src/server.ts
import { StandardIOServerTransport } from "@mcp/sdk/server/stdio";

async function main() {
    const transport = new StandardIOServerTransport();
    await server.connect(transport);
}

main();
```

So we can say `StandardIOServerTransport` just like this. And then I just need to make sure that I import this. So if I just scroll to the top, I'll paste in the code for importing that. You can see it comes from the `@mcp/sdk/server/stdio` library. And then we can just say that we want to connect our server. So we can say `await server.connect` and we can connect on that transport just like that. And now we've essentially created an **MCP** server. We hooked up to a transport layer. It just doesn't do anything yet. But we do have an **MCP** server.

### Inspecting the Server

Now before we go about implementing any of the complex stuff inside of our **MCP** server, I want to make sure what we've written actually works. So what we're going to do is we're going to download a tool coming from that same model context protocol team that allows us to essentially inspect all the stuff inside of our server. It's like Postman but for **MCP**.

So what we want to do is we want to `npm i`. This is going to be a dev dependency and it's `@mcp/inspector`. So we're just going to install that library real quick. And then what we want to do is we want to create a really simple `package.json` script to be able to run that inspector for us. So we can go into our `package.json`. And what I want to do is I want to have a `server:inspect`.

```json
// package.json
"scripts": {
    // ...
    "server:inspect": "DANGEROUSLY_OMIT_AUTH=true @mcp/inspector --command \"npm run dev\""
}
```

And what this command is going to do is going to run that model context protocol library that we just downloaded. So we can say `@mcp/inspector`. That's going to run the library. And what we need to do is pass it a command that we can run to actually run our server. So our server command is called `dev`. So we'll pass along the command `npm run dev`. And what that essentially does is it tells our inspector that our server is using this command to run. So when we start up our inspector, it'll also start up our server at the exact same time.

Now, one other thing I'm going to do that will make this a little bit easier for testing purposes is I'm going to set an environment variable called `DANGEROUSLY_OMIT_AUTH`. I'm going to set that to `true`. Just like that. So essentially, I'm setting this environment variable before I run my command. This is just going to make it easier for us to test because normally whenever we would make changes, we would need to restart our inspector, which gives us a brand new auth token. So, we would need to manually reopen our website at that brand new auth token every single time. But by making sure we set this to be true, we at least get rid of that auth token. So, when we're doing our testing, we don't have to worry about restarting it every single time. So, this is just a little bit easier for us to test with.

So, now let's see if that works. We'll say `npm run server:inspect`. Just like that. You should see that it's going to start up. It's going to give us a little warning because we're essentially omitting that auth token, but it will give us a URL that when we open up is actually going to give us our thing. So, if we open this up right here and I drastically zoom this back out to a more reasonable size, you can see that essentially we have here our transport type, which in our case is standard IO. We have the command `npm run server:dev`. That's what we typed in. And now we have the ability to connect if we want to. So, when we click on connect, you'll see it looks like it's not quite working. And there we go. It just took a little second. So, now if we connect, you can see we are connected. And I can click this ping server button and it ran the ping command, sent it to our server and it gave us back an empty response which is exactly what you're supposed to get when you do a ping command. So essentially we have a server. It is running. We're able to connect. We just don't have any **tools**, **prompts** or **resources** yet. We just have the ability to ping it and get back a response for that.

### Implementing a Tool

So let's get started by creating our very first tool. So we can come into here into our server. And to create a tool, it's super simple. We can just say `server.tool`. And then what we need to do is we need to pass along information for what our tool is going to look like. Now, there's a lot of different ways you can create a tool, but every tool needs to have a name. And we're going to call this one `createUser`. This is essentially the name that the AI is going to see. And then we can also use a human readable name as well as a description for the AI as well.

So, the next thing we're going to pass along is a description. And this is what the AI can use to determine what this function does. So, we'll just say `create a new user in the database`. There we go.

And then finally, we can pass along what all the different parameters we're passing in. So in our case to create a user we need to pass along things such as the name, email and so on. So we'll come in here. We'll say that the name is going to be a **Zod** string and we're going to be using **Zod** for this. Now in the particular case of this library you can use JSON schema or **Zod** but I find **Zod** much easier and better to work with. So we're going to use **Zod**. So we just need to make sure that we install **Zod**. So let's come over here, open a brand new terminal, `npm i zod`. That'll install that **Zod** library for us. Put it directly inside of here so we can use that. There we go. Looks like that just finished.

```typescript
// src/server.ts
import { z } from "zod";

server.tool({
    name: "createUser",
    description: "Create a new user in the database.",
    params: z.object({
        name: z.string(),
        email: z.string(),
        address: z.string(),
        phone: z.string()
    }),
    // ...
});
```

So now we can come into here and we can import `z` from `zod`. And now we have a name field. We can set an email field. We can set an address field. And then we can finally come in here with a phone number. And that is going to be a string as well. So now we have a name. We have a description. We have all the different arguments this thing accepts. It could be an empty object if we don't have any arguments at all.

And then the next step is going to be to define all the different annotations for this. So if we open this up, there's quite a few different annotations we can pass along.

```typescript
// src/server.ts
// ... inside server.tool
    annotations: {
        title: "Create User",
        readOnly: false,
        destructive: false,
        itempotent: false,
        openWorld: true
    },
// ...
```

As you can see here, we have a `title`. This is like a string you can give to the user. This is just a more human readable version. So we'll call this `Create User`. We then have this `destructive` hint, `itempotent` hint, `openWorld` hint, and `readOnly` hint. And these essentially determine how this interacts with different things.

*   `createUser` is obviously not read only. So we're going to set `readOnly` to `false`. This helps the AI know that this is something that does not read data. It also manipulates or creates or inserts or updates data.
*   We also can determine does this destroy things. In our case, this doesn't delete any data. So we can say that this is not `destructive`. Again, this gives the AI a little bit of a hint because if it is a destructive action, the AI may want to warn you and say, "Hey, this is going to delete data. Are you sure you want to do this?" It gives you extra steps of warning potentially.
*   And then we can come in here and we can say is this `itempotent`. What that essentially means is if you run this multiple times, is it going to have any side effects or is it going to change things differently? In our case, if you run this function multiple times with the same input, it's going to create multiple users. So obviously this is not `itempotent`. So we're going to come in here with `false`. You can kind of think about this. Is this a pure function or not? And if you're kind of unsure what a pure function is, I'll link a video in the cards and description where I talk all about pure functions.
*   Now finally, we can come in here with an `openWorld` hint. And this determines is this something that accesses data outside of the particular environment it's inside of. Does it go and access the web for data and things like that. Now in our particular case, I'm going to set this to `true` just because we're interacting with a fake database and a database would be external to our particular application. So it is going to interact with something external to itself.

So these sections right here for all these annotations, they're all optional. We don't have to pass any of these along. I can actually remove this entire parameter and everything is still going to work fine. The only reason I'm passing along all of these different parameters is because it allows me to provide extra information to the AI to give it the best possible chance of using these tools the best that it can.

Now, finally, we come into our function down here. This is going to get all of our different parameters. And if we hover over this `params` type, it's just giving a bunch of errors right now. But essentially, this `param` type right here is a type that is this name, email, address, and phone.

Now, in order to create and interact with a user in a database, we need some type of database. In our case, I'm just going to be using a file as our database. So, we'll come in here with a file at the data location. We'll say `users.json`. And I'll just paste in some random placeholder JSON data for three different users. It doesn't matter what this data is. This could be a real database. It could be an API request. It doesn't matter how you interact with your data. This is just a placeholder for whatever that data interaction looks like.

```json
// src/data/users.json
[
  { "id": 1, "name": "User One", "email": "one@test.com", "address": "123 Main St", "phone": "111-111-1111" },
  { "id": 2, "name": "User Two", "email": "two@test.com", "address": "456 Oak Ave", "phone": "222-222-2222" },
  { "id": 3, "name": "User Three", "email": "three@test.com", "address": "789 Pine Ln", "phone": "333-333-3333" }
]
```

Now, here we can actually go ahead and we can create a user. So, I'm going to wrap this in a `try catch` just in case we have any errors. I want to make sure we return an error down to the actual AI to let it know there was an error. And when you're working with these AIs, they expect you to return down a very specific response. Inside this response, it should be an object. We need to pass along what the `content` is going to be inside of here. And this `content` is an array of data. And inside of here, you can see we have a bunch of different stuff we can pass along. In our particular case, we're going to be passing along text. So, we're going to set the `type` of this to `text`. And this is just going to be a little bit of a warning. So, we'll pass along the `text` portion. And this is just going to say `failed to save user`. So, we're just essentially passing along an error to the AI. That way they can notify the client or the person using the AI, hey, this didn't actually work like we expected it to.

Then inside of our `try`, we can just really simply come in here and we can call a function called `createUser`, which we're going to create in a second. We'll pass along all the different parameters that we want and this will return us an ID that we get for that particular user that was created. Then we can return some information inside of here. So I'll put a little return. We can get rid of this one down here and this will be a successful one. Essentially, we'll just come in here and say `user`, pass in the ID `created successfully`. There we go. So, now we know what user was created and that they were created successfully.

Now, we can just go ahead and create that really simple function for creating a user.

```typescript
// src/server.ts
import fs from "fs/promises";

async function createUser(user: { name: string, email: string, address: string, phone: string }) {
    const { default: users } = await import("./data/users.json", {
        with: { type: "json" }
    });

    const id = users.length + 1;
    users.push({ id, ...user });

    await fs.writeFile("./src/data/users.json", JSON.stringify(users, null, 2));
    return id;
}
```

So, we'll say `createUser`. That's going to take in all of our params. So, we're going to have a user object with a name, which is a string, an email, which is a string, an address, which is a string, and finally a phone, which is a string. There we go. That gets rid of all the errors that we had up here. So, this entire function is essentially done. We don't need to look at this anymore. All we need to do is implement the actual creation of a user inside of our database.

Now, to do this, we're going to use a little bit of an experimental feature inside of Node.js that allows us to dynamically import JSON files. So, we don't have to worry about the file system or anything like that. We can just directly import by saying that we're going to get our users. We're going to call the `import` function. We're going to pass it in the path to that data folder. Just like that. There's `users.json`. And then this second argument here, we can say `with` and we can pass along the `type` which is going to be `json`. So this allows us to do an experimental JSON import where we're importing essentially all that information and it's coming directly into this user's object. And as you can see we get all that information in this default object. So what I want to do is I want to come in here with a simple `then`. I want to take my module and I want to get the default export. So now you can see we get an array of information being returned to us.

Now what I can do is I can come in here. I can say my ID is just going to be my `users.length + 1`. And then we can push in a new user by saying `users.push`. We can push along that ID as well as all of our user information just like that. So now we have a brand new user in that array. And then we can use the file system to actually write it out. So we'll come in here. We'll import `fs` from and we want to get that from `node:fs/promises`. There we go. That gives us the promise version of this particular library. We need to make sure we export it like that or import it like that. Sorry.

So now we can come in here with `writeFile` and essentially we can just write whatever our user array is back to that particular file. So let's copy over that path to our file. This we need to make sure comes from the `src` folder. So we're going to say `src` like that because we're writing a file is essentially from our root directory while this import is relative to the file we're inside of. Then what we can do is we can pass along the data we want. So we can just stringify up our users and we're going to make sure that this prints itself pretty. So we can come in here with `null` and `2` and that's going to make sure it prints it with proper spacing and so on like we have here already instead of just collapsing it down to the smallest size possible. So now we can wait for that to finish and we can return the ID of the user. So this function all it does is just add a user to this particular file when we call the function.

And up here this tool allows an AI user to know hey we have a name, email, address, they're all strings that need to be passed along and it's saying all this additional information about it and it's going to just call that function for us. So, we've essentially created a way for our AI to know that we have the ability to create users. It knows what all the parameters are. It knows additional information about this function. And now, when I try to interact with my AI, I can tell it, hey, create me a user with this name, this email, this address, and this phone. And it'll go ahead and push that information to this tool for me.

So, now let's go ahead and test that. We can save this file, and we're just going to go to where we're doing our inspection. And I'm just going to restart that server completely. This is why I removed that auth section, so that's a lot easier to work with. And if we bring over our **MCP** server, we can give this a quick refresh. We can click the connect button. We should see now we have tools. And the really nice thing about these **MCP** servers is they have the ability to list tools. So we can click list tools. And it gives us a list of all the tools that we have available. Then with that list tools, we can say, hey, we have our `createUser` tool. It has our description. We can click on this tool. You can see we get fields for all the different information. So I can come in here with Kyle, test@test.com. Let's say the address is just bogus and it's just some random numbers for the phone. We can click run tool. And that's actually going to call out to our API. You can see it says it was a success and it says user 4 was created successfully. And if we go back to our application in that `user.json` file, we see we get user 4 with all that information that I typed in.

### Integrating with an AI Client (VS Code & Copilot)

So at least we know that we can inspect this and that it's working. But how do we interact with an actual chatbot to make this work? Now depending on what chatbot you're using, it might be slightly different, but overall it's going to be relatively similar. Inside of VS Code, if I hit `control+shift+p`, I can search for **MCP**. And you can see here that I have a list servers and add server. List server just shows me the servers I have configured. And add server allows me to add a brand new server. And that's obviously what we want to do.

So we're going to click on that. And you can see here we have the option from the standard IO way, the HTTP way, or we can hook up npm packages directly that export this configurations as well as Python and Docker. In our case, we have standard IO. So that's what we're going to hook into. And all we need to do is write whatever the command we want to run is. So in our case, this is that `npm run`. And that is going to be `dev`. Just like that. That's the command to start up our environment. That is our **MCP** server. Depending on how you have your server configured for standard IO, you may run different commands depending on what you install, but that is our command for our application.

Then we can give our server a name. We're just going to call this `Test MCP Video Server`. There we go. It doesn't matter what the name is. And now what we can see is where do we want to save this? I'm going to save this inside of my workspace settings so that it's only available in this particular workspace. So we can see what that looks like. And now if I give that a quick save, you can see I have a `.vscode` folder with an `mcp.json`. And inside here, it gives me all the information for my server. So you can see it runs `npm run dev` just like that. And you'll also see it tells me right here how many tools I have running in this particular server.

Now there are a few different things that I want to show you about this specifically in VS Code that make it really nice to work inside of because we have really good debugging tools as well. We have an option that we can pass in called `dev`. And what we can pass inside of here is the ability to debug.

```json
// .vscode/mcp.json
{
    "servers": [
        {
            "name": "Test MCP Video Server",
            "connection": {
                "type": "stdio",
                "command": "node",
                "args": ["build/server.js"],
                "cwd": "${workspaceFolder}"
            },
            "dev": {
                "debug": {
                    "type": "node"
                },
                "watch": [
                    "build/**/*.js"
                ]
            }
        }
    ]
}
```

So we can say `debug` just like that. And I can say I want to debug the type of `node`. And what that's going to do is allow me to debug a node-based command. But our case, we're using npm as our command. So we want to use `node` instead as our command. And `node` should essentially run whatever script is in my build file. So `server.js`. Just like that. The only reason I'm swapping this is so that I can use the debugging features if I wanted to. You have to have the command up here match the type. And this type can only be `node` or `python` right now. And we're obviously using node. So we'll type in `node`.

Also, we can pass in what the current working directory is going to be. In our case, we're going to use the workspace folder for our current work direct working directory for where it runs different things from. And then we can also set up a watch command inside of here. So, we're going to say `watch` and we're going to say that this is specifically going to be that `build` folder looking for any JavaScript files. So, anytime a JavaScript file changes inside of here, it'll refresh for when we're doing our debugging purposes.

Now, to make sure that we have files inside of our build section, we need to make sure that we actually run that command. So we can say `npm run build:watch`. That's a command directly from our `package.json`. And all this does is it rebuilds our files every single time that they change and puts them inside of that build folder for us. So if we look at our files, you see we have `build` and inside of here we have our `server.js` file. And that's going to be what's hooked up to our server right here. You can see we have our server running with one tool running.

Now you may need to click this restart button on your server to get it actually running. And the nice thing about that is it shows you all the output for your server down here. So you can actually see the communication between our server and our client. It's giving back information such as, hey, here I exist. Here give me all your different tools. Here's all the different tools you have. Give me all your resources. And so on. So you can see it's like, hey, what do you have? What do you have here? What do you have? So it's telling me all the stuff this thing has, then you can actually work with GitHub **Copilot** or whatever AI tool you want.

Now, you may need to restart **Copilot** or whatever AI tool you have as soon as your server is added the first time just so that it can hook it up. But if you want to run a command or a tool, should I say directly from your AI chatbot, just hit the hashtag symbol here inside of GitHub **Copilot** and you can type in `createUser`. And you can see I have that command right here. And if I hit enter, it's going to work on that command and it'll most likely say, hey, I see that you have this. I need specific parameters from you. So now it's essentially giving me a bunch of different parameters. And I just went ahead reduced my font size a little bit, but as you can see, it's giving me a name, an email, an address, and a phone. Just a generic version of all these things. And I can essentially say, hey, do I want to particularly do this? Because you can see it's saying all these fields are required. Let me create that for you. And if I click continue, it'll run this command with this particular input. So if I hit continue here, you can see it's taking a second. It's running. And you can see it says it successfully created a user with the ID 5. You can even see the response that I got back as well as the input that was passed in. And if we look at our user data, you can see user 5 with all that data passed in.

So now I was able to successfully call a function on my server from an AI chatbot just by asking it to do it directly. Now I can also ask it to do it indirectly as well. I could come in here and I could say, "Can you please create a new user for me with the name Kyle, the email test.com, the address 1234 Main Street, and the phone, and just some random numbers." And it should go ahead and actually use that function for us as well. As you can see, it detected that we have a `createUser` function. It's using all that information I passed in. And if I click continue, you can see right away it created a brand new user with all that information that I asked it to do. So now we have tools hooked up and we can call them inside of our client and we can use them on our server to do some really cool stuff.

### Implementing Resources

So now I want to go ahead and start looking at **resources** before we dive into some of the more complex things such as sampling. So we'll come in here. We collapse down that tool. And to do a resource, it's super simple. Just type in `server.resource`. And we're going to pass in a lot of the similar stuff. We need to have a name. We'll call this one `users`. Just like that. We also need to pass in a URI. Now, this is essentially a unique identifier that we can create however we want, but it must match a certain protocol. The protocol is relatively straightforward. Essentially, it's similar to like a URL, but it can be referencing anything. It doesn't have to just be HTTP, for example. We could have a file in here or we could even have our own custom type. For example, we could call this like a `users` type and we could have it linked to `all`, for example. It doesn't really matter what you do. It just must match this similar format that you get with HTTP inside of like web browser URLs. But it can match anything whether it's a file, users, whatever. So depending on your application, you may have your own standards you use. You may create your own schemas. We're just going to use this `users` protocol at the start here because that's just a custom one that we can create for our application.

```typescript
// src/server.ts
server.resource({
    name: "users",
    uri: "users:all",
    // ...
});
```

Then we can specify again some additional properties inside here. So we can specify for example a description that helps the AI know what this does. So we can say `get all users data from the database`. There we go. Then we can come in here with a title. This is again human readable title. We'll call this `Users`. And we can also specify a mime type. For example, what type of data is being returned. In our case, we have `application/json`. So we'll say that it's JSON data.

Lastly, we have a function which is going to take in whatever that URI is that I'm trying to pass in. And we can come in here with that information. Now, this URI, if we hover over it, is essentially just a URL. So it's relatively straightforward what it is. And inside here, we return essentially the exact same thing we returned from our tool down here. It's that same exact like content and so on. So, I'll come in here. I'll copy that over. And our content is going to be a type of whatever. And then we're going to pass along some information for it. So, what we want to do is we want to get all of our users. Well, we already know how to get our users because we did that inside the `createUser` function. So, I'm just going to copy this and I'm going to paste it up here because getting our users is relatively straightforward.

Then, what we can do is we can pass down our content information. And this is going to take not exactly this information because it doesn't really need a type. Instead, we're going to have a URI that we pass along, which is just our `uri.href`. Essentially, what is the URL of the resource that we are trying to access. Then, what I want to do after that is I want to get our text. So, our text here is just going to be a JSON stringified version of our users. And then finally, we can pass along a mime type just like this. And the mime type in our case is `application/json`. And that just tells our application how to actually use this particular piece of information. We know that it is JSON data being returned to us.

And now let's go ahead and test this before we actually start working with it. But you do see we have an error. Actually, the reason for that error is that this should say `contents` instead of `content`. That should hopefully fix our error. And also, I misspelled `text` down here. There we go. That actually cleaned up all of our different errors. So now we have the correct information being passed down to use these resources.

Now, in order to test this, all we want to do is just restart our inspection server. We're going to pull over our inspection server, and we're just going to give this a quick restart. And as you can see, now we have the ability to list out our resources. You can see we have this `users` resource. I'm going to open that up. And if we look over on our screen over here, you can see we get our URI, which is our `users:all`. And the text inside of here is just an array of all of our information inside that JSON format. So you can see it's returning to us all of our different user data.

So now let's see how we can actually use this inside of our AI chatbot. This is relatively simple to do inside of GitHub **Copilot**, for example. You can add this as a context. So I can click add context. And you'll actually notice it does not list the information for our resources. You should see a section inside here for resources, but we don't see it. The reason for that is because when an **MCP** server starts up for the first time, it asks the server, "What are all your resources and what are all your tools?" Well, our server has already started and it told our client, "We have one tool and we don't have any resources." So, we just need to restart our server because now we've added resources and tools into this section. So, now we should hopefully see that it has those tools inside of it. And if I go to add context, looks like it's still not showing up. My guess why this is happening is because I need to restart **Copilot** because now I have new information for it to access. I can try creating a new actual chat and see if that pops it up, but it does not look like it does. One nice thing is I can come in here and I can click on browse resources and we can see that resource right there. And when I click on it, it is going to open up a file with all that information inside of it, which is great. But I want to get it inside the chat window. So let me just restart that real quick.

Now all I did was just restart Visual Studio Code. That restarted everything. And now when I click add context, you can see I have **MCP** resources and I can open up any of those resources. If I make this a little bit larger, it should be easier to see. You can see I can click on that users. And now if I click on that, it adds that into the context and I can say `what is the name of the user with the ID 4`. And now it should tell me whatever that user is. You can see it says their name is Kyle. And if we double check that, you can see that that name is Kyle. So at least it's giving us information. We can use that resource for different things inside of the context chat for our application.

### Resource Templates

Now getting a user based on the ID may be something that's really common that we want to be able to do a lot. So we can actually expose a resource directly for that. So what I can do inside my server is I can create a resource template.

```typescript
// src/server.ts
import { ResourceTemplate } from "@mcp/sdk/server";

server.resource(
    new ResourceTemplate({
        name: "userDetails",
        uriTemplate: "users/{userId}/profile",
        // ...
    })
);
```

So we'll come in here. We'll say `server.resource`. I want to add a resource for `userDetails`. And that's going to give me all the information for a user based on their ID. And instead of passing along just a single URI like this, I want this to be a template. So we can say we want a `new ResourceTemplate`. And this resource template is going to look like `users/` just like that. We're going to have a dynamic user ID. We're going to put that inside of these curly brackets to denote that this is a dynamic parameter. And then we'll say that this is like the profile route or something like that. It doesn't really matter too much as long as you have consistent URIs, but that's just going to be the URI for us.

Now, also, we're going to come in here and we're going to say `list` and we're going to set that to `undefined`. Essentially, this list is for matching all of the resources that match this template. We don't really care about this, so we're just going to set it to `undefined` to essentially completely ignore it. Then, we can go along and we can pass essentially all the same stuff we passed to here, our description, so on, as well as our function. So, let's go ahead. I'm going to paste all that in. close off our function just like that. And now we can specify that this is going to be getting a user's details from the database. And we can come in here. We can say it's going to be `User Details`. And it's going to return to us JSON data.

Now, we want to do pretty similar stuff to what we did up here. So, I'm actually just going to copy this. I'm going to paste it into here. And then we'll modify it to get a particular user instead of all of our users. And that's because this function not only takes in a URI, but it's also going to take in all of our parameters. In our case, our `userId` parameter is the only one that we really care about. You can see this is going to be a string or a string array because it works just like a URL essentially. So now we have our user ID information. We can come in here. We can get our user which is equal to `users.find`. And we want to get the user here where the `user.id` is equal to parsing an int for our `userId`. And we know that this is going to be a string. So we'll just cast it to a string just like that.

This is going to get us a user. Obviously if we don't have a user we want to let them know. So if `user` equals `null`, well essentially just returns no information. So we can return `contents`. `contents` is going to be an array. We're going to pass in our `uri.href` just like we did before. We're going to pass in some text. And this is going to be JSON to match that up here where we had our mime type. We want to make sure that matches. So we're going to pass along JSON that just says `error: user not found`. And then finally, we can come in here with that mime type, which is going to be `application/json`. Then what I want to do down here is instead of returning all of my users, I want to just return a single user if I'm able to find them. So if I don't find them, give them an error. Otherwise, return what that user information is.

So now I have a brand new resource that I can use. Let's try to restart our server to hopefully see if it picks up on that new resource. If we come in here with **MCP** resources and we expand this, you do see we have our user detail right here. When I click on this, you can see it's asking me for a value for `userId`. Let's type in four, like here. And we're going to click enter. And now it added in the information for that user right here. And I can say, what is this user's information? There we go. And it should just hopefully print out whatever that user's information is. And you can see the user with the ID 4 has this particular information. So this is a really great way for me to be able to access different resources and attach them into queries. For example, if I wanted to do something with user 4, I could just come in here. I could say I want to get the user details for four. And now my AI knows that this is the user with all this information. And I can do something like make a sale for this user. And it could do different stuff on the back end for all of that.

### Implementing Prompts

Now, the next thing that we need to work on is going to be creating **prompts**. And **prompts**, like I said, are much less used than tools and resources, but they're still really handy for particular use cases. In our particular use case here, it's going to be a little bit contrived, but it's going to work out just fine. So, let's come in here and minimize a lot of this code that we have because we don't need all of this to be open all the time. And down here below all of this, we're going to come in here, `server.prompt`.

Again, just like everything else, we need to give it a name, description, all that different stuff. So, we'll call it `generateFakeUser`. Just like that. And this will say `generate a fake user based on a given name`. So, this allows me to create a prompt that's going to give me some fake user information. I also need to pass along any params that I have. In our case, we just have one param, which is a name. So, we'll pass along the name just like that.

And then finally we have our function which takes in whatever that parameters is. So we have our name object is the only thing being passed in. Now inside of here we can return an array that contains messages and these are essentially the messages that we want our AI to run. In our case we just have one single message. We're going to say that the role for this one is `user`. And then we're going to specify the content for that message. So we'll say `type` here is going to be `text`. And then we can specify what the text of that message is going to be. I'm going to copy this over because it's kind of a little bit of a long text. So, we'll just paste that in. Essentially, it just says `generate a fake user with the name, whatever our name is. The user should have a realistic email address and phone number.` So, essentially what this prompt does right here is I give it a name and it's going to return to me all of this prompt right here that I can run inside of my AI tool. So, this is a really powerful way to create really complicated prompts from very small amounts of information. In this case, it's not a very complicated prompt, but you can hopefully see the power of this as we start to implement it.

So, before we use this inside of our **Copilot** tool, let's first of all get it working inside of our inspector. So, we can just close out of our inspector, rerun it so that it starts up with the latest code. We can bring this over real quick. Click on the restart button. And now you can see we have a prompt section. We can list our prompt. Here is our prompt right here. We can give it a name, Kyle, for example. There we go. And now I can say get prompt. And as you can see, it returns to me that exact prompt with that name being passed in right here. Obviously, you can make yours more complicated than this, but that's going to work for our particular use case.

Now, to run a prompt inside of **Copilot**, we just need to come in here with a slash command. And you can see this `Test MCP Video Server` and it's `generate fake whatever`. If I get it expanded all the way, you can see `generateFakeUser`. Gives me my description. I can just say that I want to run that particular prompt. I can insert whatever that name is. So we'll just say that the name for this one is going to be Sally. There we go. Now I can hit enter and it's given me back a prompt. `Generate a fake user with the name Sally. The user should have a realistic email address and phone number.` So now when I hit enter on this, it should generate that user for us. As you can see, it gave me information for a particular user. And now I can add that user to the database. It even says, "Would you like that to be added to your data? Let me know if you want it inserted." Yes. Insert. And now hopefully it's going to call that function with that brand new data. Actually, it's directly just inserting it into my JSON file, which is not quite what I want. But obviously, if you were working on this in a more real world application, you wouldn't have access to the database file inside of your project. So, mine just read that database file and inserted it there. But, it could use that `createUser` function, and normally it would if obviously that database file wasn't there.

### Implementing Sampling

Now, the next thing on the server side before we start diving into the client side is going to be **sampling**. And this is kind of where we bridge between the client and the server. So let's just kind of minimize that down. We'll get a brand new setup started. So when we start working with this new tool, we can actually see how the sampling works. And to do this sampling, I'm actually going to do it inside of a tool.

So we can say `server.tool`. And for this tool, I want this to be called `createRandomUser`. And it'll just say `create a random user with fake data`. There we go. And then inside of here, we're going to pass along all of the different additional things. Same thing that we did inside of our tool here. This one doesn't take parameters. So we're going to skip that step and we're just going to copy this code over directly. So we can come into here, paste down all that code and this one is going to be called `Create Random User` and in our particular case obviously it is not a readonly. It is not destructive. It is not item potent and it does access things that are external because it's going to create a user in our database. So we have essentially the exact same prompts here.

Now we have an asynchronous function just like that. And for this function I need to get random fake user data. Now, I could write some type of program to do that for me, or I could say, you know what, I just want the AI to give me fake data for me. So, I'm going to use **sampling** to tell the AI, I have a prompt I want you to run, and then I want you to send me that information.

So, to do that, we can call the `server.request`. And `request` allows me to send a request directly to whatever the client is. So, we can come in here with that request and we can specify the method in this case is going to be `sampling/createMessage`. And this allows me to run a prompt on the AI. Elicitation allows me to ask the user for additional information while sampling allows me to essentially say, "Hey, run this prompt on your AI." So, we want to run this particular prompt. I can come in here with `params` and we're going to say that we have our messages and the messages are just going to be an array of the things we want to run. So, in our case, we're going to have a role of `user` and we can have some content just like that. It's going to be a text prompt that we want to run and it's going to have a bunch of text inside of it. I'm just going to copy this over, but as you can see, it says `generate fake user data. The user should have a realistic name, email address, and phone number. Return this data as a JSON object with no other text or formatter so it can be used with JSON.parse.`

Lastly, we need to pass along a max token. So, we'll just say `1024`. We don't need a ton of tokens. We'll just pass along a value here. And then finally, at the very very end of this request right here, we need to essentially specify what the schema is. In our case, this is a `createMessageResultSchema`. Just like that. So, now if we give that a save, you can see all this is working. And we're going to get some results back from this by calling `await`. And if you look at this result object, you can see here it's got a bunch of information for all the messages and stuff being passed through.

So now right below this, what I can do is I can use this message. First of all, I can say if my `res.content.type` is not equal to `text`, and it looks like I'm getting an error, and that's because I imported the wrong thing. This is a `messageResultSchema`. Essentially, this is supposed to be specifying what the result of this request should be. So this is the result of my call out to this. And now you can see that I have that content. And if the type is not equal to `text`, well, I'm just going to return some type of error. So, I'm just going to say, hey, I expected some text because I expect some JSON formatted text. So, I'm just going to say there's failed to create the user.

Then what we can do below that is we can just wrap a `try catch` to actually try to use this fake user information. So, I can get my fake user. I want to do a simple `JSON.parse` and I want to take my `res.content.text`. I'm going to make sure I trim that text to get rid of any whitespace. And I'm also going to do a little bit of array replacement because a lot of times when I'm using these different AI tools, they like to wrap things inside of markdown. So I want to remove the markdown from this if it has any. So I can come in here and I can say I want to replace anything that starts with that ````json` or anything that ends with those triple backticks. Those would be the JSON way of formatting code. We obviously don't want that. So we're going to remove those from us.

So now that's going to give us all the information for our fake user. And to finish off our `try catch` just to get rid of all our errors, we can do that just like that. Now, if we do have a failure, I want to just make sure we come in here and we can return that information. So, inside of our catch, if we have some type of failure, we'll return that. Now that we have our fake user, you can see this is our user information. What we want to do is we want to get our ID by calling `await createUser`, passing it in our fake user information. And then we can turn return all of our information down here. And this is just going to say `type: text`. And the text for this is going to be `user`, pass in that ID, `created successfully`.

The whole purpose of this function is not really to be super useful because you're probably not going to write code like this very often, but it's to show you how this sampling style interaction works. So now we have a brand new tool. We'll need to most likely restart our server so it knows about that. Actually, it already restarted for us. As you can see, we have that watch set up. So it looks like it was smart enough to do that. So now we can come down here and I can type in `create`. And you can see now we have `createRandomUser`. And if I run this, it should hopefully respond to us by saying, hey, do you want to run this prompt? So we'll just click enter on this. It's going to work. And now it says, do you want to run this function? That's fine. This is to run the tool. And now you can see this message essentially saying has issued a request to make a language model call. Do you want to allow it to make request? Essentially, this is them confirming, hey, your tool is asking you to run a command. Is that okay? I'm going to say that that's okay. I'm going to allow that.

... after some debugging ...

After lots of looking through the code and debugging and trying to figure out what the particular problem was, I realized that the problem all along was that I had this `trim` in the wrong place. I had the `trim` after my `JSON.parse` which essentially meant I was trying to trim an object which clearly wasn't working and throwing an error. So now if I put my `trim` right here that should fix my particular problem because I just want to trim after I do all my replacements.

So now let's try that out. Let's just call that `createRandomUser` function. You can see here it's trying to run `createRandomUser`. It's going to send a request to the AI to say hey give me some fake data. It's going to then send that back to our application up here. We don't even need this console log anymore. And inside of our application, it's going to use that information to try to create a user right here. And as we can see, it created user 8 successfully. And if I look at my actual `user.json` file and I look at user 8, you can see it created all this information based on a prompt that was sent down to the AI. You can see none of this information was something that I typed in myself. The AI randomly generated all that information for me. So this is essentially how we set up to use these different AI prompts directly inside of here by using a sampling to tell the AI, hey, give me information from this prompt and then you can use that information in whatever tool, resource or prompt that you want.

## Building an MCP Client

So now with that done, we can actually go ahead and we can work on creating our client because we've done everything that you could do with a server. Obviously, you could get more complex with more tools, resources, and so on, but that's the basics of how you set up all of these different systems, and you just need to integrate it in your own project.

Now, we can go ahead and we can create a `client.ts`. Obviously, you normally wouldn't create these in the same location, but this is just going to be for testing purposes to show you how to create a client. Doesn't matter where you put them. We're just going to put them in the same project so we can have them side by side.

### Client Setup

So, we can come in here and we can say we want to create a new client that's coming from that model context protocol library again. And we pass in pretty much the same things we did into the other thing. So, we have a name. We're just going to call this our `Test Client Video`. Then, we come in here with a version. We'll call this `1.0.0`. And then what we come in here is we could specify all the different capabilities that this thing has as well. So we'll say `capabilities`. In our case, we have `sampling` as our capability. So we'll pass that in as an empty object. And again, because that sampling is something that happens on the client, we tell the client, hey, I want you to give me information. So it's a client capability.

```typescript
// src/client.ts
import { MCP } from "@mcp/sdk";

const mcp = new MCP({
    name: "Test Client Video",
    version: "1.0.0",
    capabilities: {
        sampling: {}
    }
});
```

Then we need to specify our transport protocol. And again, this is just like on the server. We need to specify how we're doing our transport. So, we're going to use a `StandardIOClientTransport` this time. And in here, it's going to look very similar to what we have in this MCP file with command, args, and so on. Because if you come in here, you can see we have a command. So, we can come in here and say that that's going to be `node`. We have our `args`. In our case, that's going to be `build/server.js`. Just like that. So, there we go. That is our server now running. And we can also come in here and we're going to say `stderr`. And we're going to set that to `ignore`. So if something throws an error and prints to the error console inside of our server, it won't show up in our client. Now the only reason that I'm doing that is because we're using experimental features inside of node specifically features for example here where we do our import with a JSON file and this actually prints a warning to the standard error saying that there is a experimental feature. So I'm just making sure I ignore that so it doesn't show up in our output because we're going to be creating a client interface application a CLI to interact with this. And you can do this however you want. You can create a website, CLI, it doesn't matter. This is just the easiest way for me to show you how to get working with this system.

### Building the CLI

So now let's go ahead and we're going to create an `async` function called `main`. Then below that we'll just call `main` just like that. So now we at least have this async function. And what I want to do is I want to first connect. So I can say `mcp.connect` and I want to connect to that particular transport. So we can say `transport` just like that.

Then I can try to get my tools, resources and so on. So I can say `await mcp.listTools` and that's going to give me all of my tool information. So I can come in here, I can say `tools` just like that and probably you should wrap this inside of like a `Promise.all`. So we're going to do that. We'll just put all of these inside of here. So we'll do our `listTools`. Then we'll come in here. Whoops. There we go. `mcp.listPrompts`. We're going to do another one. There we go. `mcp.listResources`. Then we're going to get `mcp.listResourceTemplates` and that's all the stuff we want to get from the server essentially give me all the things that you can do and this is going to return to us a bunch of data and we're just going to extract that data out by calling `await` right here so the first thing we're going to get is our tools, then the next thing that we're going to get is our prompts, then we're going to get our resources and finally our resource templates. So essentially this is just all the things that our server is capable of listed out for us so we can use each of those individual things.

Now for creating our AI application it's going to be a CLI. And to make working with the CLI a little bit easier, I'm actually going to create a or I'm going to use a library called `@inquirer/prompts`. That's going to allow us to really easily work with these prompts. So, we can say `npm i @inquirer/prompts`. And I also want to get `dotenv` as well. Oops. Make sure I spell that correctly. And that's because we're going to create a `.env` file. And this file is going to have an API key for **Gemini**. So, we can just come over here. We can paste in that **Gemini** API key. And this allows us to interact with the **Gemini** AI. The reason I chose **Gemini** is because they have a free tier that you can use. So you don't need to pay any money to actually try to test this out.

So now we can come in here. We can go into our client and we can make sure that we use all this information, specifically that inquirer prompt. So first I'm just going to log a simple message that says `You are connected`. So that'll show up inside of the console. And then I'm going to set up a simple `while` loop that's just going to infinitely loop forever because that's going to essentially be our main menu where we select our different options like execute a tool, a prompt, and so on. And I'm going to come in here and I'm just going to say `const option is equal to await select`, which comes from that prompt, and it essentially gives me a select option inside of the command line. It's going to pass in a message that says, "What would you like to do?" And then we can pass it in all of our different choices. And our choices are going to be `query`, which would just be calling our AI directly. We can also call a `tool` directly, we can call a `resource` directly, or we can call a `prompt` directly. There we go. So now that gives us all of our different options. we can just set up a simple `switch` statement on our option just like this to call each one of those.

#### Handling Tools

Let's start with one of the more basic cases which is just going to be calling a tool directly. So the first thing I want to do is I want to essentially create a select that's going to be for getting a tool. So we can say we're going to select a tool name. We can just say `select a tool` and then I can list out my choices which is going to be all of my different tools. So I can take my tools, I can map through each tool and I can just return information in that format. So, I have here a name, which is my `tool.annotations.title`. Or if we don't have a title, we can just use the `tool.name`. That's just going to be the non-human readable version. Our value is our `tool.name`. Just like that. And then finally, a description that will show up on the element for us to make it a little bit easier to read is just going to be our `tool.description`. There we go. So, that's just all the information for our different choices. And then we can just come in here. We'll do a quick `console.log` that just says our tool name so we can test to see if this is working.

Now, inside of our `package.json`, we need to create a script for this. So, we're just going to say that this is going to be our `client:dev`. And that'll just run `tsx src/client.ts`. There we go. So, now I can run `npm run client:dev`. And we should hopefully see our command line tool pop up. You can see what would you like to do? I have all my options. I'm going to click tools. You can see my two different tools popped up. They have human readable names with descriptions. And if I click on the tool, it prints out what tool I selected because we haven't implemented any of the rest of the logic yet.

So let's implement the basic logic for that. Essentially, I'm going to see if we can find a tool with that particular name. If we can't, I'm going to log out an error. And then if we find a tool, we're going to call this `handleTool` function. And the reason I'm putting this in a function is just to make it a little bit easier to work with. So let's get that tool type imported. Make sure this is an `async` function. And now we can work on actually calling this tool.

Now the really nice thing is we have an `mcp.callTool` function that allows us to pass in the name of our tool, which is `tool.name`. And it allows us to pass in our arguments, which is just going to be some arguments that we're going to create. And it allows us to call our tool with that information. Now, in our case, we need to get those arguments. So, we're going to say `const args`, which is just going to be a record of strings that map to other strings. That's going to be an empty object. There we go. And then what I want to do is I want to go and loop through each one of my different arguments. So I'll say `value` just like that of `Object.entries`. I'm going to pass in my `tool.inputSchema.properties` and we'll default that to an empty object. So essentially what I'm doing is if I have parameters, I'm going to be looping through each one of those and that'll give me a value as well as a key for all the different information that we need.

Now if we look at this properties object, you can essentially see here it has an unknown type. It doesn't really know what those properties are supposed to be, but our properties will specifically have a `type` keyword on them that we can work off of. So what I can do is I can say I want to take an argument for the particular key. So if my parameter name was age, this would say age right here. And then what I want to do is I want to say `await input` which is another prompt that I can get. And this will allow me to get input user from the data. And I can say whatever I want. For example, `message: 'Enter value for ' + key}` just like that. And then I can put what the type that's going to be as well by saying `value.type`. And this is going to give me an error because this is an unknown type. So, we're just going to give that a little bit of a cast right here. So, we're going to say it's `as { type: string }`. Just like that. Make sure I get my parenthesis in the correct location. That gets rid of that error for me. And then I can just put a colon right there.

So, what this little bit of code right here does is essentially it's looping through every single one of the parameters that I defined on my server. It's asking the user to give their input for the name of the parameter and it tells them the type so that way they know what the type of that parameter is as well. Next, we can come down here and we can get the result for that which is just calling `await` on that. And this is going to be essentially a bunch of information that comes directly from the server. So we can just parse out that information and log it out to the user. So we'll say `console.log(res.content[0].text)`. Now in our case again this returns an unknown type which is a little bit annoying. So we're just going to get around that by casting that directly because we don't really care about the TypeScript errors. So we know that this is going to return to us an array and that array is going to have a text which is a string inside of it. Then we just want to make sure we get the first property of that and we want to get the text portion.

So, this should hopefully work by actually calling out our tool and passing along whatever information we want to it. So, let's go ahead and try that out. We're going to close our program and restart it. We're going to go to tools and we're going to try to create a user. And you can see the first parameter is name. So, let's pass in a name of Kyle. Email, we'll say `test.com`. For the address, we'll just put in some random numbers. And for the phone number, let's just do some letters for this one. So, it's really easy to see it in our database. When we hit enter, you can see it says user 9 successfully created. And if we look inside of our data, you can see here we have user 9 with all that data that I passed in. So you can see that this properly called the tool on our server, sent across the information, saved it in our database, and then returned to us a response that told us the ID for that user.

#### Handling Resources and Prompts

... The transcript continues with similar detailed implementation steps for handling resources, resource templates, and prompts in the client-side CLI, including getting user input for parameters and displaying the results. ...

#### Handling General Queries

So let's go ahead and take a look at that. Inside of our switch statement, we'll add another case here. This one is going to be for `query`. And this is just going to call `handleQuery`. Just like that. And we're going to actually pass it in all of our tools so it knows what tools we have access to. Then we can go ahead and just create that function `handleQuery`. `tools` is just going to be a tool array.

Now the first thing we're going to do is we're just going to ask the user enter whatever your query is that you want to ask the AI. So this is just what is your prompt. Then what we're going to do is we're going to try to get a bunch of information. Specifically, we're going to get text as well as tool results by calling `await generateText`. There we go. And this `generateText` is going to take in the exact same model we used down here. So let me just copy that model over. Paste that in. Of course, I failed on my copying. There we go. Paste in what our model is to get the exact same one. Our prompt here is just going to be whatever our query is. And then importantly, we're going to have tools that we're going to pass in.

And this tools is just going to be reformatted into the proper format because it needs an object instead of an array. A little bit annoying, but we can go ahead and we can do a little bit of a reduce to reduce this down to what we want. So we want to take whatever our current object looks like and then we want to add in our current `tool.name` as a new value inside of our object. It should take in a description which is our `tool.description`. It should take in parameters that is our JSON schema which is unfortunate. It comes in JSON schema already but we need to format it using this `asJSONSchema` function from the AI library to actually make it work. So we'll say `tool.inputSchema`. that essentially converts this schema into the correct format to use with this AI library. So just formatting between different libraries.

And then we have an `execute` function. Essentially what this allows us to do is it says, hey, if I want to call a tool, what do I do? And if we pass an `execute` function, it'll just call this `execute` function when it wants to execute a tool. So we can come in here. This is going to be an `async` function that takes in arguments which are a record of string and any because we don't know what the other type is. And then we can actually return some data from here. So all I want to do is I just want to call `mcp.callTool` and we want to call a tool that has a name for our particular tool which is `tool.name` and our arguments are going to be these args right here. There we go. And then I can just return whatever that result is. So we'll just put a return right here. There we go. So essentially I'm saying if you want to call a tool just call this function right here and return that information to the user by calling tool and returning whatever the result is.

Now, we also need to make sure that this reduce function has a default value. We're going to default it to an empty object. And this specifically will be typed as a `toolSet`. Just like that, because that is what this particular object is. So, now we're getting not only text, but we're also possibly getting tool results back. The reason for that is because if we told the AI, hey, create me a brand new user with this name, email, address, and phone number. It'll say, oh, I have a tool for that. I'm going to call that tool, call this `execute` function, and I'm going to give you what the result of that tool is, which in our case is like user created with ID 7 and so on. So what we need to do is we need to make sure we log out the results for our text or our tool results. So all the way down here, we can just do a simple `console.log`. In our case, we have our text. So if we have text, let's log that out. If we don't have any text, then we're going to take our tool results. We want to get the very first result from that because it could be an array. And then what I want to do is I want to get the result. I want to get the content. I want to get the first piece of the content. And then I want to get the text. Now, unfortunately, again, we're getting unknown or never technically as the type here because we're using dynamic parameters here instead of hard-coded parameters. The easiest way to do this is just we're just going to ignore this TypeScript error for now. So, we'll just say `@ts-expect-error` to ignore that error. But, this is the proper format being returned to us. I just don't want to deal with the TypeScript for it. Finally, we'll come in here and we can just say `no text generated`. Just like that. So, it's either going to return to us our text, it's going to return to us what the tool says, or it's going to return just no text generated. And that's all we need to do to handle this query.

So, let's see if that works as well. I'm going to close out of this. Restart our server. I'm going to use query. And of course, we're getting an error because I forgot to put in `await` up here. There we go. So, now let's try that again. Hopefully, we quick query. It's going to ask us for our query. I'll say, "How are you?" Should hopefully just return to me some random AI response. There you go. You can see it returned a response. Let's do another query. And I say, can you create a user with the name Kyle, email test.com, address random address, and phone random phone. Let's see if this actually works. You can see user 10 created. And if we look at our user data, go down to number 10, it has that information. So it was able to use that tool indirectly without us telling us to use it directly, which is really great and important.

#### Handling Sampling on the Client

So now we need to make sure we have sampling working because that's the final step. Now, unfortunately, sampling isn't as easy to set up as everything else because it's not built into the library directly, but it's relatively easy to do. We can come in here and we can say `setRequestHandler`. And what this allows us to do is handle any request that follows a specific schema. And we have the ability to use the `createMessageRequestSchema`. So, we're saying, hey, I'm going to expect a request that is in this `createMessageRequestSchema` format. And when I get that, I want to take in a request right here, and I want to do something with that. So essentially I know that this is going to be the schema for my `sampling/createMessage` which is what's passed down from our server when we call that tool.

Now if we take a look you'll notice that we have an array of messages. It could ask for multiple messages. So what I'm going to do is I'm going to just loop through all of our messages. So I'm going to say `for const message of request.params.messages`. For each one of these I want to call that `handleServerMessage`. I want to pass it along my message and I want to just put this inside of some type of text just like that. So, this is going to return to me a string or undefined depending on if we ran this request or not. And I want to put these inside of an array. So, we'll say `text` is equal to a string array. By default, it's an empty array. And if my text is not equal to null, then I just want to take my text and I want to push in that brand new text. So, essentially, every time I call something successfully on the AI, I want to push the result of that into this array.

Then, I can take that response and send it back to the **MCP** server so it can use it for its particular use case. So I can come in here. I can just say that the role is going to be `user`. Doesn't really matter too much. Our model here is going to be the model that actually is running this. So we can say `gemini-1.5-flash`. There we go. We can come in here with a stop reason why this thing is stopping. We're going to call it `end_turn`. Essentially saying that this is the end of the client's interactions and it's back to the server. Then finally, we can pass along the content which is all the stuff we're sending to them which is going to be text. and it's specifically going to be a text that just takes that text array and it's going to join it on a new line character. There we go. So now we're just combining all that together. And if I make sure I put a comma here, it should hopefully be able to now understand how sampling works. And we should see that inside of our application.

So let's go ahead. I'm going to use the tool for creating a random user. And you can see immediately I'm getting back a response which is that sampling. It says `generate fake user data. The user should have a realistic name, email address, blah blah blah.` Would you like to run the above prompt? This is what's coming from the server to sample our AI. I'll say yes to run that. It's going to run that. Send that information back to the actual server. And now you can see the server created user 11. And if we open up our JSON, you can see we have a random user number 11. It looks like the format was not exactly what I expected, but it's good enough for our particular use case. Obviously, if this is a real world application, I would have some type of schema validation before I save things in my database. But again, this is purely for testing, just to show you the interactions between these clients and these servers.

## Conclusion

Now, if you enjoyed this video and want to see more AI related content, let me know down in the comments. And also, if you want to see a massive project that has tons of AI integrations, I'll link that right over here. It's over 10 hours long and has tons of different AI features built into it. With that said, thank you very much for watching and have a good day.