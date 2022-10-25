# How to Build a Blog with Fresh

_This is an example repo of a blog built with Fresh. Original tutorial can be
[viewed here](https://deno.com/blog/build-a-blog-with-fresh)._

[Fresh](https://fresh.deno.dev) is an edge-first web framework that delivers
zero JavaScript to the client by default with no build step. It’s optimized for
speed and, when hosted on the edge with [Deno Deploy](/deploy), can be
[fairly trivial to get a perfect Lighthouse pagespeed score](https://deno.com/blog/ecommerce-with-perfect-lighthouse-score).

This post will show you how to build your own markdown blog with Fresh and
deploy it to the edge with Deno Deploy.

## Create a new Fresh app

Fresh comes with its own install script. Simply run:

```shell
deno run -A -r https://fresh.deno.dev my-fresh-blog
```

We’ll select yes for [Tailwind](https://tailwindcss.com/) and VSCode.

Let’s run `deno task start` to see the default app:

![Our default fresh app](https://deno.com/build-a-blog-with-fresh/default-fresh-app.png)

Voila!

## Update the directory structure

The Fresh init script scaffolds a generic app directory. So let’s modify it to
fit the purposes of a blog.

Let’s add a `posts` folder that will contain all markdown files:

```shell
$ mkdir posts
```

And remove the unnecessary `components`, `islands`, and `routes/api` folders:

```shell
$ rm -rf components/ islands/ routes/api
```

The final top-level directory structure should look something like this:

```
my-fresh-blog/
├── .vscode
├── posts
├── routes
├── static
├── deno.json
├── dev.ts
├── fresh.gen.ts
├── import_map.json
├── main.ts
├── README.md
└── twins.config.ts
```

## Write a dummy blog post

Let’s create a simple markdown file called `first-blog-post.md` in `./posts` and
include the following frontmatter:

```markdown
---
title: This is my first blog post!
published_at: 2022-11-04T15:00:00.000Z
snippet: This is an excerpt of my first blog post.
---

Hello, world!
```

Next, let’s update the routes to render the blog posts.

## Update the routes

Let’s start with `index.tsx`, which will render the blog index page. Feel free
to delete everything in this file so we can start from scratch.

### Getting post data

We’ll create an interface for a Post object, which includes all of the
properties and their types. We’ll keep it simple for now:

```ts
interface Post {
  name: string;
  title: string;
  publish_date: string;
  snippet: string;
}
```

Next, let’s create a
[custom `handler` function](https://fresh.deno.dev/docs/getting-started/custom-handlers)
that will grab the data from the `posts` folder and transform them into data
that we can easily render with tsx.

```ts
import { Handlers } from "$fresh/server.ts";

export const handler: Handlers<Post[]> = {
  async GET(_req, ctx) {
    const posts = await getPosts();
    return ctx.render(posts);
  },
};
```

Let’s define a helper function called `getPosts`, which will read the files from
`./posts` directory and return them as a `Post` array. For now, we can stick it
in the same file.

```ts
async function getPosts(): Promise<Post[]> {
  const files = Deno.readDir(“./posts”);
  const promises = [];
  for await (const file of files) {
    const slug = file.name.replace(".md", "");
    promises.push(getPost(slug));
  }
  const posts = await Promise.all(promises) as Post[];
  posts.sort((a, b) => b.publishedAt.getTime() - a.publishedAt.getTime());
  return posts;
}
```

We’ll also define a helper function called `getPost`, a function that accepts
`slug` and returns a single `Post`. Again, let’s stick it in the same file for
now.

```ts
// Importing two new std lib functions to help with parsing front matter and joining file paths.
import { extract } from “$std/encoding/front_matter.ts”
import { join } from “$std/path/mod.ts”

async function getPost(slug: string): Promise<Post | null> {
  const text = await Deno.readTextFile(join(‘./posts’, `${slug}.md`));
  const { attrs, body } = extract(text);
  return {
    slug,
    title: attrs.title,
    publishedAt: new Date(attrs.published_at),
    content: body,
    snippet: attrs.snippet
  }
}
```

Now let’s put these functions to use and render the blog index page!

### Rendering the blog index page

Each route file must export a default function that returns a component.

We’ll name our main export function `BlogIndexPage` and render the post data
through that:

```tsx
import { PageProps } from “$fresh/server.ts”;

export default function BlogIndexPage(props: PageProps<Post[]>) {
  const props = props.data;
  return (
    <main class="max-w-screen-md px-4 pt-16 mx-auto">
      <h1 class="text-5xl font-bold">Blog</h1>
      <div class="mt-8">
        {posts.map((post) => <PostCard post={post} />)}
      </div>
    </main>
  )
}
```

Let’s run our server with `deno task start` and check localhost:

![A first look at our blog index page](https://deno.com/build-a-blog-with-fresh/blog-index-page.png)

Awesome start!

But clicking on the post doesn’t work yet. Let’s fix that.

### Creating the post page

In `/routes/`, let’s rename `[name].tsx` to `[slug].tsx`.

Then, in `[slug].tsx`, we’ll do something similar to `index.tsx`: create a
custom handler to get a single post and export a default component that renders
the page.

Since we’ll be reusing the helper functions `getPosts` and `getPost`, as well as
the interface `Post`, let’s refactor them into a separate utility file called
`posts.ts` under a new folder called `utils`:

```
my-fresh-blog/
…
├── utils
│   └── posts.ts
…
```

Note:
[you can add `”/”: “./”, “@/”: “./”` to your `import_map.json`](https://deno.land/manual/linking_to_external_code/import_maps)
so that you can import from `posts.ts` with a path relative to root:

```ts
import { getPost } from “@/utils/posts.ts”
```

In our `/routes/[slug].tsx` file, let’s create a custom handler to get the post
and render it through the component. Note that we can access `ctx.params.slug`
since we used square brackets in the filename `[slug].tsx`.

```ts
import { Handlers } from “$fresh/server.ts”;
import { getPost, Post } from “@/utils/posts.ts”;

export const handler: Handlers<Post> = {
  async GET(_req, ctx) {
    const post = await getPost(ctx.params.slug);
    if (post === null) return ctx.renderNotFound();
    return ctx.render(post);
  }
}
```

Then, let’s create the main component for rendering `post`:

```tsx
import { PageProps }

export default function PostPage(props: PageProps<Post>) {
  const post = props.data;
  return (
    <main class="max-w-screen-md px-4 pt-16 mx-auto">
      <h1 class="text-5xl font-bold">{post.title}</h1>
      <time class="text-gray-500">
        {new Date(post.publishedAt).toLocaleDateString("en-us", {
          year: "numeric",
          month: "long",
          day: "numeric"
        })}
      </time>
      <div class="mt-8"
        dangerouslySetInnerHTML={{ __html: post.content }}
        />
    </main>
  )
}
```

Let’s check our localhost:8000 and click on the post:

![Our first blog post](https://deno.com/build-a-blog-with-fresh/our-first-blog-post.png)

There it is!

### Parsing markdown

Currently, this does not parse markdown. If you write something like this:

![raw markdown blog post file](https://deno.com/build-a-blog-with-fresh/raw-markdown-in-our-first-blog-post.png)

It’ll show up like this:

![unprocessed markdown on the blog](https://deno.com/build-a-blog-with-fresh/our-first-blog-post-with-failed-markdown.png)

In order to parse markdown, we’ll need to import the module
[`gfm`](https://deno.land/x/gfm/mod.ts) and pass `post.content` through the
function [`gfm.render()`](https://deno.land/x/gfm/mod.ts?s=render).

Let’s add this line to `import_map.json`:

```ts
“$gfm”: “https://deno.land/x/gfm@0.1.26/mod.ts”
```

And update the `<div>` in our component to be:

```tsx
<div
  class="mt-8"
  dangerouslySetInnerHTML={{ __html: gfm.render(post.content) }}
/>;
```

Now, our post looks like:

![markdown working successfully](https://deno.com/build-a-blog-with-fresh/our-first-blog-post-with-markdown.png)

Better, but we can make this look nicer by injecting `gfm.css` as a style tag.
Since [`gfm`](https://deno.land/x/gfm/@mod.ts) ships with a stylesheet, we can
access that and include it like such:

```ts
// Other dependencies...
import { CSS, render } from "$gfm";

// ...

export default function PostPage(props: PageProps<Post>) {
  const post = props.data;
  return (
    <>
      <Head>
        <style dangerouslySetInnerHTML={{ __html: CSS }} />
      </Head>
      // ...
      <div
        class="mt-8 markdown-body"
        dangerouslySetInnerHTML={{ __html: render(post.content) }}
      />
    </>
  );
}
```

Note we'll need to include the class `markdown-body` on the div for the gfm
stylesheet to work.

Now markdown looks much better:

![markdown working even better](https://deno.com/build-a-blog-with-fresh/better-markdown-styling.png)

## Deploying to the edge

[Deno Deploy](/deploy) is our globally distributed v8 isolate cloud where you
can host arbitrary JavaScript. It’s great for hosting serverless functions as
well as entire websites and applications.

We can easily deploy our new blog to Deno Deploy with the following steps.

- Create a GitHub repo for your new blog
- Go to https://dash.deno.com/ and connect your GitHub
- Select your GitHub organization or user, repository and branch
- Select “Automatic” deployment mode Select `main.ts` as an entry point
- Click “Link”, which will start the deployment

When the deployment is complete, you’ll receive a URL that you can visit.
[Here's a live version.](https://fresh-blog-example.deno.dev/)

## What’s next?

This is a simple tutorial on building a blog with Fresh that demonstrates how
Fresh retrieves data from a filesystem, which it renders into HTML, all on the
server.

_Stuck? Get help with Fresh and Deno on [our Discord](https://discord.gg/deno)
or [Twitter](https://twitter.com/deno_land)!_
