---
title: Foo
---

## Issue 1: `<` on its own

(look at the source for the page here)

{{< copying >}}

---

## Issue 2: conditional tags

{{< foo >}}This should be an em.{{< /foo >}}

<br>

{{< foo "i" >}}This should be an i.{{< /foo >}}

---

## Issue 3: shortcodes with spacing in tags

This is a shortcode with a gap in the tag: {{< baz >}}This should be a cite.{{< /baz >}}

This is the same shortcode without a gap: {{< bar >}}This should be a cite.{{< /bar >}}

**Both work fine. Now let's put them inside another shortcode:**

{{< blockquote >}}
This is a **Markdown** test.

This is an <b>HTML</b> test.

This is a {{< q >}}shortcode with closing shortcode{{< /q >}}.

This is a shortcode with a gap in the tag: {{< baz >}}This should be a cite.{{< /baz >}}

This is the same shortcode without a gap: {{< bar >}}This should be a cite.{{< /bar >}}

Here's the gap one again, but with whitespace removal: {{< zoo >}}This should be a cite.{{< /zoo >}}
{{< /blockquote >}}

**Now let's try a Markdown blockquote:**

> This is a **Markdown** test.
>
> This is an <b>HTML</b> test.
>
> This is a {{< q >}}shortcode with closing shortcode{{< /q >}}.
>
> This is a shortcode with a gap in the tag: {{< baz >}}This should be a cite.{{< /baz >}}
>
> This is the same shortcode without a gap: {{< bar >}}This should be a cite.{{< /bar >}}
>
> Here's the gap one again, but with whitespace removal: {{< zoo >}}This should be a cite.{{< /zoo >}}
