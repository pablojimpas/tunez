> [!NOTE]
> **OpenCollection Showcase Fork** — This `opencollection-showcase` branch is a fork of [sevenseacat/tunez](https://github.com/sevenseacat/tunez) (`end-of-chapter-10`) demonstrating [ash_opencollection](https://github.com/pablojimpas/ash_opencollection) and [open_collection](https://github.com/pablojimpas/open_collection) with JSON:API and GraphQL requests (see `collections/tunez/` and [`documentation/tutorials/tunez-walkthrough.md`](documentation/tutorials/tunez-walkthrough.md)).

# Tunez

The starter app for the upcoming [Ash Framework](https://pragprog.com/titles/ldash/ash-framework/) book.

## Setup

The versions of Elixir and Erlang we're using are specified in the `.tool-versions` file. Tunez should work with any reasonably recent versions, but newer is better!

* If you're using `asdf` to manage installed versions of languages, run `asdf install` to install them. 
* If you're using `mise` to manage installed versions of languages, run `mise install` to install them. 

Once you have those installed:

* Run `mix setup` to install and setup Elixir dependencies
* Start Phoenix endpoint with `mix phx.server` or inside IEx with `iex -S mix phx.server`

Now you can visit [`localhost:4000`](http://localhost:4000) from your browser.
