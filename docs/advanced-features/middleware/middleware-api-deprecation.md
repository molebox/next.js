---
description: Learn how to use Middleware in Next.js to run code before a request is completed.
---

# Middleware Deprecations guide

As we actively work on improving Middleware for GA and understanding how you are using them, we have had to make decisions to deprecate some APIs and in some cases, the way you define middleware.

This document is intended to detail such deprecations along with their reasoning and an overview of how to take over it.

There are a number of deprecations, but this document is intended to provide transparency on the deprecation and provide steps forward that you can take to ensure continued use of your middleware.

1. [No Nested Middleware](#no-nested-middleware)
2. [No Response Body](#no-response-body)
3. [Cookies API Revamped](#cookies-api-revamped)
4. [No More Page Match Data](#no-mpre-page-match-data)
5. Executing Middleware on Internal Next.js Requests

## No Nested Middleware

One of the most important deprecations is **how** you declare Middleware. Until now you could create a `_middleware.js` file under the `pages` directory at any level. It could be at the very root or nested within a few pages.

The relationship between its declaration and its activation was based on the path where it was created so that we’d call the middleware checking a RegExp generated from such path. For example, one could declare `pages/dashboard/users/_middleware.js` and Next.js would invoke that middleware whenever there was a page that started with `/dashboard/users/*`.

The suggested mental model however was a bit different. Developers tend to create a relationship between sibling pages being rendered and the middleware being invoked.

For example, if colocated with middleware there was `/pages/dashboard/users/[slug].js` then people would tend to think that middleware would be invoked whenever that page was matched but that wasn’t true because middleware could be called for cases where there is a static asset that prevents the page from rendering. For example, if there was an asset that matches for `/dasboard/users/banner.png` it would prevent the dynamic page from rendering and people would easily think that in such match the middleware would not be invoked but there is no way of knowing the match at the phase where middleware is invoked and therefore this assumption is incorrect.

Another problem with allowing defining middleware such way is that it implies that there can be more than one middleware that also can overlap. For example, one can define a middleware in `pages/dashboard/_middleware.js` and another in `pages/dashboard/users/_middleware`. This implies that a request to `/dashboard/users/*` matches _both_ and therefore we need to model how middleware behaves in terms of invocation, order, how are effects dragged around, etc.

### Suggestions for upgrading your app

You should declare **one single middleware file** in the application. It should be located at the root of the project directory (**not** inside of the `pages` directory) and named without an `_`. Middleware will be invoked for **every route in the app** and a custom matcher can define matching filters. An example for a middleware that triggers for `/about/*` would be as follows:

```typescript
// middleware.ts
export function middleware(request) {
  return NextResponse.rewrite(new URL('/about-2', request.url));
}

export config = {
  pathname: '/about(/:path*)'
}
```

## No Response Body

The most common use case for Middleware consists of rewriting and redirecting. Although it is possible to respond with a body it is not very convenient to use if you are rendering HTML. This is _not_ the intended way to use middleware as it, for example, skips the caching layer.

We will introduce other options soon to allow to deploy API functions to the Edge which covers this case.

For this reason we are deprecating sending response bodies in middleware, this ensures that middleware is only used to `rewrite`, `redirect`, or modify the incoming request (e.g. setting cookies). **Middleware will not be able to respond with a body and if it does there will be a runtime error**.

### Suggestions for upgrading your app

For cases where middleware is used to respond (such as authorization), you should migrate it to use rewrites/redirects to pages where they can show authorization errors or login forms, or to an API route.

## Cookies API Revamped

The Cookies API originally included in `NextRequest` and `NextResponse` was defined like it is today by accident, from the deprecated Serverless Loader. This was done thinking Next.js already was implementing it and wanting to make things homogeneous but its API is not very good.

For middleware GA we are changing this API in favour of one that looks more like a get/set map and that allows to easily detach it from request and response. This is mostly a DX decision to improve user experience.

### Suggestions for upgrading your app

## No More Page Match Data

It is not possible to efficiently know ahead of time if an asset will be served instead of a dynamic page and therefore we only do an estimation based on the routes manifest.

Currently, we offer a property `[request.page](http://request.page)` that shows the page that we estimate will render and its match parameters when it is a dynamic page. This information is inaccurate and we are deprecating it for this reason. People tend to think that the information we show there is final and accurate, while it is not.

### Suggestions for upgrading your app

Instead you should use [`URLPattern`](https://developer.mozilla.org/en-US/docs/Web/API/URLSearchParams) to check if a middleware is being invoked for a certain page match. This makes the middleware lighter and more straightforward. The `URLPattern` API will be available once we ship the **edge runtime** to Next.js.

## Executing Middleware on Internal Next.js Requests

We used to avoid executing middleware for `_next` path names as in pre-fetch requests. This leads to security issues when middleware is used for authorization. For example: if you want to protect a SSR page with an auth middleware, a raw request to the data URL that correspond to such page would _not_ execute the middleware.

With this new change, middleware will be executed for all requests, including `_next`. For `/_next/data` request the data URL will be parsed in `NextURL` as if it was a page. This means that if we have a SSR page `/about` then `request.nextUrl.pathname` will be `/about` for both a request to `/about` and a request to `/_next/data/<buildId>/about.json`. This will allow for writing middleware thinking of pages only, there is no distinction between a direct request to the page and the client-side navigation to the page.
