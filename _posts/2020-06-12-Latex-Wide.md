---
toc: false
layout: post
description: Pro tip - Resize it
categories: [LaTeX, table]
title: How to make a really wide table in LaTeX
---

> ℹ️ Click this [link](https://gist.github.com/kevin-deyoungster/3e716b8eba5962a4f46d369b4d6b3036) to get a template LaTeX document with the different tables layouts in this post.

I was working on a literature review for my research class back in college. I needed to make a really wide table to summarize my results in the [ACM LaTeX template](https://www.overleaf.com/LaTeX/templates/association-for-computing-machinery-acm-large-2-column-format-template/qwcgpbmkkvpq). Was I using a single-column LaTeX document, it would’ve been straight forward, but this was a double-column document which brought a challenge.

Chances are you’d want to create a wide table in a double-column LaTeX document, and if you use the [regular](https://www.overleaf.com/learn/LaTeX/tables#Multi-page_tables) code for a table in LaTeX you end up with something like this:

![](https://imgur.com/e2ct547.png "Regular table overflows")

After some googling, you'd find this [good answer on stackoverflow](https://tex.stackexchange.com/questions/89462/page-wide-table-in-two-column-mode) and get this:

![](https://imgur.com/sXgJYAO.png "We get a double-column table alright, but it stilll doesn't fit the page exactly")

When what you really want is something like this:

![](https://imgur.com/1syQl1Q.png "Ahhhhh much better")

To get the really wide table (above) to fit on the page over both columns, you're going to resize it. The only caveat is that your font sizes will be smaller, which isn't too much of a problem.

The final code looks like this:

```LaTeX

% import the adjustbox package at the top of your document
\usepackage{adjustbox}

\begin{table*}[ht]
    \caption{Table Caption}
    \centering
    % wrap the tabular element in a resizebox 
    \resizebox{\linewidth}{!}{
        \begin{tabular}{ |l|l|l|l|l|l|l|l| } 
            % 
            % table contents go in here
            % 
        \end{tabular}
    }
\end{table*}
```

And there you go! Checkout [this link](https://gist.github.com/kevin-deyoungster/3e716b8eba5962a4f46d369b4d6b3036) to get a LaTeX template with all three table types. 