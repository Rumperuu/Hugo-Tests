---
title: Test
---

## Issue 1: `<` Escaped within HTML Comment Block

### Issue

When inside a HTML comment block, `<`s are escaped to `&lt;`. This is against the CommonMark Markdown spec, but I don't think this is a Markdown issue.

### Expected Result

An HTML comment block containing the string `< foo@bar.baz >`.

### Actual Result

Inspect the source immediately below this line.

{{< copying >}}

### Investigation

< foo bar >

Inspect the source for the page above: both the `<` and `>` are rendered as-is (though wrapped in a `<p>`).

The opening `<` in the HTML comment block, however, is escaped as `&lt;`, whilst the closing `>` is left untouched. Per the CommonMark spec, the Markdown parser should be in HTML mode at this point per the [second defined start condition](https://spec.commonmark.org/0.30/#html-blocks) and not be escaping anything until it encounters a `-->`.

**Conclusion:** The Markdown parser is acting contrary to the CommonMark spec.

However, in this demonstrative example, the HTML comment is rendered within a post. However, in my actual site it is rendered [directly into the page template](https://code.bengoldsworthy.net/Rumperuu/Omphaloskepsis-2/src/branch/main/layouts/_default/baseof.html#L3) so should never be parsed as Markdown, but the same `<` escape bug is still happening.

**Conclusion:** This may not be Markdown-related at all, but rather a Go templating bug.

---

## Issue 2: Conditional HTML Tags Escaped

### Issue

When Go template syntax occurs between the opening `<` and the name of an HTML element, the whole HTML tag is escaped and rendered as text.

### Expected Result

> <em>This should be an em.</em>
> <i>This Should be an i.</i>

### Actual Result

> {{< tag-conditional >}}This should be an em.{{< /tag-conditional >}}
> {{< tag-conditional "i" >}}This should be an i.{{< /tag-conditional >}}

### Investigation

The Go template renderer should run over the template first, converting the conditional element render into a string (either `<i...` or `<em...`), which should then be passed to the Markdown parser.

The Markdown parser should recognise these tags as HTML per the [seventh defined start condition](https://spec.commonmark.org/0.30/#html-blocks) from the CommonMark spec.

**Conclusion:** Either the Go template renderer is escaping the opening `<`s because they aren't immediately followed in the template by `[a-z]` (but I wouldn't have expected the Go template renderer to do any HTML parsing like that, rather than being solely focussed on instructions within `{{`/`}}`s), or the Markdown parser is escaping valid HTML for no reason.

If we wrap the above block in `<pre>`s:

<pre>
{{< tag-conditional >}}This should be an em.{{< /tag-conditional >}}
{{< tag-conditional "i" >}}This should be an i.{{< /tag-conditional >}}
</pre>

...and then [copy the output](https://spec.commonmark.org/dingus/?text=%3Cem%3EThis%20should%20be%20an%20em.%3C%2Fem%3E%0A%0A%3Ci%3EThis%20should%20be%20an%20i.%3C%2Fi%3E) into the CommonMark reference implementation, we get the desired result.

**Conclusion:** The issue lies with the Markdown renderer, which is for some reason escaping the HTML entities despite them being spec-compliant.

---

## Issue 3: HTML Elements Containing Whitespace Not Recognised as HTML when Passed to `.Page.RenderString`

### Expected Result

{{< blockquote >}}
This is a **Markdown** test.

This is an <b>HTML</b> test.

This is a {{< q >}}shortcode with closing shortcode{{< /q >}}.

This is a shortcode with a newline before the closing `>`: {{< tag-closed-on-newline-with-whitespace-removal >}}This should be a cite.{{< /tag-closed-on-newline-with-whitespace-removal >}}

This is the same shortcode without a newline: {{< tag-normal >}}This should be a cite.{{< /tag-normal >}}

Here's the one with a newline again, but with Hugo templating whitespace removal: {{< tag-closed-on-newline-with-whitespace-removal >}}This should be a cite.{{< /tag-closed-on-newline-with-whitespace-removal >}}
{{< /blockquote >}}

### Actual Result

{{< blockquote >}}
This is a **Markdown** test.

This is an <b>HTML</b> test.

This is a {{< q >}}shortcode with closing shortcode{{< /q >}}.

This is a shortcode with a newline before the closing `>`: {{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}

This is the same shortcode without a newline: {{< tag-normal >}}This should be a cite.{{< /tag-normal >}}

Here's the one with a newline again, but with Hugo templating whitespace removal: {{< tag-closed-on-newline-with-whitespace-removal >}}This should be a cite.{{< /tag-closed-on-newline-with-whitespace-removal >}}
{{< /blockquote >}}

### Exploration

This happens when using the `{{%/**/%}}` syntax too:

{{% blockquote %}}
This is a **Markdown** test.

This is an <b>HTML</b> test.

This is a {{< q >}}shortcode with closing shortcode{{< /q >}}.

This is a shortcode with a newline before the closing `>`: {{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}

This is the same shortcode without a newline: {{< tag-normal >}}This should be a cite.{{< /tag-normal >}}

Here's the one with a newline again, but with Hugo templating whitespace removal: {{< tag-closed-on-newline-with-whitespace-removal >}}This should be a cite.{{< /tag-closed-on-newline-with-whitespace-removal >}}
{{% /blockquote %}}

This is what happens when the above is written inside a Markdown blockquote (rather than my blockquote shortcode):

> This is a **Markdown** test.
>
> This is an <b>HTML</b> test.
>
> This is a {{< q >}}shortcode with closing shortcode{{< /q >}}.
>
> This is a shortcode with a gap in the tag: {{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}
>
> This is the same shortcode without a gap: {{< tag-normal >}}This should be a cite.{{< /tag-normal >}}
>
> Here's the gap one again, but with whitespace removal: {{< tag-closed-on-newline-with-whitespace-removal >}}This should be a cite.{{< /tag-closed-on-newline-with-whitespace-removal >}}

**Conclusion:** The issue is the shortcode, not the blockquote environment.

Let's try the same shortcodes, but not nested:

> This is a shortcode with a newline before the closing `>`: {{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}
>
> This is the same shortcode without a newline: {{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}

**Conclusion:** The newline doesn't cause a problem when the shortcode isn't nested.

Let's try with a new shortcode that just renders `.Inner | markdownify` and nothing else (which is similar to how the shortcode I was nesting them in originally renders its inner content):

> {{< markdownify >}}{{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}{{< /markdownify >}}
> 
> {{< markdownify >}}{{< tag-normal >}}This should be a cite.{{< /tag-normal >}}{{< /markdownify >}}

**Conclusion:** `markdownify` causes the problem.

[`markdownify`](https://gohugo.io/functions/markdownify/) calls the `.Page.RenderString` method as part of its operation.

Let's try with a new shortcode that just renders `.Inner | .Page.RenderString` and nothing else:

> {{< render-string >}}{{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}{{< /render-string >}}
> 
> {{< render-string >}}{{< tag-normal >}}This should be a cite.{{< /tag-normal >}}{{< /render-string >}}

**Conclusion:** The specific issue is passing raw HTML with a element tag that has a newline before the closing `>` to the `.Page.RenderString` Hugo method.

The same happens when the `display` argument is set to `block`.

Setting the `markup` argument to `html`, `htm`, `md` or `markdown` produces an error:

> Error calling RenderString: no content renderer found for markup "markdown" 

However, by changing the `markup` argument to `org`, the HTML is still escaped but it isn't then mistaken for Markdown blockquote syntax:

> {{< render-string-org-mode >}}{{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}{{< /render-string-org-mode >}}
> 
> {{< render-string-org-mode >}}{{< tag-normal >}}This should be a cite.{{< /tag-normal >}}{{< /render-string-org-mode >}}

We can see here that the resulting HTML `<cite >` (with the newline rendered as a space; if you inspect the inner HTML of the `<blockquote>` element you can still see that the newline is present). Copy-pasting that into the [CommonMark reference implementation](https://spec.commonmark.org/dingus/?text=%3Ccite%20%3Efoo%20bar%3C%2Fcite%3E) produces the expected HTML output, because having whitespace (including newlines) within HTML elements like this is [standards-compliant](https://stackoverflow.com/questions/3314535/is-whitespace-allowed-within-xml-html-tags/3314572#3314572) 

Whilst a [blank line _is_ the terminator](https://spec.commonmark.org/0.30/#html-blocks) for Markdown HTML parsing per the CommonMark spec, this is a) only for certain tag names and b) requires 'a line containing no characters', and we can clearly see that this is not the case here.

**Conclusion:** The issue lies with the Markdown parser/renderer.

Wrapping the initial block inside `<pre>` tags correctly displays the `>`s at the start of most of the lines, but does not stop the inside section from being rendered as a Markdown blockquote:

<pre>
> {{< render-string >}}{{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}{{< /render-string >}}
> 
> {{< render-string >}}{{< tag-normal >}}This should be a cite.{{< /tag-normal >}}{{< /render-string >}}
</pre>

**Conclusion:** The Markdown parser/renderer is very confused, as `<pre>` should be the [highest-priority start condition](https://spec.commonmark.org/0.30/#html-blocks) for a HTML block per the CommonMark spec, and the end condition should be the `</pre>` and nothing before it.

We can wrap the org-mode-rendered version in a pair of `<pre>` elements:

<pre>
> {{< render-string-org-mode >}}{{< tag-closed-on-newline >}}This should be a cite.{{< /tag-closed-on-newline >}}{{< /render-string-org-mode >}}
> 
> {{< render-string-org-mode >}}{{< tag-normal >}}This should be a cite.{{< /tag-normal >}}{{< /render-string-org-mode >}}
</pre>

If we [copy-paste the content of this `<pre>`](https://spec.commonmark.org/dingus/?text=%3E%20%3Ccite%0A%3EThis%20should%20be%20a%20cite.%3C%2Fcite%3E%0A%3E%20%0A%3E%20%3Ccite%3EThis%20should%20be%20a%20cite.%3C%2Fcite%3E) into the CommonMark reference implementation, we get a different rendering than the one seen in the previous step.

**Conclusion:** The issue is specific to the Markdown parser/renderer, which seems to go off-spec when parsing here.

We can pipe the above org-mode-generated string directly into `.Page.RenderString`:

{{< render-string-direct >}}

...which results in the same output as the reference implementation, and not the messy nested blockquote that we see in the examples detailed here.

**Conclusion:** The content, rendered via org-mode and then passed to `.Page.RenderString`, renders correctly per the reference implementation. Passing it directly to the Markdown renderer, however, completely breaks its parsing of HTML, to the point of breaking out of even a surrounding `<pre>` environment.

You can [add a newline inside an HTML tag](https://spec.commonmark.org/dingus/?text=%3E%20%3Ccite%20%0Astyle%0A%3D%0A%22%0Acolor%0A%3A%0Ablue%0A%3B%0A%22%3EThis%20should%20be%20a%20cite.%3C%2Fcite%3E%0A%0A%3E%20%3C%0Acite%20%0Astyle%0A%3D%0A%22%0Acolor%0A%3A%0Ablue%0A%3B%0A%22%3EThis%20should%20be%20a%20cite.%3C%2Fcite%3E%0A%0A%3E%20%3Ccite%20%0Astyle%0A%3D%0A%22%0Acolor%0A%3A%0Ablue%0A%3B%0A%22%0A%3EThis%20should%20be%20a%20cite.%3C%2Fcite%3E%0A%0A%3E%20%3Ccite%20style%3D%22color%3Ablue%3B%22%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%20%3EThis%20should%20be%20a%20cite.%3C%2Fcite%3E) at any point and Markdown will still parse them correctly, UNLESS the newline is immediately after the opening `<` or before the closing `>`. That [makes sense](https://spec.commonmark.org/0.30/#example-153) given the [seventh start condition](https://spec.commonmark.org/0.30/#html-blocks) from the spec, which requires that the 'line begins with a complete open tag'.

[This blog post](https://oostens.me/posts/hugo-shortcodes-with-markdown-gotchas/#solution) suggests that one shouldn't trim whitespace around the `.Inner` if using `{{%/**/%}}`, but that doesn't seem to have had any effect on the blockquote shortcode.
