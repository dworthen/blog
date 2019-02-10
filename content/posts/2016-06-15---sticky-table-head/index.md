---
title: "Fixed HTML Table Header"
date: "2016-06-15"
template: "post"
draft: false
slug: "/posts/sticky-table-head/"
category: "JavaScript"
tags:
  - "HTML"
  - "CSS"
  - "JavaScript"
description: "Large HTML tables got you down? Easily fix table headers for readability. Best part. No framework or library necessary."
---

Scrolling down a long list of tabular data without fixed column headings is annoying.

## TLDR

[Skip to the end to view source code and working demo](#demo)

---

Most solutions solutions to the question "how to create [sticky table headers](https://css-tricks.com/scroll-fix-content/)" typically take one of two approaches.

1. Add a fixed position to the thead or first row of the table. This approach is simple and suffices if the table is the first or only element on the page. What I am after is a scroll-then-fixed solution. Here is an [example](https://css-tricks.com/examples/PersistantHeaders/) of what I am after.
2. Use JavaScript to duplicate the first table row and place it in a new table fixed to the top of the page and reveal or hide the duplicated row depending on the viewer's scroll position. This approach feels messy and and its worth mentioning that any event listeners set on the original elements will not work on the duplicated row (unless using [event delegation](https://davidwalsh.name/event-delegate)).

The approach I am after is somewhere in the middle. A scroll-then-fixed solution using the original column headings to maintain event listeners.

#### HTML

```HTML
<table>
  <thead>
    <tr>
      <th>Col 1</th>
      <th>Col 2</th>
      <th>Col 3</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>One</td>
      <td>Two</td>
      <td>Three</td>
    </tr>
    ...
  </tbody>
</table>
```

The HTML is straightforward. Moving on...

#### CSS

```CSS
.fixed {
  position: fixed;
  top: 0;
}
```

Not much to the CSS either. The `.fixed` class will be added to the thead element when a viewer scrolls to the table.

#### JS

```JavaScript
/**
* First Calculate
*   - Where the thead lies on the page
*   - Where the table lies on the page
*   - Table height
**/
var table = document.querySelector('table'),
    thead = table.querySelector('thead'),
    theadOffset = thead.getBoundingClientRect().top,
    tableHeight = +getComputedStyle(table)
                   .getPropertyValue('height')
                   .split('px')[0],
    tableOffset = table.getBoundingClientRect().top,
    tableCells = document.querySelectorAll('th, td');

// Set the widths on all table cells so they dont resize when fixing
// the first row
for (var i = 0, l = tableCells.length; i < l; i++) {
  var cell = tableCells[i];
  cell.width = getComputedStyle(el).getPropertyValue('width');
}

function fixTableHeader(e) {
  // If viewer has scrolled past the first row
  // then fix/stick it to the top of the page
  if( window.pageYOffset > theadOffset ) {
    thead.classList.add('fixed');
  }

  // If viewer has scrolled back above or past the table
  // then unfreeze the first row.
  if ( window.pageYOffset < theadOffset ||
       window.pageYOffset > (tableOffset + tableHeight)) {
      thead.classList.remove('fixed');
  }

}

// Listen...
window.addEventListener('scroll', fixTableHeader, false);
```

## DEMO

<p data-height="265" data-theme-id="dark" data-slug-hash="XKKMBo" data-default-tab="result" data-user="dworthen" data-embed-version="2" data-pen-title="Fixed Table Head" data-preview="true" class="codepen">See the Pen <a href="https://codepen.io/dworthen/pen/XKKMBo/">Fixed Table Head</a> by Derek Worthen (<a href="https://codepen.io/dworthen">@dworthen</a>) on <a href="https://codepen.io">CodePen</a>.</p>
<script async src="https://production-assets.codepen.io/assets/embed/ei.js"></script>
