# SvelteKit Tailwind website

## Summary

This proposal introduces using [SvelteKit](https://kit.svelte.dev/) and [Tailwind](https://tailwindcss.com/) for the iced.rs website. An implementation of which is hosted at https://iced-website.pages.dev/.

## Motivation

The current Website system, which consists of Bulma and Zola, allows for bad practices that degrade the user experience. For example, the current website does not define the width and height of images, does not define the alt or href of a link, and uses a deprecated API.

Also, Bulma and Zola are not performant. Bulma results in a large amount of unused CSS, and Zola bundles FontAwesome into a single javascript file and Bulma into a single CSS file which results in .56 seconds of render-blocking resources.

Tailwind and SvelteKit will provide a better developer experience and a more performant application.

## Guide-level explanation

### Tailwind

Tailwind provides a JIT-compiled utility class library, allowing fewer lines of code and helpful guidelines to follow while maintaining the control of pure CSS and small bundle size. For example, when the developer types

```html
<div class="flex"></div>
```

Tailwind will automatically generate

```css
.flex {
  display: flex;
}
```

Tailwind also provides media and state queries, like hover, dark mode, and screen size. Some examples are

```html
<div class="hover:text-sky-400"></div>

<div class="dark:text-slate-200"></div>

<div class="lg:hidden"></div>

<div class="dark:hover:bg-slate-700"></div>
```

Which respectively generate

```css
.hover\:text-sky-400:hover {
  --tw-text-opacity: 1;
  color: rgb(56 189 248 / var(--tw-text-opacity));
}

@media (prefers-color-scheme: dark) {
  .dark\:text-slate-200 {
    --tw-text-opacity: 1;
    color: rgb(226 232 240 / var(--tw-text-opacity));
  }
}

@media (min-width: 1024px) {
  .lg\:hidden {
    display: none;
  }
}

@media (prefers-color-scheme: dark) {
  .dark\:hover\:bg-slate-700:hover {
    --tw-bg-opacity: 1;
    background-color: rgb(51 65 85 / var(--tw-bg-opacity));
  }
}
```

Tailwind's guidelines can also be overridden with custom values. For example

```html
<div class="bg-[#50d71e]"></div>
```

will generate

```css
.bg-\[\#50d71e\] {
  --tw-bg-opacity: 1;
  background-color: rgb(80 215 30 / var(--tw-bg-opacity));
}
```

This overall allows for faster development, cleaner code, the control of pure CSS, and a minimal CSS bundle.

### SvelteKit

SvelteKit includes many different components that each improve the DX:

- [Svelte](https://svelte.dev/) is a powerful and lightweight frontend framework (more details below).
- [Vite](https://vitejs.dev/) is frontend tooling with features like Hot Module Reload and optimized builds. Hot module Reload means that Vite can reload just a part of an application's javascript. This allows Vite to automatically load changes without reloading or losing the application's state.
- [TypeScript](https://www.typescriptlang.org/) is a type-safe version of JavaScript. The type declarations also lead to better IDE tooling like autocomplete and better static analysis.
- [ESLint](https://eslint.org/) is a static analysis tool. It helps prevent the bad practices in the original website.
- [Prettier](https://prettier.io/) is a code formatting tool.
- A client-side router to make site navigation more responsive.
- Rendering options such as Prerendering, SSR, and CSR. Prerendering or SSR allows for better SEO and faster page loads.

The most influential part of SvelteKit is Svelte as a frontend framework. Svelte is a language that is like a reactive version of javascript and transpiles into vanilla javascript. For example, a counter in Svelte looks like

```html
<script>
  let count = 0;

  function handleClick() {
    count += 1;
  }
</script>

<button on:click="{handleClick}">
  Clicked {count} {count === 1 ? 'time' : 'times'}
</button>
```

and transpiles into 56 lines of vanilla javascript. As a transpiler, it has a much simpler syntax than DOM libraries and is lightweight and fast because it ships no unnecessary javascript.

As a result, the implantation at https://iced-website.pages.dev/ has a fast development time, fantastic LightScore score, and a (subjectively) solid UI.

## Implementation strategy

The code for the implementation is at https://github.com/AlistairKeiller/iced-website.

The main code is at /src/routes/+layout.svelte and /src/routes/+page.svelte, while the rest of the project is mostly configuration and assets. +layout is served at the top of every page and currently contains the navigation bar, while +page.svelte is served as the index.html and is the main page's content. Most of the code is just Tailwind/HTML, with some Svelte for the dropdown navbar that appears on mobile.

I only have two noteworthy remarks about the current implementation. First, the SVG icons are manually inlined from Google Material Symbols for performance and flexibility at the cost of long lines of code. Second, the code example is manually syntax highlighted for performance and accuracy, at the cost of requiring a lot of Tailwind/HTML.

The plan for this implementation would be to finish up the main page by adding suggestions in this RFC, then add the other pages discussed in the [GitHub conversation](https://github.com/iced-rs/iced-rs.github.io/issues/3), like a Demo page, a Quickstart page, and a Contributors page.

The only edge case I can think of is whether the iced demo page would work with SvelteKit. If iced_web does not play nice with SvelteKit, for some reason, we can host the demo at a subdomain with a system that does play nice with iced.

## Drawbacks

SvelteKit has not hit 1.0; it is in the release candidate phase. There are "no more planned breaking API changes," but according to SvelteKit's GitHub page, "expect bugs." Nevertheless, I have found it to be incredibly stable.

Svelte and SvelteKit are both relatively new, so developers may not be familiar with them. However, Svelte's syntax is very similar to HTML and JavaScript, so it should feel approachable and familiar to most web developers.

## Rationale and alternatives

### Web Framework

There are an endless number of alternatives. The major categories of alternatives to SvelteKit are web frameworks in a non-web language, static site generators, and Javascript web frameworks.

#### Web frameworks in a different language

Web frameworks in a non-web language, like [Elm](https://elm-lang.org/), [Ruby on Rails](https://rubyonrails.org/), [Yew](https://yew.rs/), and [Phoenix](https://www.phoenixframework.org/), will probably be less familiar than something closer to JS, HTML, or CSS. Even if that is not the case for something like Elm or Yew, it will certainly not have the tooling or package library of SvelteKit. So, for example, Markdown rendering and Tailwind would be harder to implement.

#### Static Site generators

Static Site generators, like [Hugo](https://gohugo.io/), [Jekyll](https://jekyllrb.com/), and [Zola](https://www.getzola.org/), don't have the control or tooling of a full web framework which has the disadvantages listed in the motivation section above.

#### Javascript web frameworks

These are probably the more compelling alternatives. The Javascript frameworks I compared against are [Angular](https://angular.io/), [Vue](https://vuejs.org/), [NextJS](https://nextjs.org/), [Fresh](https://fresh.deno.dev/), [Qwik](https://qwik.builder.io/), [SolidJS](https://www.solidjs.com/), and [Astro](https://astro.build/) ( with Svelte ).

One of the most significant components of these frameworks is the frontend. We are comparing the frontends Angular, Vue, React, and Svelte. Svelte is a compiler, while the others are Virtual DOMs. This allows Svelte to be the closest to plain Javascript and have the simplest syntax for reactivity. The most significant disadvantage to Svelte is that it is new and, therefore, less popular than the other three. Overall, I think React, Vue, and Svelte will take an iced developer a similar amount of time to get to a good level of productivity, while Angular is probably worse than the other three. But Svelte's simplicity means it providesa better developer experience and higher productivity ceiling. So overall, Svelte as a frontend is better than the alternatives.

However, the React frameworks have some other compelling features. Fresh is based on Deno instead of Node, so it has first-class typescript support, no build step, and is based on the island's architecture for providing javascript only to the components that need it. A disadvantage of being based on Deno is that it is not compatible with npm packages, which leaves much less flexibility. SolidJS has excellent DOM performance. However, that is not a limiting factor for a small website. Qwik lazily loads javascript on the fly to minimize page load times. However, that is at the cost of requiring an async version of React that makes for a worse developer experience. Finally, NextJS has a massive ecosystem. However, that is not a significant advantage for a small and simple project. So none of the React frameworks have benefits that outweigh Svelte's and SvelteKit's benefits.

Finally, Astro uses the island architecture for only providing javascript to components that need it. This would significantly help the iced website's performance, mostly time to interactive. However, time to interactive is not an issue with the SvelteKit website, and Astro with Svelte also has many disadvantages. Astro is not built for Svelte, which shows in the Developer experience; it does not have the same level of tooling for Svelte as SvelteKit. Also, Astro does not provide a client-side router for fast site navigation.

So overall, SvelteKit is the best web framework for building the iced.rs website.

### CSS management solution

The alternative CSS management solutions categories are CSS component libraries, CSS frameworks, CSS in JS, and CSS Utility class libraries.

#### CSS component libraries

A CSS component library for Svelte, like [SvelteMaterialUI](https://sveltematerialui.com/) or [Smelte](https://smeltejs.com/) takes a lot of the work and control away from the developer. They are heavily opinionated, so creating custom designs with them is extremely difficult. I think control over the website is worth more than the time a component library could save.

#### CSS frameworks

A CSS framework like [Bootstrap](https://getbootstrap.com/) or [Bulma](https://bulma.io/) provides utility classes that are very opinionated and prebuilt. This has less design inflexibility than a component library but still makes it difficult to leave their design system and restrictions, which is not ideal. Also, Bootstrap and Bulma are inefficient because they bundle the entire library, including dead code, to the browser.

#### CSS Utility class libraries

A CSS utility class library like Tailwind or Windi is about as high a level as one can go without losing control. They seem intercompatible, and I only choose Tailwind because it is much more popular and has excellent documentation.

Some other alternatives are not mentioned, like SCSS and Stylized Components, but they do not solve the problem of confusing and wordy CSS.

So overall, Tailwind provides the control of pure CSS with a much faster development time and a much better developer experience.

## Unresolved questions

I expect that there will be a lot of discussion about the design language of the website, some implementation details, and possibly the addition of more tools and libraries.
