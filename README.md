# Takehome Assignment

## About

In this project, my goal was to create a functional web app for searching for pokemon and displaying the results.

![https://media.tenor.com/EJz4yytoX3EAAAAC/king-of.gif]

## Overview of My Experience and Process

I initially thought about writing this in TypeScript + React but landed on Svelte because it seems to play better with RxJS. I was really excited to utilize RxJS in this project because it really _streamlines_ the search logic (pun intended) and has a lot of built in functionality for what was needed for this assignment.

**The most difficult part of the project** for me was definitely the pagination. I wanted to use the RxJs abstractions as much as possible and avoid stateful side-effect stuff. Fetching the next page to automatically and joining it with the current results definitely broke my brain for a little. Ultimately, I had to look up some similar pagination problems (links in source comments). Once I understood how the `expand` and `scan` operators worked a lightbulb went off. The `expand` operator turned every API request into a bunch of little requests that fetched the next page and `scan` concatenated all the results for us. It was a really amazing learning experience seeing that come together!

Overall, this was a really enjoyable takehome. It was challenging but I felt like I understood the challenges, the solutions to them, and learned some new things! While I didn't finish one of the extensions, I would have been able to if given more time.

### Tech used

- Typescript
  - Coming from Swift on my last project, I knew I wanted to use a language with a stronger type system than JS.
- RxJs
  - This project involved lots of streams of events (typing, network calls, etc) so RxJs felt like a good fit.
- Svelte + SvelteKit
  - I've had a very small amount exposure to Svelte but I knew it supported TS and RxJs.
  - I'm glad I went with it... setup was easy and I learned a lot!
  - To be honest, I don't really know much about SvelteKit but it came with the sevlte project setup.
- CSS
  - Nothing fancy

## Future Considerations

- I didn't have time to add any real tests.
- Code cleanup / refactoring
  - For the most part, everything is in one file. Might be better in multiple.
  - Some logic (constructing API requests, etc) can probably be reused.
- I would love to have added some extra UX goodies like immediately giving the user some feedback even though we are still debouncing the input.
- Make the UI pretty

# Connect with Me

- [GitHub](https://github.com/KATIETOLER/)
- [LinkedIn](https://linkedin.com/in/katie--toler)
