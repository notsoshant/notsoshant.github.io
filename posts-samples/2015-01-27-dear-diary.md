---
layout: post
title: Dear diary
---

What is it with that Mary girl?  Dragging me to school every day. As if I had a choice.  What you don't hear in those nursery rhymes is that she starves me if I don't go to school with her; it's the only way I can stay alive!  I'm thinking about being adopted by Little Bo Peep, sure I may get lost, but anything is better than being with Mary and those little brats at school (shudder, shudder).

```html
<div id="full-tags-list">
    <h2 id="{{- tag -}}" class="linked-section">
        <i class="fa fa-tag" aria-hidden="true"></i>
    </h2>
    <div class="post-list">
            <div class="tag-entry">
                <a href="{{ post.url | relative_url }}"></a>
                <div class="entry-date">
                    <time datetime="{{- post.date | date_to_xmlschema -}}"></time>
                </div>
            </div>
    </div>
</div>
```
