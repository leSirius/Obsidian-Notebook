```toc
```
# Chapter 3. **Optimizing Fonts and Images**

[Cumulative Layout shift](https://web.dev/cls/). Next.js automatically optimizes fonts in the application when you use the `next/font` module.
## `<Image>` features:

- Preventing layout shift automatically when images are loading.
- Resizing images to avoid shipping large images to devices with a smaller viewport.
- Lazy loading images by default (images load as they enter the viewport).
- Serving images in modern formats, like [WebP](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types#webp) and [AVIF](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types#avif_image), when the browser supports it.

# Chapter 4. Creating Layouts and Pages

Next.js uses file-system routing where **folders** are used to create nested routes. Each folder represents a **route segment** that maps to a **URL segment. Folder name can use ( ) and [ ].**
## Structure (APP Route):

![[structure1.png|502]]
![[structure2 1.png|501]]
![[structure3.png|505]]
![[file-sturcture4.jpg|502]]
# Chapter 5. Navigating Between Pages

## `<Link>` features

To improve the navigation experience, Next.js automatically code splits your application by route segments. This is different from a traditional React [SPA](https://developer.mozilla.org/en-US/docs/Glossary/SPA), where the browser loads all your application code on initial load.

Splitting code by routes means that pages become isolated. If a certain page throws an error, the rest of the application will still work.

Furthermore, in production, whenever [`<Link>`](https://nextjs.org/docs/pages/api-reference/components/link)  components appear in the browser's viewport, Next.js automatically **prefetches** the code for the linked route in the background. By the time the user clicks the link, the code for the destination page will already be loaded in the background, and this is what makes the page transition near-instant!

Only the shared layout, down the rendered "tree" of components until the first `loading.js` file, is prefetched and cached for `30s`.

Further details in [how navigation works](https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating#how-routing-and-navigation-works). (important)
## [`usePathname()`](https://nextjs.org/docs/app/api-reference/functions/use-pathname) hook

As a hook, can only be used in a client component, noted by `"use client"`, to get user’s current path from the URL.

# Chapter 7. Fetching Data

## [Route Handlers](https://nextjs.org/docs/app/api-reference/file-conventions/route) (API layer)

Route Handlers are defined in `rout.js | ts` file in side the `app` directory, similar to `page.js` and `layout.js`. But there cannot be a route.js file at the same route segment level as `page.js`.
Each `route.js` or `page.js` file takes over all HTTP verbs for that route.
### Caching
![[attachments/4.png]]

Route handlers are cached by default when using `GET` method with `Response`. Revalidation of cached data can be set using `next.revalidate` option.
opting out of caching:
- Using the `Request` object with the `GET` method
- Using any of the other HTTP methods
- Using Dynamic Functions like `cookies` and `headers`
- The Segment Config Options manually specifies dynamic mode
## Server Component to fetch data
### a few benefits:
- Server Components support promises, providing a simpler solution for asynchronous tasks like data fetching. You can use `async/await` syntax without reaching out for `useEffect`, `useState` or data fetching libraries.
- Server Components execute on the Server, so you can keep expensive data fetches and logic on the server and only send the result to the client.
- As mentioned before, since Server Components execute on the server, you can query the database directly without an additional API layer.
### Sample
```
export async function fetchLatestInvoices (...)= {
...
	const data = await sql<LatestInvoiceRaw>`SELECT invoices.amount, customers.name, customers.image_url, customers.email FROM invoices JOIN customers ON invoices.customer_id = customers.id ORDER BY invoices.date DESC LIMIT 5`;
...
	return data
}

// in other files (Server Component)
const latestInvoices = await fetchLatestInvoices();
```
### possible points needing optimization
1. The data requests are unintentionally blocking each other, creating a **request waterfall**.
2. By default, Next.js **prerenders** routes to improve performance, this is called **Static Rendering**. So if your data changes, it won't be reflected in your dashboard.
#### Request waterfall
A "waterfall" refers to a sequence of network requests that depend on the completion of previous requests. In the case of data fetching, each request can only begin once the previous request has returned data.
![[attachments/Pasted image 20240427013436.png]]
##### Solution — Parallel data fetch
`Promise.all()` and `Promise.allsettled()` functions can be used to to initiate all promises at the same time.
```
const data = await Promise.all([
	invoiceCountPromise, customerCountPromise, invoiceStatusPromise,
]);
```
By using this pattern, you can:
- Start executing all data fetches at the same time, which can lead to performance gains.
- Use a native JavaScript pattern that can be applied to any library or framework. 
However, if there is a data request slower than others, all the other promise objects have to wait for it.

# Course 8. Static and Dynamic Rendering
## Static Rendering 
With static rendering, data fetching and rendering happens on the server at build time (when you deploy) or during [revalidation](https://nextjs.org/docs/app/building-your-application/data-fetching/fetching-caching-and-revalidating#revalidating-data). The result can then be distributed and cached in a [Content Delivery Network (CDN)](https://nextjs.org/docs/app/building-your-application/rendering/server-components#static-rendering-default).
![[attachments/Pasted image 20240427022343.png]]
Whenever a user visits your application, the cached result is served. There are a couple of benefits of static rendering:
- **Faster Websites** - Prerendered content can be cached and globally distributed. This ensures that users around the world can access your website's content more quickly and reliably.
- **Reduced Server Load** - Because the content is cached, your server does not have to dynamically generate content for each user request.
- **SEO** - Prerendered content is easier for search engine crawlers to index, as the content is already available when the page loads. This can lead to improved search engine rankings.

Static rendering is useful for UI with **no data** or **data that is shared across users**, such as a static blog post or a product page. It might not be a good fit for a dashboard that has personalized data that is regularly updated.

## Dynamic Rendering
With dynamic rendering, content is rendered on the server for each user at **request time** (when the user visits the page). There are a couple of benefits of dynamic rendering:
- **Real-Time Data** - Dynamic rendering allows your application to display real-time or frequently updated data. This is ideal for applications where data changes often.
- **User-Specific Content** - It's easier to serve personalized content, such as dashboards or user profiles, and update the data based on user interaction.
- **Request Time Information** - Dynamic rendering allows you to access information that can only be known at request time, such as cookies or the URL search parameters.

Making Code Dynamic
You can use a Next.js API called `unstable_noStore` inside your Server Components or data fetching functions to opt out of static rendering. Let's add this.
```
import { unstable_noStore as noStore } from 'next/cache';
export async function fetchRevenue() { 
	// Add noStore() here to prevent the response from being cached. 
	// This is equivalent to in fetch(..., {cache: 'no-store'}). 
	noStore();
	// ...
}
```
>**Note:** `unstable_noStore` is an experimental API and may change in the future. If you prefer to use a stable API in your own projects, you can also use the [Segment Config Option](https://nextjs.org/docs/app/api-reference/file-conventions/route-segment-config) `export const dynamic = "force-dynamic"`.

# Course 9. [Streaming](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming#streaming-with-suspense) 
More detailed explanation can be found in the above link. 
Streaming is a data transfer technique that allows you to break down a route into smaller "chunks" and progressively stream them from the server to the client as they become ready. (React components can be easily considered to be chunks.)
By streaming, you can prevent slow data requests from blocking your whole page. This allows the user to see and interact with parts of the page without waiting for all the data to load before any UI can be shown to the user.
From
![[attachments/server-rendering-without-streaming.avif]]![[attachments/server-rendering-without-streaming-chart.avif]]
To
![[attachments/server-rendering-with-streaming.avif]]![[attachments/server-rendering-with-streaming-chart.avif]]

There are two ways you implement streaming in Next.js:
1. At the page level, with the `loading.tsx` file.
2. For specific components, with `<Suspense>`.
```
import { Suspense } from 'react';
//...
<Suspense fallback={<RevenueChartSkeleton />}> 
	<RevenueChart /> 
</Suspense>
```

# Chapter 11. Search and Pagination
Search functionality in the code will span the client and the server. When a user searches for an invoice on the client, the **URL params** will be updated, data will be fetched on the server, and the table will re-render on the server with the new data. (Different with using client side state.)

## Search with URL
A few benefits:
- **Bookmarkable and Shareable URLs**: Since the search parameters are in the URL, users can bookmark the current state of the application, including their search queries and filters, for future reference or sharing.
- **Server-Side Rendering and Initial Load**: URL parameters can be directly consumed on the server to render the initial state, making it easier to handle server rendering.
- **Analytics and Tracking**: Having search queries and filters directly in the URL makes it easier to track user behavior without requiring additional client-side logic.
```
// Handler function
function handleSearch(term: string) { 
	const params = new URLSearchParams(searchParams); 
	if (term) { 
		params.set('query', term); 
	} else {
		params.delete('query'); 
	} 
	replace(`${pathname}?${params.toString()}`); }

// Keeping the URL and input in sync.
// Saving search query in URL usually using defaultValue, 
// while using React state may need value attribute.
<input 
	className="peer block w-full rounded-md border border-gray-200 py-[9px] pl-10 text-sm outline-2 
	placeholder:text-gray-500" placeholder={placeholder} 
	onChange={(e) => { handleSearch(e.target.value); }} 
	defaultValue={searchParams.get('query')?.toString()}
/>

// Updating the table. 
// Notice again, this method doesn't use state, 
// so after inputing query and replace function to a new URL, 
// page needs to update the table.
const query = searchParams?.query || ''; 
const currentPage = Number(searchParams?.page) || 1;
//...
<Suspense key={query + currentPage} fallback={<InvoicesTableSkeleton />}> 
	<Table query={query} currentPage={currentPage} /> 
</Suspense>
```
>**When to use the `useSearchParams()` hook vs. the `searchParams` prop?**
>You might have noticed you used two different ways to extract search params. Whether you use one or the other depends on whether you're working on the client or the server.
> - `<Search>` is a Client Component, so you used the `useSearchParams()` hook to access the params from the client.
> - `<Table>` is a Server Component that fetches its own data, so you can pass the `searchParams` prop from the page to the component.
As a general rule, if you want to read the params from the client, use the `useSearchParams()` hook as this avoids having to go back to the server.

## Debouncing
**Debouncing** is a programming practice that limits the rate at which a function can fire. In our case, you only want to query the database when the user has stopped typing.

> **How Debouncing Works:**
> 1. **Trigger Event**: When an event that should be debounced (like a keystroke in the search box) occurs, a timer starts.
> 2. **Wait**: If a new event occurs before the timer expires, the timer is reset.
> 3. **Execution**: If the timer reaches the end of its countdown, the debounced function is executed.

```
import { useDebouncedCallback } from 'use-debounce';
const handleSearch = useDebouncedCallback(callback, time);
```

## Pagination 

Navigate to the `<Pagination/>` component and you'll notice that it's a Client Component. You don't want to fetch data on the client as this would expose your database secrets (remember, you're not using an API layer). Instead, you can fetch the data on the server, and pass it to the component as a prop.

Better to read code to learn about this part

# Course 12 Mutating data
[`'use server'`](https://react.dev/reference/rsc/use-server) marks server-side functions that can be called from client-side code. Add `'use server'` at the top of an async function body to mark the function as callable by the client. We call these functions _Server Actions_.
`'use client'` lets you mark what code runs on the client.

# Course 13 Handling errors
- `try/catch` Note how `redirect` is being called outside of the `try/catch` block. This is because `redirect` works by throwing an error, which would be caught by the `catch` block.
- `error.tsx` serves as a **catch-all** for unexpected errors and allows you to display a fallback UI to your users.
- `notFound() and not-found.tsx` 

# Course 14 Accessibility
Accessibility refers to designing and implementing web applications that everyone can use, including those with disabilities. It's a vast topic that covers many areas, such as keyboard navigation, semantic HTML, images, colors, videos, etc.

## Form validation
**Client-Side validation**: `required` attribute.
```
<input 
	id="amount" 
	name="amount" 
	type="number" 
	placeholder="Enter USD amount" 
	className="peer block w-full rounded-md border border-gray-200 py-2 pl-10 text-sm outline-2 placeholder:text-gray-500" 
	required
/>
```
