---
date: 2021-05-23
title: Create a dynamic sitemap with a Next.js API route
shortDescription: How to use Next API routes to create an always up-to-date sitemap for user-generated dynamic routes
category: React
tags: [react, next, sitemap, api]
hero: ./maps-andrew-neel-1-29wyvvLJA-unsplash.jpg
heroAlt: Maps lying on the floor
heroCredit: 'Photo by [Andrew Neel](https://unsplash.com/@andrewtneel)'
---

Sitemaps for our apps, blogs or other sites are important because it allows search engines like Google to more intelligently crawl the site. Typically a search engine can discover the majority of a site if the pages are properly linked. But a sitemap is especially helpful if the site has pages that aren't well linked, is really large, or is pretty new (with few sites linking to it).

Google has a [guide for building a sitemap](https://developers.google.com/search/docs/advanced/sitemaps/build-sitemap) that you can use to manually build your own for your [Next.js](https://nextjs.org/) app. This can work well if the pages in your site are constant or you can determine them at build time. But if you have user-generated content, you likely have [dynamic routes](https://nextjs.org/docs/routing/dynamic-routes). And as a result, there will likely be new pages added in between builds. **So let's see how we can use a Next [API route](https://nextjs.org/docs/api-routes/introduction) to build a constantly updating sitemap.**

Here's the full code for the API route:

```js
// /src/pages/api/sitemap.js

import { createGzip } from 'zlib'
import { SitemapStream } from 'sitemap'

const STATIC_URLS = [
  // all the non-dynamic URLs
]

const sitemapApi = async (req, res) => {
  // ensure response is XML & gzip encoded
  res.setHeader('Content-Type', 'application/xml')
  res.setHeader('Content-Encoding', 'gzip')

  // makes necessary API calls to get all the dynamic
  // urls from user-gen content
  const userGenPageUrls = await getUserGeneratedPages()

  const sitemapStream = new SitemapStream()
  const pipeline = sitemapStream.pipe(createGzip())

  // write static pages to sitemap
  STATIC_URLS.forEach((url) => {
    sitemapStream.write({ url })
  })

  // write user-generated pages to sitemap
  userGenPageUrls.forEach((url) => {
    sitemapStream.write({ url })
  })

  sitemapStream.end()

  // stream write the response
  pipeline.pipe(res).on('error', (err) => {
    throw err
  })
}

export default sitemapApi
```

That gives the sitemap a URL route of `/api/sitemap`. Typically sitemaps live at the root at `/sitemap.xml`. We can use [Next rewrites](https://nextjs.org/docs/api-reference/next.config.js/rewrites) in the `next.config.js`, however, to rewrite `/sitemap.xml` to the real `/api/sitemap` route.

```js
// next.config.js

module.exports = {
  rewrites: async () => [
    {
      source: '/sitemap.xml',
      destination: '/api/sitemap',
    },
  ],
}
```

And that's it! 🎉 Feel free to copy and paste and be on your merry way. But if you'd like a breakdown of how it all works, I've got your covered. 😉

---

We use the [`sitemap`](https://www.npmjs.com/package/sitemap) npm package that does most of the heavy lifting for us. **Our job will be to determine the links to include in the sitemap, while the `sitemap` package will handle generating the XML and streaming it to the response.**

```js
const sitemapApi = async (req, res) => {
  // ensure response is XML & gzip encoded
  res.setHeader('Content-Type', 'application/xml')
  res.setHeader('Content-Encoding', 'gzip')

  // ...
}
```

It's starts off by ensuring that the response type is XML instead of the typical JSON. And we'll be gzipping the XML in order to make the download faster for the search engines. This is particularly important if there are a lot of URLs (like tens of thousands).

```js
const STATIC_URLS = [
  // all the non-dynamic URLs
  '/',
  '/about',
]

const sitemapApi = async (req, res) => {
  // ensure response is XML & gzip encoded

  // makes necessary API calls to get all the dynamic
  // urls from user-gen content
  const userGenPageUrls = await getUserGeneratedPages()

  // ...
}
```

Then we grab the URLs of the pages we want to include in the sitemap. This is going to be very app-dependent. I've abstracted the dynamic, user-generated URLs behind a `getUserGeneratedPages()` helper function. That would make all the necessary API or DB calls to generate all the dynamic URLs.

Then there are the static URLs that are never changing (like the home page for instance). Those can be as simple as a `STATIC_URLS` variable or as sophisticated as reading the filesystem for all of the non-dynamic page routes (using [`readdir`](https://nodejs.org/api/fs.html#fs_fspromises_readdir_path_options) for instance).

```js
import { createGzip } from 'zlib'
import { SitemapStream, streamToPromise } from 'sitemap'

const sitemapApi = async (req, res) => {
  // ensure response is XML & gzip encoded

  // get URLS

  const sitemapStream = new SitemapStream({
    hostname: 'https://yourdomainhere.com',
  })
  const pipeline = sitemapStream.pipe(createGzip())

  // ...
}
```

Next, we finally start using the `sitemap` package. We create a stream that is configured to prepend the specified `hostname` to all of the URLs added to it. We also pipe in a `GZip` object from the native [`zlib`](https://nodejs.org/api/zlib.html#zlib_zlib) library. This way the generated sitemap will automatically be gzipped as it is streamed to the response.

```js
const sitemapApi = async (req, res) => {
  // ensure response is XML & gzip encoded

  // get URLS

  // create sitemap stream

  // write static pages to sitemap
  STATIC_URLS.forEach((url) => {
    sitemapStream.write({ url })
  })

  // write user-generated pages to sitemap
  userGenPageUrls.forEach((url) => {
    sitemapStream.write({ url })
  })

  sitemapStream.end()

  // ...
}
```

Afterwards we write all the URLs to the sitemap stream. In this code, we're only writing the `url` property for each sitemap entry. We can also specify the `changefreq` and `priority` of each entry as well, but Google ignores those values so I don't bother. Once we're done writing, we end the stream.

```js
const sitemapApi = async (req, res) => {
  // ensure response is XML & gzip encoded

  // get URLS

  // create sitemap stream

  // write pages to sitemap

  // stream write the response
  pipeline.pipe(res).on('error', (err) => {
    throw err
  })
}
```

Lastly, we wrap things up by streaming the sitemap through the response. Streaming means that we don't have to generate the whole sitemap and keep it in a string before returning it in the response. We can stream the output as it's generated. This is super helpful for large sitemaps so that we don't have to keep the whole thing in memory.

---

Hopefully that helps! I had to piece this solution together from a couple of other blog posts, the `sitemap` docs, and the Next.js docs. My pain is now your gain. 😅 Feel free to reach out to me on Twitter at [@benmvp](https://twitter.com/benmvp) with any comments, questions, or suggestions you have!

Keep learning my friends. 🤓
