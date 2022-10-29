# Iced website outline

## Summary

The goal of this RFC is to make a decision on which technology is best for Iced's website based on the goals and requirements for the website. This RFC details every requirement for the Iced website so that once we agree on the requirements, we can decide on the best technology for the website.

## Requirements

- The website should support i18n
- The website should support a11y
- The website should support dark mode
  - The dark mode should default to the user's system preference and be overridable by the user
- The website should be navigable
  - Every action a user could reasonably take, e.g., going from the landing page to the contributor's page or from one guide to another, should be easily and clearly accessible within one click
- The guide should be searchable
  - The search should automatically index the guide
  - The search bar should suggest results interactively as the user types
  - The search bar should be accessible from any page
- The website should have a landing page
  - The landing page should explain what Iced is to someone who has never heard of it
  - The landing page should explain the reasons someone should use Iced
- The website should have a guide
  - The guide should be readable for someone with little experience with Rust and no experience with Iced
  - The guide should walk through the creation of some Iced examples
  - The guide should go through Iced's architecture and core concepts
  - The guide should be written in Markdown
    - The Markdown's code blocks should be rendered with syntax highlighting and a copy button
    - Headings should be rendered with a clickable item next to them that links to their corresponding anchor
    - The headings in a given page of Markdown should be displayed in a "On this page" sidebar
      - The headings in the "On this page" section should be clickable to jump to the corresponding section
      - The headings in the "On this page sections" should be highlighted to indicate the current section
- The website should have a showcase page
  - The showcase page should display all the projects using Iced
  - The showcase page should display Iced's examples
  - Each example should be shown with a image or gif, and a link to the project's source code
- The website should have a contributors page
  - The contributors page should display the GitHub profile all the people who have contributed to Iced in the order of GitHub's contribution ranking
- The website should have a roadmap page
- The website should have a 404 page
- The website should be free to host
- The website should be accessible to anyone from the domain iced.rs

## Goals

These are like requirements, but they do not have a clear yes or no answer.

- The website should be responsive
  - The website should be mobile friendly
  - The website should be high DPI friendly
- The website should be performant
- The website should be easy to maintain
  - The website should minimize the codebase
    - The website should not rewrite and maintain a tool if another developer is already maintaining a tool that does the same thing
- The website should have accessible development
  - The website's development environment should be easy to install and run
  - The website should be written in with tools that an average developer could quickly use, either by being familiar, easy to learn or a mix of both
- The website should have a good DX
  - Changes to the code should interactively update in the browser
  - Features should be fast to write
- The website should have good SEO
- The website should look good ( subjective )
- The website should look consistent across pages ( subjective )
  - There should be a consistent color scheme across the website
  - As many components as possible should be reused across pages, like a navigation bar and search bar

## Nice to have

These are here to decide between tools if all else is equal.

- The website's software stack should be open-source
- The website's software stack should be built with Rust
