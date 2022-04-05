# iced RFCs

If you want to contribute a significant change to [`iced`], you may be asked to write an RFC.


## What is an RFC?

In short, an RFC (Request For Comments) is a written document explaining the design and rationale of a specific feature or a set of changes.

RFCs are intended to provide a consistent and controlled path for new features to be added to the library while understanding their impact in the evolution of the library.


## When do I need to write an RFC?

You need to write an RFC if you intend to make significant changes to [`iced`], the website, the book, or the RFC process itself. In general, a change is considered significant when it creates, removes, or impacts established ideas or APIs in the library.

On the other hand, small changes (bugfixes, small tweaks, documentation, etc.) do not generally require an RFC.

If you are unsure whether your changes are small or require an RFC, please create a discussion in the [`iced`] repository.

If you submit a pull request to implement a new feature without going through the RFC process, it may be closed with a polite request to submit an RFC first.

[`iced`]: https://github.com/iced-rs/iced


## How do I create an RFC?

Let's say you want to write an RFC about a new feature called: `my_feature`. The process is quite straightforward.

1. Fork this repository.
1. Create a new `my_feature` branch for your new RFC.
1. Copy the `0000-template.md` to `text/0000-my_feature.md`.
1. Fill in the RFC.
1. Submit a pull request in this repository.
1. Replace the `0000` in the filename `0000-my_feature.md` with the number of your PR.
1. Wait for a review of the core team and iterate the design until consensus is reached.
1. If the PR is...
    - __merged__, then contributors can start working on the implementation and create a PR in the [`iced`] repository.
    - __closed__, then the design was dismissed in its current state because consensus was not reached.

