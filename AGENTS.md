# Atlas — Notes for AI Agents

Atlas is an Emissary template package for map-based, location-tagged posting, installed onto a running Emissary server through a Git package adapter rather than compiled or deployed on its own. See [README.md](README.md) for the product description. Every top-level folder is one template whose folder name must equal the `templateId` inside its `template.hjson`, and the `.html` files beside it are the views its actions render: [atlas-outbox](atlas-outbox/) (the profile page), [atlas-post](atlas-post/) (a located post), [atlas-search](atlas-search/) (the map), and [atlas-theme](atlas-theme/).

This file lives at the repo root and is the only place to add notes. Do not create files inside a template folder.

## A bundle folder holds only source — every file in it is concatenated

`bundles:` in a `template.hjson` or `theme.hjson` declares a folder by name and a content type. Emissary's `populateBundle` then reads **every** file in that folder, skipping only subdirectories — there is no extension filter — concatenates them all, and runs the result through a minifier for that content type. A stray `AGENTS.md`, `README.md`, or editor backup is therefore fed to the CSS or JS minifier as if it were code, corrupting the served bundle. [atlas-theme](atlas-theme/) declares one such bundle, `stylesheet`, as `text/css`.

A theme's `resources/` folder is different — it is served as plain static files, one URL per file, not concatenated. Nothing there corrupts a bundle, but anything you drop in becomes publicly downloadable, so it is still not a place for notes.

## `extends` composes templates, and the base templates live in Emissary

A template inherits actions, states, roles, schema, and datasets from each entry in its `extends` array, and a value defined locally wins over an inherited one. Every base named here — `user-outbox` for the outbox, `base-intent` and `base-social` for posts and search — ships inside Emissary's `_embed/templates`, **not** in this repo, so an unexplained action or role comes from there. Unlike the sibling Bandwagon and Qwertylicious packages, Atlas has no local `-common` template; there is no shared dataset holder to extend.

## Option providers are `{templateId}.{datasetName}`

A form field like `options:{provider:"atlas-post.deleteAfter"}` is split on the dot: Emissary loads the template named by the first segment and looks up the second in its `Datasets` map. Atlas's only local dataset, `deleteAfter`, is declared in [atlas-post/template.hjson](atlas-post/template.hjson) and referenced from that same template, so the two halves match directly here. Were a dataset ever inherited through `extends`, the provider name would still use the *child* template's id, because datasets are copied down at inheritance time. Providers whose names have no dot — `circles`, used here — are built into the server instead.

**A wrong provider name fails silently in the UI.** The lookup reports the error to the server log and returns an empty group, so the form still renders — as an empty dropdown with no visible complaint. When a select is mysteriously blank, check the provider name before anything else.

## hjson does not reject a malformed key — it silently creates a different one

hjson is deliberately lenient, so a typo in a key name is not a parse error; the field simply arrives under the wrong name and whatever read it gets a zero value. A stray quote such as `{Label":"Light Green", Value:"#aad816"}` parses cleanly into a key literally named `Label"`, leaving that entry with no `Label` at all and a blank row in the rendered list — the sibling Qwertylicious package carries exactly that bug today. Nothing warns you. After editing a `template.hjson`, confirm the change actually took effect in the rendered page rather than trusting that it parsed.

## The server caches template folders — restart before concluding an edit did nothing

Emissary loads Git and filesystem template packages into memory at startup. An edit to a `template.hjson` or an `.html` view will not appear until the server reloads that package, so a change that seems to have no effect is usually a stale copy rather than a broken template.

## Pin the htmx and hyperscript resource paths

[atlas-theme/includes-foot.html](atlas-theme/includes-foot.html) resolves to `htmx-1.9.12/htmx.min.js` and `hyperscript-0.9.93/_hyperscript.min.js`, matching Emissary's own default theme. Emissary still ships legacy unversioned `htmx/` and `hyperscript/` folders holding much older engines, and pointing at one of those does not 404 — the old parser simply fails on the shared behaviors bundle and every hyperscript behavior on the page stops installing, silently. On any htmx or hyperscript upgrade, sweep this repo too, not just Emissary's `_embed`.

## An unset hyperscript `:variable` is `undefined`, not `nil`

In hyperscript 0.9.93 the `is nil` / `is not nil` operators compare against `null` only, so on a fresh element `:foo is not nil` is TRUE and an init guard written that way exits immediately and never runs. Use `exists` / `does not exist` for presence checks, or the `no` operator, which covers null, undefined, and empty together.

## `hx-swap="none"` is inherited and silently discards descendants' responses

htmx inherits `hx-swap` down the DOM, so putting `hx-swap="none"` on a container poisons every `hx-boost` or `hx-get` inside it: the request fires, the server answers 200, and htmx throws the response away with no console or server error. When a boosted link "does nothing," check its ancestor chain for an inherited `hx-swap="none"` before suspecting JavaScript. `hx-disinherit="hx-swap hx-push-url"` on the container fixes it while leaving the container's own swap behavior intact.

## `class="turboclick"` on a container is intentional

Emissary's turboclick fires its synthetic click on the element actually pressed, not on the nearest `.turboclick` ancestor, so nested links, buttons, `hx-get`s, and hyperscript `on click` handlers all still fire. Container-level turboclick is correct — don't "fix" it by moving the class onto children. If a click inside a turboclick region does nothing, the cause is almost certainly the `hx-swap` trap above.

## A new funcmap helper couples this repo to a minimum Emissary version

Go's `html/template` resolves function names at **parse** time, so calling a helper that the running server's template funcmap does not define does not degrade that one expression — the entire template fails to parse and the page dies. Template helpers arrive in Emissary through its pinned `benpate/rosetta` dependency, so a template edit that adopts a newly added helper cannot deploy until Emissary itself ships a build carrying it. Check that the helper exists in the target server's build before using it in a template here. The same rule applies to `cssValue`, the validating helper any *computed* style value should pass through instead of the raw `css` cast.

## `Top120` is a date-ORDERED CAP, not a date filter

The `atlas-search` template draws markers from `.Search.Top120.ByCreateDate.Reverse.Slice`, which compiles to `find({location: {$geoWithin: …}}).sort({createDate: -1}).limit(120)`. MongoDB sorts the whole match set and *then* truncates, so once more than 120 results fall inside a viewport the oldest pins silently vanish. It looks exactly like a date cutoff and is not; the diagnostic is that zooming in brings the old pins back.

There is no hidden date filter anywhere — `Common.Search()` builds criteria from `tags`, `date`, and `location` only, and `atlas-search/view.html` never sends `date`, so that filter is inert. For a map or viewport template prefer `.All` (unbounded, but then you need marker clustering) or a larger preset; `ByShuffle` at least truncates without an age bias. Presets live in Emissary's `build/searchBuilder.go` and stop at `Top600`.
