---
layout: post
title: Media Distribution with Django, Nginx, and Wagtail
author: Neal
categories: [wagtail, django, nginx, media, devops]
image: /assets/img/posts/media.jpg
alt: media
intro: ...distributing content to paying customers, a short albeit rather complex how-to.
---
<div class="row">
    <div class="col-12 col-md-4 my-auto">
        <h4 class="section-heading text-uppercase text-center">By {{ page.author }}</h4>
    </div>
    <div class="col-12 col-md-4 text-center team-member mt-4">
        <img class="img-fluid rounded-circle" src="{{ page.image }}" alt="{{ page.alt }}">
    </div>
    <div class="col-12 col-md-4 my-auto">
        <h4 class="section-heading text-uppercase text-center"> &bull; {{ page.date | date: "%b %-d, %Y" }}</h4>
    </div>
</div>

Want to build a premium media distribution service with Django and Wagtail? Don't want to pay Patreon or Substack 15%-18% of your gross receipts to do it? Here's how...

The assumptions and considerations this was built around:

* User credentials are sent in the request, users do not need to be "logged in" via an app or framework. In this post's example, we will consider subscriber media distributed as a podcast RSS feed, which the user may want to consume in an app or device that doesn't support authentication. In short we're building a widely-compatible backend here, not a walled garden that requires users to consume the content they've paid for in a specific application.

* The storage backend is a *storage backend*, not intended to be another web server. Rarely mentioned in guides on using object storage services is the fact that in the case of AWS S3 there is a fee per number of requests, and in the case of Digital Ocean as of this document's writing there is a cap on the number of requests to a storage bucket over certain time periods. We will cache and proxy downloads via our own web server with these considerations in mind, the object storage will truly be "just storage."

* Authentication is handled by Django, against Django users in the Django database, which means users need to hit the backend for permissions to get a file. While there are authentication services both standalone and within the offerings of various cloud platforms they can be [quite pricey](https://auth0.com/pricing){:target="_blank" rel="noopener"}, without even getting into the existential question of whether or not you trust third party services with your user data.

* Nginx will serve (and cache) the actual files themselves, but not do any sort of authentication on its own. I don't think spreading authentication functions into multiple services / backends is a good idea in general, so we will begin with the assumption that authentication should live within the framework where the user accounts and credentials live, and that framework's tools should be used to implement authentication.

With all of this in mind, here's what our typical request for a premium media file will look like:

<div class="text-center"><svg
   version="1.1"
   width="436.00018"
   height="179.27"
   style="background-color:rgb(37,41,47)"
   viewBox="-0.5 -0.5 436.00018 179.27"
   content="&lt;mxfile host=&quot;app.diagrams.net&quot; modified=&quot;2021-10-25T15:58:37.192Z&quot; agent=&quot;5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/15.0 Safari/605.1.15&quot; etag=&quot;Yjc_TJsMqmH5N7kmR1sS&quot; version=&quot;15.5.9&quot;&gt;&lt;diagram id=&quot;JxrAazggplFgWvb1SUAi&quot; name=&quot;Page-1&quot;&gt;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&lt;/diagram&gt;&lt;/mxfile&gt;"
   id="svg1568"
   sodipodi:docname="Untitled.svg"
   inkscape:version="1.1.2 (b8e25be8, 2022-02-05)"
   xmlns:inkscape="http://www.inkscape.org/namespaces/inkscape"
   xmlns:sodipodi="http://sodipodi.sourceforge.net/DTD/sodipodi-0.dtd"
   xmlns:xlink="http://www.w3.org/1999/xlink"
   xmlns="http://www.w3.org/2000/svg"
   xmlns:svg="http://www.w3.org/2000/svg">
  <sodipodi:namedview
     id="namedview1570"
     pagecolor="#25292f"
     bordercolor="#666666"
     borderopacity="1.0"
     inkscape:pageshadow="2"
     inkscape:pageopacity="0"
     inkscape:pagecheckerboard="0"
     showgrid="false"
     inkscape:zoom="2.8387909"
     inkscape:cx="191.10249"
     inkscape:cy="72.389974"
     inkscape:window-width="1296"
     inkscape:window-height="799"
     inkscape:window-x="0"
     inkscape:window-y="25"
     inkscape:window-maximized="0"
     inkscape:current-layer="svg1568"
     width="497px"
     fit-margin-left="20"
     fit-margin-right="20"
     fit-margin-top="20"
     fit-margin-bottom="20" />
  <defs
     id="defs1532" />
  <g
     id="g1566"
     transform="translate(19.500197,20.08)">
    <image
       x="140.5"
       y="-0.57999998"
       width="56"
       height="64"
       xlink:href="data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiB2aWV3Qm94PSIwLjk5OTg4Mzg5MDE1MTk3NzUgMC45OTk3MzQ5MzgxNDQ2ODM4IDU1Ljc3MzkxMDUyMjQ2MDk0IDYzLjk5NjY3NzM5ODY4MTY0IiBmaWxsPSIjZmZmIiBmaWxsLXJ1bGU9ImV2ZW5vZGQiIHN0cm9rZT0iIzAwMCIgc3Ryb2tlLWxpbmVjYXA9InJvdW5kIiBzdHJva2UtbGluZWpvaW49InJvdW5kIiB3aWR0aD0iNTUuNzczOTEwNTIyNDYwOTQiIGhlaWdodD0iNjMuOTk2Njc3Mzk4NjgxNjQiPjx1c2UgeGxpbms6aHJlZj0iI0EiIHg9IjEiIHk9IjEiLz48c3ltYm9sIGlkPSJBIiBvdmVyZmxvdz0idmlzaWJsZSI+PGcgc3Ryb2tlPSJub25lIiBmaWxsLXJ1bGU9Im5vbnplcm8iPjxwYXRoIGQ9Ik0uMDAyIDMyLjA0NlYxNi43NzJhMS4zNiAxLjM2IDAgMCAxIC43Ny0xLjMwMkwyNy4xMTguMjU0Yy40NzQtLjI5NiAxLjAwNi0uMzU2IDEuNDgtLjA2bDI2LjQ2NCAxNS4yNzRhMS40MiAxLjQyIDAgMCAxIC43MSAxLjMwMnYzMC40OWExLjQyIDEuNDIgMCAwIDEtLjcxIDEuMzAybC0yMi43MzQgMTMuMTQtMy42MTIgMi4wNzJhMS41NSAxLjU1IDAgMCAxLTEuNiAwTC43MTIgNDguNTU4Yy0uNDc0LS4yOTYtLjcxLS42NTItLjcxLTEuMjQ0VjMyLjA0eiIgZmlsbD0iIzAwOTQzOCIvPjxwYXRoIGQ9Ik0xOC42NSAyNi4zNnYxNy44YzAgMi4wMTItMS42IDMuNzg4LTMuNzMgMy43My0xLjMtLjA2LTIuMzA4LS41OTItMy0xLjcxNi0uMzU2LS41MzItLjQ3NC0xLjEyNC0uNDc0LTEuNzc2VjE5LjY3MmMwLTEuNjYgMS4wMDYtMi44NCAyLjMwOC0zLjM3NHMyLjYwNC0uNDE0IDMuOTA4IDBjMS4yNDQuMzU2IDIuMTkgMS4xMjQgMyAyLjA3MkwzNi40MSAzNy4yNTZjLjA2LjA2LjEyLjIuMjM2LjI5NnYtMThjMC0xLjg5NCAxLjMtMy4zNzQgMy4xNC0zLjU1MiAyLjMwOC0uMjk2IDMuODQ4IDEuMzYgNC4wODQgMy4wOHYyNS4yYzAgMS40LS42NTIgMi40MjgtMS44MzYgMy4wOC0uODg4LjQ3NC0xLjgzNi41OTItMi44NC41MzJhNi40NiA2LjQ2IDAgMCAxLTMuOTA4LTEuNjU4Yy0uNTkyLS41MzItMS4wMDYtMS4xODQtMS41NC0xLjc3NmwtMTUtMTcuOTRjMC0uMDYtLjA2LS4xMi0uMTItLjJ6IiBmaWxsPSIjZmVmZWZlIi8+PC9nPjwvc3ltYm9sPjwvc3ZnPg=="
       preserveAspectRatio="none"
       id="image1534" />
    <path
       d="m 51,45 69.9,0.44"
       fill="none"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="stroke"
       id="path1536" />
    <path
       d="m 127.65,45.48 -9.03,4.44 2.28,-4.48 -2.23,-4.52 z"
       fill="#ffffff"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="all"
       id="path1538" />
    <image
       x="310.5"
       y="24.5"
       width="84"
       height="36"
       xlink:href="data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI1MDQuMDg5OTk2MzM3ODkwNiIgaGVpZ2h0PSIyMTUuOTk0MDAzMjk1ODk4NDQiIHhtbDpzcGFjZT0icHJlc2VydmUiIGVuYWJsZS1iYWNrZ3JvdW5kPSJuZXcgMCAwIDUwNC4wOSAyMTUuOTk0IiB2ZXJzaW9uPSIxLjAiIHZpZXdCb3g9IjAgMCA1MDQuMDg5OTk2MzM3ODkwNiAyMTUuOTk0MDAzMjk1ODk4NDQiPiYjeGE7JiN4YTsgPGc+JiN4YTsgIDx0aXRsZT5MYXllciAxPC90aXRsZT4mI3hhOyAgPHBhdGggaWQ9InN2Z18xIiBkPSJtNTA0LjA5LDE4Ny45OTRjMCwxNS40NjQgLTEyLjUzNiwyOCAtMjgsMjhsLTQ0OC4wOSwwYy0xNS40NjQsMCAtMjgsLTEyLjUzNiAtMjgsLTI4bDAsLTE1OS45OTRjMCwtMTUuNDY0IDEyLjUzNiwtMjggMjgsLTI4bDQ0OC4wOSwwYzE1LjQ2NCwwIDI4LDEyLjUzNiAyOCwyOGwwLDE1OS45OTR6IiBmaWxsPSIjMDkyRTIwIi8+JiN4YTsgIDxnIGlkPSJzdmdfMiI+JiN4YTsgICA8ZyBpZD0ic3ZnXzMiPiYjeGE7ICAgIDxwYXRoIGlkPSJzdmdfNCIgZD0ibTg2Ljk0NSwzMy45MTlsMjMuODcyLDBsMCwxMTAuNDk2Yy0xMi4yNDYsMi4zMjUgLTIxLjIzNywzLjI1NSAtMzEuMDAyLDMuMjU1Yy0yOS4xNDIsMCAtNDQuMzMzLC0xMy4xNzQgLTQ0LjMzMywtMzguNDQzYzAsLTI0LjMzNiAxNi4xMjIsLTQwLjE0NyA0MS4wNzgsLTQwLjE0N2MzLjg3NSwwIDYuODIsMC4zMTEgMTAuMzg2LDEuMjM5bDAsLTM2LjRsLTAuMDAxLDB6bTAsNTUuNjJjLTIuNzksLTAuOTI5IC01LjExNSwtMS4yMzkgLTguMDYsLTEuMjM5Yy0xMi4wOTEsMCAtMTkuMDY3LDcuNDQxIC0xOS4wNjcsMjAuNDZjMCwxMi43MTMgNi42NjYsMTkuNjg4IDE4LjkxMiwxOS42ODhjMi42MzQsMCA0LjgwNSwtMC4xNTUgOC4yMTUsLTAuNjE4bDAsLTM4LjI5MXoiIGZpbGw9IiNGRkZGRkYiLz4mI3hhOyAgICA8cGF0aCBpZD0ic3ZnXzUiIGQ9Im0xNDguNzkzLDcwLjc4M2wwLDU1LjM0MWMwLDE5LjA2NSAtMS4zOTUsMjguMjEgLTUuNTgsMzYuMTE3Yy0zLjg3Niw3LjU5NiAtOC45OTIsMTIuMzk5IC0xOS41MzIsMTcuNjdsLTIyLjE2NywtMTAuNTQxYzEwLjU0MSwtNC45NiAxNS42NTYsLTkuMjk3IDE4LjkxMSwtMTUuOTY2YzMuNDExLC02LjgxOSA0LjQ5NywtMTQuNzI3IDQuNDk3LC0zNS40OThsMCwtNDcuMTIzbDIzLjg3MSwwem0tMjMuODcxLC0zNi43MzdsMjMuODcxLDBsMCwyNC40OTNsLTIzLjg3MSwwbDAsLTI0LjQ5M3oiIGZpbGw9IiNGRkZGRkYiLz4mI3hhOyAgICA8cGF0aCBpZD0ic3ZnXzYiIGQ9Im0xNjMuMjEyLDc2LjIwOWMxMC41NDIsLTQuOTYxIDIwLjYxNywtNy4xMyAzMS42MjMsLTcuMTNjMTIuMjQ2LDAgMjAuMzA2LDMuMjU1IDIzLjg3Miw5LjYxMWMyLjAxNCwzLjU2NCAyLjYzNCw4LjIxNCAyLjYzNCwxOC4xMzdsMCw0OC41MTdjLTEwLjY5NywxLjU1MiAtMjQuMTgyLDIuNjM2IC0zNC4xMDIsMi42MzZjLTE5Ljk5NiwwIC0yOC45ODgsLTYuOTc3IC0yOC45ODgsLTIyLjQ3NmMwLC0xNi43NDQgMTEuOTM2LC0yNC40OTMgNDEuMjM0LC0yNi45NzVsMCwtNS4yNzFjMCwtNC4zMzkgLTIuMTcsLTUuODg4IC04LjIxNiwtNS44ODhjLTguODM1LDAgLTE4Ljc1NiwyLjQ3OSAtMjguMDU4LDcuMjg1bDAsLTE4LjQ0NmwwLjAwMSwwem0zNy4zNTgsMzcuOTc4Yy0xNS44MTIsMS41NTIgLTIwLjkyNyw0LjAzMSAtMjAuOTI3LDEwLjIzMWMwLDQuNjUgMi45NDYsNi44MjEgOS40NTYsNi44MjFjMy41NjYsMCA2LjgyLC0wLjMxMSAxMS40NzEsLTEuMDg0bDAsLTE1Ljk2OHoiIGZpbGw9IiNGRkZGRkYiLz4mI3hhOyAgICA8cGF0aCBpZD0ic3ZnXzciIGQ9Im0yMzIuOTY4LDc0LjUwNWMxNC4xMDUsLTMuNzIyIDI1LjczMSwtNS40MjYgMzcuNTEyLC01LjQyNmMxMi4yNDYsMCAyMS4wODIsMi43ODggMjYuMzU0LDguMjE2YzQuOTYsNS4xMTMgNi41MDksMTAuNjkzIDYuNTA5LDIyLjYzMmwwLDQ2LjgxM2wtMjMuODcxLDBsMCwtNDUuODg0YzAsLTkuMTQ1IC0zLjEsLTEyLjU1NyAtMTEuNjI1LC0xMi41NTdjLTMuMjU1LDAgLTYuMiwwLjMxMSAtMTEuMDA3LDEuNzA2bDAsNTYuNzM0bC0yMy44NzEsMGwwLC03Mi4yMzRsLTAuMDAxLDB6IiBmaWxsPSIjRkZGRkZGIi8+JiN4YTsgICAgPHBhdGggaWQ9InN2Z184IiBkPSJtMzEyLjYyMywxNTkuNzYxYzguMzcyLDQuMzM5IDE2Ljc0Miw2LjM1NCAyNS41NzcsNi4zNTRjMTUuNjU1LDAgMjIuMzIxLC02LjM1NCAyMi4zMjEsLTIxLjU0NmMwLC0wLjE1NCAwLC0wLjMxIDAsLTAuNDY3Yy00LjY1LDIuMzI2IC05LjMwMSwzLjI1NyAtMTUuNSwzLjI1N2MtMjAuOTI3LDAgLTM0LjI2LC0xMy43OTcgLTM0LjI2LC0zNS42NTJjMCwtMjcuMTI4IDE5LjY4OCwtNDIuNDczIDU0LjU2NCwtNDIuNDczYzEwLjIzMiwwIDE5LjY4OCwxLjA4NCAzMS4xNTksMy40MDdsLTguMTc0LDE3LjIyMmMtNi4zNTYsLTEuMjQxIC0wLjUwOSwtMC4xNjcgLTUuMzEyLC0wLjYzMmwwLDIuNDhsMC4zMDksMTAuMDc0bDAuMTU0LDEzLjAyMmMwLjE1NSwzLjI1MyAwLjE1NSw2LjUxIDAuMzExLDkuNzY0YzAsMi45NDUgMCw0LjM0MiAwLDYuNTEyYzAsMjAuNDYyIC0xLjcwNSwzMC4wNzMgLTYuODIsMzcuOTc3Yy03LjQ0MSwxMS42MjcgLTIwLjMwNywxNy4zNjIgLTM4LjU5OCwxNy4zNjJjLTkuMzAxLDAgLTE3LjM2LC0xLjM5NiAtMjUuNzMyLC00LjY1MWwwLC0yMi4wMWwwLjAwMSwwem00Ny40MzQsLTcxLjMwNmMtMC4zMSwwIC0wLjYxOSwwIC0wLjc3NCwwbC0xLjcwNiwwYy00LjY0OSwtMC4xNTUgLTEwLjA3NCwxLjA4NCAtMTMuNzk2LDMuNDA5Yy01LjczNCwzLjI1NyAtOC42ODEsOS4xNDYgLTguNjgxLDE3LjUxOGMwLDExLjkzNyA1Ljg5MiwxOC43NTYgMTYuNDMyLDE4Ljc1NmMzLjI1NSwwIDUuODkxLC0wLjYyIDguOTksLTEuNTVsMCwtMS43MDVsMCwtNi41MWMwLC0yLjc5IC0wLjE1NCwtNS44OTIgLTAuMTU0LC05LjE0NmwtMC4xNTQsLTExLjAwNmwtMC4xNTYsLTcuOTA1bDAsLTEuODYxbC0wLjAwMSwweiIgZmlsbD0iI0ZGRkZGRiIvPiYjeGE7ICAgIDxwYXRoIGlkPSJzdmdfOSIgZD0ibTQzMy41NDMsNjguNzdjMjMuODcxLDAgMzguNDQzLDE1LjAzNyAzOC40NDMsMzkuMzcxYzAsMjQuOTU3IC0xNS4xOSw0MC42MTMgLTM5LjM3Myw0MC42MTNjLTIzLjg3MywwIC0zOC41OTksLTE1LjAzNiAtMzguNTk5LC0zOS4yMTZjMC4wMDEsLTI1LjExNCAxNS4xOTMsLTQwLjc2OCAzOS41MjksLTQwLjc2OHptLTAuNDY3LDYwLjc2M2M5LjE0NywwIDE0LjU3MywtNy41OTYgMTQuNTczLC0yMC43NzNjMCwtMTMuMDE5IC01LjI3MSwtMjAuNzcxIC0xNC40MTUsLTIwLjc3MWMtOS40NTcsMCAtMTQuODg0LDcuNTk4IC0xNC44ODQsMjAuNzcxYzAuMDAxLDEzLjE3OCA1LjQyNywyMC43NzMgMTQuNzI2LDIwLjc3M3oiIGZpbGw9IiNGRkZGRkYiLz4mI3hhOyAgIDwvZz4mI3hhOyAgPC9nPiYjeGE7IDwvZz4mI3hhOzwvc3ZnPg=="
       preserveAspectRatio="none"
       id="image1540" />
    <rect
       x="311"
       y="25"
       width="84"
       height="36"
       fill="none"
       stroke="#f7f7f7"
       stroke-width="2"
       pointer-events="all"
       id="rect1542" />
    <image
       x="312.5"
       y="75.769997"
       width="80"
       height="62.919998"
       xlink:href="data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIxMDg3LjQxMTAxMDc0MjE4NzUiIGhlaWdodD0iODU1Ljg0MTAwMzQxNzk2ODgiIHZpZXdCb3g9IjEuNTc4MDAwMDY4NjY0NTUwOCAyLjE4NjAwMDEwODcxODg3MiAxMDg3LjQxMTAxMDc0MjE4NzUgODU1Ljg0MTAwMzQxNzk2ODgiPjxwYXRoIGZpbGw9IiNGN0E4MEQiIGQ9Ik0zMTguODM5IDU0Ny43ODVsLTk5LjYyIDQyLjc5MiA5Mi4yNiAzOS40NTUgMTA2Ljk4LTM5LjQ1NS05OS42Mi00Mi43OTJ6bS0xNDkuNzczIDUzLjQ5bC0zLjMzOCAxOTIuNTY0IDE0NS43NSA2NC4xODhWNjU4LjEwNGwtMTQyLjQxMi01Ni44Mjl6bTI5OS41NDUgMGwtMTMxLjcxNSA1MC4xNTJWODM5Ljk3bDEzMS43MTUtNTMuNDlWNjAxLjI3NXpNNjI1Ljc0MyAyLjE4Nkw1MjUuNDM4IDQ0Ljk3OWw5Mi45NDQgMzkuNDU0IDEwNi45OC0zOS40NTQtOTkuNjE5LTQyLjc5M3ptLTEzOS4wNzQgNTYuODVWMjUxLjZsMTI0LjM1NCAzNi4xMTYgNC4wMjItMTc1LjE5MS0xMjguMzc2LTUzLjQ4OXptMjc4LjE0OSAxMC42OTdMNjQ3LjE0IDExOS44ODZ2MTg5LjIyN2wxMTcuNjc5LTUzLjQ5VjY5LjczM3pNMTU0LjY4OCAyNzMuMjFsLTk5LjYyIDQyLjc5MiA5Mi4yNiAzOS40NTUgMTA2Ljk4LTM5LjQ1NS05OS42Mi00Mi43OTJ6TTQuOTE2IDMyNi43TDEuNTc4IDUxOS4yNjVsMTQ1Ljc1IDY0LjE4OFYzODMuNTI4TDQuOTE2IDMyNi43em0yOTkuNTQ1IDBsLTEzMS43MTQgNTAuMTUydjE4OC41NDJsMTMxLjcxNC01My40OVYzMjYuN3ptMTcxLjE2OC02MC41OTRsLTk5LjYyIDQyLjc5MiA5Mi4yNiAzOS40NTUgMTA2Ljk4LTM5LjQ1NS05OS42Mi00Mi43OTJ6bS0xNDkuNzczIDUzLjQ5MWwtMy4zMzggMTkyLjU2NCAxNDUuNzUgNjQuMTg4VjM3Ni40NDZsLTE0Mi40MTItNTYuODQ5em0yOTkuNTQ1IDBsLTEzMS43MTQgNTAuMTUydjE4OC41NDJsMTMxLjcxNC01My40OVYzMTkuNTk3ek05MzkuMjE3IDIuMTg2bC05OS42MTkgNDIuNzkyIDkyLjI2IDM5LjQ1NCAxMDYuOTc5LTM5LjQ1NC05OS42Mi00Mi43OTJ6bS0xNDkuNzczIDUzLjQ5bC0zLjMzNyAxOTIuNTY0IDE0NS43NSA2NC4xODhWMTEyLjUyNUw3ODkuNDQ0IDU1LjY3NnptMjk5LjU0NSAwbC0xMzEuNzE0IDUwLjE1MlYyOTQuMzdsMTMxLjcxNC01My40OVY1NS42NzZ6Ii8+PC9zdmc+"
       preserveAspectRatio="none"
       id="image1544" />
    <path
       d="M 131,95.5 61.1,95.06"
       fill="none"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="stroke"
       id="path1546" />
    <path
       d="m 54.35,95.02 9.03,-4.44 -2.28,4.48 2.23,4.52 z"
       fill="#ffffff"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="all"
       id="path1548" />
    <path
       d="M 1,95 C 1,75 1,65 21,65 7.67,65 7.67,45 21,45 c 13.33,0 13.33,20 0,20 20,0 20,10 20,30 z"
       fill="#eeeeee"
       stroke="#ffffff"
       stroke-width="2"
       stroke-miterlimit="10"
       pointer-events="all"
       id="path1550" />
    <image
       x="147"
       y="74.5"
       width="43"
       height="64.190002"
       xlink:href="data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHhtbG5zOnhsaW5rPSJodHRwOi8vd3d3LnczLm9yZy8xOTk5L3hsaW5rIiB2ZXJzaW9uPSIxLjEiIGlkPSJMYXllcl8xIiB4PSIwcHgiIHk9IjBweCIgdmlld0JveD0iODQuMjU5MDAyNjg1NTQ2ODggMCAzNDMuNDgxOTk0NjI4OTA2MjUgNTExLjk5OTAyMzQzNzUiIHN0eWxlPSJlbmFibGUtYmFja2dyb3VuZDpuZXcgMCAwIDUxMiA1MTI7IiB4bWw6c3BhY2U9InByZXNlcnZlIiB3aWR0aD0iMzQzLjQ4MTk5NDYyODkwNjI1IiBoZWlnaHQ9IjUxMS45OTkwMjM0Mzc1Ij4mI3hhOzxnPiYjeGE7CTxnPiYjeGE7CQk8cGF0aCBkPSJNMjU0LjMwMSw5MC4zNzdjLTI3LjgxOSwwLTUwLjQ1MiwyMi42MzMtNTAuNDUyLDUwLjQ1MnMyMi42MzMsNTAuNDUyLDUwLjQ1Miw1MC40NTJzNTAuNDUyLTIyLjYzMyw1MC40NTItNTAuNDUyJiMxMDsmIzk7JiM5OyYjOTtTMjgyLjEyLDkwLjM3NywyNTQuMzAxLDkwLjM3N3ogTTI1NC4zMDEsMTc1Ljk5MmMtMTkuMzg5LDAtMzUuMTY0LTE1Ljc3NS0zNS4xNjQtMzUuMTY0czE1Ljc3NS0zNS4xNjQsMzUuMTY0LTM1LjE2NCYjMTA7JiM5OyYjOTsmIzk7czM1LjE2NCwxNS43NzUsMzUuMTY0LDM1LjE2NFMyNzMuNjkxLDE3NS45OTIsMjU0LjMwMSwxNzUuOTkyeiIgc3Ryb2tlPSJ3aGl0ZSIvPiYjeGE7CTwvZz4mI3hhOzwvZz4mI3hhOzxnPiYjeGE7CTxnPiYjeGE7CQk8cGF0aCBkPSJNMjI4Ljk3NCwzMjIuODk1Yy00LjAwNS0xMS4wMDMtMTYuMjE1LTE2LjY5MS0yNy4yMTctMTIuNjljLTUuMzMsMS45NC05LjU4NSw1LjgzOS0xMS45ODIsMTAuOTc5JiMxMDsmIzk7JiM5OyYjOTtjLTIuMzk3LDUuMTQxLTIuNjQ5LDEwLjkwNy0wLjcwOCwxNi4yMzZjMS45NCw1LjMzLDUuODM5LDkuNTg1LDEwLjk3OSwxMS45ODJjMi44NTUsMS4zMzIsNS45MDIsMi4wMDIsOC45NjEsMi4wMDImIzEwOyYjOTsmIzk7JiM5O2MyLjQ0OCwwLDQuOTA2LTAuNDMsNy4yNzUtMS4yOTJDMjI3LjI4NSwzNDYuMTA3LDIzMi45NzksMzMzLjg5OCwyMjguOTc0LDMyMi44OTV6IE0yMTEuMDU0LDMzNS43NDUmIzEwOyYjOTsmIzk7JiM5O2MtMS40OSwwLjU0My0zLjEwNiwwLjQ3My00LjU0NS0wLjE5OGMtMS40MzktMC42NzItMi41MzEtMS44NjMtMy4wNzUtMy4zNTZjLTAuNTQ0LTEuNDkzLTAuNDc0LTMuMTA4LDAuMTk4LTQuNTQ3JiMxMDsmIzk7JiM5OyYjOTtjMC42NzEtMS40MzksMS44NjItMi41MywzLjM1NS0zLjA3NGMwLjY2My0wLjI0MiwxLjM1Mi0wLjM2MiwyLjAzNy0wLjM2MmMwLjg1NiwwLDEuNzEsMC4xODksMi41MDksMC41NjEmIzEwOyYjOTsmIzk7JiM5O2MxLjQzOSwwLjY3MSwyLjUzLDEuODYyLDMuMDc0LDMuMzU1QzIxNS43MywzMzEuMjA1LDIxNC4xMzUsMzM0LjYyNCwyMTEuMDU0LDMzNS43NDV6IiBzdHJva2U9IndoaXRlIiBmaWxsPSJ3aGl0ZSIvPiYjeGE7CTwvZz4mI3hhOzwvZz4mI3hhOzxnPiYjeGE7CTxnPiYjeGE7CQk8cGF0aCBkPSJNMzk0LjI3NiwyNjQuMTE3aC01NS43MThjLTQuMjIyLDAtNy42NDQsMy40MjItNy42NDQsNy42NDRWMzkyLjkyYzAsNC4yMjMsMy40MjIsNy42NDQsNy42NDQsNy42NDRoNTUuNzE4JiMxMDsmIzk7JiM5OyYjOTtjNC4yMjMsMCw3LjY0NC0zLjQyMiw3LjY0NC03LjY0NFYyNzEuNzYxQzQwMS45MiwyNjcuNTM4LDM5OC40OTgsMjY0LjExNywzOTQuMjc2LDI2NC4xMTd6IE0zODYuNjMxLDM4NS4yNzVoLTQwLjQyOXYtMTA1Ljg3aDAmIzEwOyYjOTsmIzk7JiM5O2g0MC40MjlWMzg1LjI3NXoiIHN0cm9rZT0id2hpdGUiIGZpbGw9IndoaXRlIi8+JiN4YTsJPC9nPiYjeGE7PC9nPiYjeGE7PGc+JiN4YTsJPGc+JiN4YTsJCTxwYXRoIGQ9Ik0zOTQuMjc2LDQwOS43MzdoLTU1LjcxOGMtNC4yMjMsMC03LjY0NCwzLjQyMi03LjY0NCw3LjY0NHY0My40ODhjMCw0LjIyMywzLjQyMiw3LjY0NCw3LjY0NCw3LjY0NGg1NS43MTgmIzEwOyYjOTsmIzk7JiM5O2M0LjIyMywwLDcuNjQ0LTMuNDIyLDcuNjQ0LTcuNjQ0di00My40ODhDNDAxLjkyLDQxMy4xNTksMzk4LjQ5OCw0MDkuNzM3LDM5NC4yNzYsNDA5LjczN3ogTTM4Ni42MzEsNDUzLjIyNWgtNDAuNDI5di0yOC4xOTkmIzEwOyYjOTsmIzk7JiM5O2g0MC40MjlWNDUzLjIyNXoiIHN0cm9rZT0id2hpdGUiIGZpbGw9IndoaXRlIi8+JiN4YTsJPC9nPiYjeGE7PC9nPiYjeGE7PGc+JiN4YTsJPGc+JiN4YTsJCTxwYXRoIGQ9Ik0yNzkuMDQ5LDQwMi4yNjNIMTQ4LjNjLTQuMjIzLDAtNy42NDQsMy40MjItNy42NDQsNy42NDR2MjQuNDYyYzAsNC4yMjIsMy40MjMsNy42NDQsNy42NDQsNy42NDRoMTMwLjc0OSYjMTA7JiM5OyYjOTsmIzk7YzQuMjIzLDAsNy42NDQtMy40MjIsNy42NDQtNy42NDR2LTI0LjQ2MkMyODYuNjkzLDQwNS42ODUsMjgzLjI3Miw0MDIuMjYzLDI3OS4wNDksNDAyLjI2M3ogTTI3MS40MDUsNDI2LjcyNWgtMTE1LjQ2di05LjE3MyYjMTA7JiM5OyYjOTsmIzk7aDExNS40NlY0MjYuNzI1eiIgc3Ryb2tlPSJ3aGl0ZSIgZmlsbD0id2hpdGUiLz4mI3hhOwk8L2c+JiN4YTs8L2c+JiN4YTs8Zz4mI3hhOwk8Zz4mI3hhOwkJPHBhdGggZD0iTTI1NC4zMDEsMTE4LjI1MWMtMTIuNDUsMC0yMi41NzgsMTAuMTI5LTIyLjU3OCwyMi41NzhjMCwxMi40NDksMTAuMTI5LDIyLjU3NywyMi41NzgsMjIuNTc3JiMxMDsmIzk7JiM5OyYjOTtjMTIuNDQ5LDAsMjIuNTc4LTEwLjEyOSwyMi41NzgtMjIuNTc3QzI3Ni44OCwxMjguMzc5LDI2Ni43NTEsMTE4LjI1MSwyNTQuMzAxLDExOC4yNTF6IE0yNTQuMzAxLDE0OC4xMTcmIzEwOyYjOTsmIzk7JiM5O2MtNC4wMTksMC03LjI5LTMuMjctNy4yOS03LjI4OXMzLjI3MS03LjI5LDcuMjktNy4yOWM0LjAxOSwwLDcuMjksMy4yNzEsNy4yOSw3LjI5JiMxMDsmIzk7JiM5OyYjOTtDMjYxLjU5MSwxNDQuODQ4LDI1OC4zMjEsMTQ4LjExNywyNTQuMzAxLDE0OC4xMTd6IiBzdHJva2U9IndoaXRlIiBmaWxsPSJ3aGl0ZSIvPiYjeGE7CTwvZz4mI3hhOzwvZz4mI3hhOzxnPiYjeGE7CTxnPiYjeGE7CQk8cGF0aCBkPSJNNDIwLjA5NywwSDkxLjkwM2MtNC4yMjMsMC03LjY0NCwzLjQyMi03LjY0NCw3LjY0NHY0OTYuNzExYzAsNC4yMjMsMy40MjIsNy42NDQsNy42NDQsNy42NDRoMzI4LjE5NCYjMTA7JiM5OyYjOTsmIzk7YzQuMjIzLDAsNy42NDQtMy40MjIsNy42NDQtNy42NDRWNy42NDRDNDI3Ljc0MSwzLjQyMiw0MjQuMzIsMCw0MjAuMDk3LDB6IE0xNzYuODA3LDI5OS41ODcmIzEwOyYjOTsmIzk7JiM5O2MtOS4zNDUsMTAuNzI3LTEzLjAxNCwyNC44NjItMTAuMDYzLDM4Ljc4MWMyLjk1MSwxMy45MTcsMTIuMDM4LDI1LjM0OCwyNC45MzIsMzEuMzYyYzYuMDgyLDIuODM1LDEyLjUzOSw0LjI0NywxOC45ODQsNC4yNDcmIzEwOyYjOTsmIzk7JiM5O2M3LjIyMywwLDE0LjQzLTEuNzc1LDIxLjA2OC01LjMwOGMxMi41NTctNi42ODYsMjEuMDI3LTE4LjU4MSwyMy4yMzgtMzIuNjM2bDguOTA1LTU2LjYyOGgzMy45NTd2MTczLjgxOUgxMjQuMDA5di0xNzMuODJoNzAuMzgxJiMxMDsmIzk7JiM5OyYjOTtMMTc2LjgwNywyOTkuNTg3eiBNMjU1LjgyNSwyMzIuMTY3bC00LjU1MiwyOC45MzdjLTAuMDAxLDAuMDA0LTAuMDAxLDAuMDA4LTAuMDAyLDAuMDEybC0xMS40MDgsNzIuNTQxJiMxMDsmIzk7JiM5OyYjOTtjLTEuNDU4LDkuMjY2LTcuMDQxLDE3LjEwOC0xNS4zMTksMjEuNTE1Yy04LjI3OSw0LjQwOS0xNy45MDMsNC42NjItMjYuNDA0LDAuN2MtOC41MDEtMy45NjUtMTQuNDkyLTExLjUtMTYuNDM3LTIwLjY3NiYjMTA7JiM5OyYjOTsmIzk7Yy0xLjk0NS05LjE3NSwwLjQ3My0xOC40OTUsNi42MzUtMjUuNTY3TDI1NS44MjUsMjMyLjE2N3ogTTI3MC43MywyMDAuMmMtMy4wODktMS40NDItNi43NTUtMC42NjItOC45OTUsMS45MDZsLTQxLjQyMiw0Ny41NDQmIzEwOyYjOTsmIzk7JiM5O2MtNDcuMzA4LTE0Ljc0Ny03OS45OTYtNTguODkyLTc5Ljk5Ni0xMDguODIzYzAtNjIuODUxLDUxLjEzMy0xMTMuOTg1LDExMy45ODQtMTEzLjk4NXMxMTMuOTg0LDUxLjEzNCwxMTMuOTg0LDExMy45ODUmIzEwOyYjOTsmIzk7JiM5O2MwLDU4LjQ3OC00My40MDcsMTA2LjUwOS0xMDAuNDIyLDExMy4xODZsNy4xODctNDUuN0MyNzUuNTc5LDIwNC45NSwyNzMuODE3LDIwMS42NCwyNzAuNzMsMjAwLjJ6IE00MTIuNDUzLDQ5Ni43MTFIOTkuNTQ3JiMxMDsmIzk7JiM5OyYjOTtWMTUuMjg5aDEyMy44ODhjLTU2LjQyNiwxMy44NzYtOTguNDA3LDY0Ljg5NC05OC40MDcsMTI1LjU0YzAsNTQuMzgxLDM0LjE5NiwxMDIuNzExLDg0LjQzNywxMjEuMjc0bC0xLjc1NSwyLjAxNGgtOTEuMzQ1JiMxMDsmIzk7JiM5OyYjOTtjLTQuMjIzLDAtNy42NDQsMy40MjItNy42NDQsNy42NDR2MTg5LjEwOGMwLDQuMjIzLDMuNDIyLDcuNjQ0LDcuNjQ0LDcuNjQ0aDE4OS4xMDdjNC4yMjMsMCw3LjY0NC0zLjQyMiw3LjY0NC03LjY0NFYyNzEuNzYxJiMxMDsmIzk7JiM5OyYjOTtjMC00LjIyMy0zLjQyMi03LjY0NC03LjY0NC03LjY0NGgtMTIuMTVjMjAuMjg4LTYuNCwzOC43OTktMTcuNzg3LDUzLjk1Mi0zMy40NjljMjMuNDA5LTI0LjIyNSwzNi4zLTU2LjEyMywzNi4zLTg5LjgxOSYjMTA7JiM5OyYjOTsmIzk7YzAtNjAuNjQ3LTQxLjk4Mi0xMTEuNjY0LTk4LjQwNy0xMjUuNTRoMTI3LjI4NlY0OTYuNzExeiIgc3Ryb2tlPSJ3aGl0ZSIgZmlsbD0id2hpdGUiLz4mI3hhOwk8L2c+JiN4YTs8L2c+JiN4YTs8L3N2Zz4="
       preserveAspectRatio="none"
       id="image1552" />
    <path
       d="m 211,46.04 69.9,0.44"
       fill="none"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="stroke"
       id="path1554" />
    <path
       d="m 287.65,46.52 -9.03,4.44 2.28,-4.48 -2.23,-4.52 z"
       fill="#ffffff"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="all"
       id="path1556" />
    <path
       d="M 169,75 V 63.92"
       fill="none"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="stroke"
       id="path1558" />
    <path
       d="M 291,95.46 221.1,95.02"
       fill="none"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="stroke"
       id="path1560" />
    <path
       d="m 214.35,94.98 9.03,-4.44 -2.28,4.48 2.23,4.52 z"
       fill="#ffffff"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="all"
       id="path1562" />
    <path
       d="M 353,76.27 V 63"
       fill="none"
       stroke="#ffffff"
       stroke-width="3"
       stroke-miterlimit="10"
       pointer-events="stroke"
       id="path1564" />
  </g>
</svg>

</div>
<br>

And a request for a subscriber-only protected file breaks down like so:

1. The user requests a file, which Nginx is the reverse proxy for, Nginx sends the request along to the Django backend.

2. Django authenticates the user based on request parameters, and if the user has permission to get the file, Django generates a temporary signed URL from the storage backend to pass back to Nginx. 

3. Django responds to Nginx with instructions to serve an [X-Sendfile](https://www.nginx.com/resources/wiki/start/topics/examples/xsendfile/){:target="_blank" rel="noopener"} response.

4. Nginx checks to see if the file is in its cache, which we'll define further along in this post. If the file is in the cache, serve it from there, else add it to the cache while sending it back to the user.

The important part here from the standpoint of the user is that the URL for the file is the one they initially sent. X-Sendfile preserves the original URL. All of the redirects and cache checks happen internally without the user's knowledge, so the user never sees a "real" URL to the file. The important part from a developer's standpoint is that the backend is agnostic, you could change storage providers, change hosting services for the front end, change from Nginx to Apache to some other web server if you like, and none of this would matter to the user (or any application the user chooses to use to access their files), because the URLs will stay the same. The only thing we're locking ourselves into is Django, as the framework holding our user accounts and the interface to the storage backend.

More on that last point later.

# Nginx Configuration
---

```nginx
/etc/nginx/nginx.conf

http {
...
    proxy_cache_path /var/cache/nginx keys_zone=media-files:4m max_size=10g inactive=1440m
...
}
```


```nginx
/etc/nginx/sites-enabled/mysite.conf

server {
    server_name www.mydomain.net;
    listen 80;
    listen [::]:80;
        location / {
            return 301 https://$host$request_uri;
        }
}
server {
    server_name www.mydomain.net;
    listen 443 ssl proxy_protocol;
    listen [::]:443 ssl proxy_protocol;
    client_max_body_size 200M;
    include /etc/nginx/ssl.conf;
    include /etc/nginx/header.conf;
    location / {
        proxy_redirect off;
        proxy_force_ranges on;
        proxy_buffering off;
        real_ip_header proxy_protocol;
	proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://127.0.0.1:8080;
    }
    location /favicon.ico { 
        access_log off; log_not_found off; 
    }
    location ~ ^/media_download/(.*?)/(.*?)/(.*) {
        internal;
        resolver 8.8.8.8 1.1.1.1 9.9.9.9 127.0.0.1 ipv6=off;
        set $download_protocol $1;
        set $download_host $2;
        set $download_path $3;
        set $download_url $download_protocol://$download_host/$download_path;
        proxy_set_header Host $download_host;
        proxy_set_header Authorization '';
        proxy_set_header Cookie '';
        proxy_hide_header x-amz-request-id;
        proxy_hide_header x-amz-id-2;
        use_temp_path=off;
        proxy_cache media-files;
        proxy_cache_key $scheme$proxy_host$download_path;
        proxy_cache_valid 1440m;
        proxy_cache_use_stale error timeout updating http_500 http_502 http_503 http_504;
        proxy_cache_revalidate on;
        proxy_ignore_headers Set-Cookie;
        add_header X-Cache-Status $upstream_cache_status;
        proxy_pass $download_url$is_args$args;
        proxy_intercept_errors on;
        error_page 301 302 307 = @handle_redirect;
    }
    location @handle_redirect {
        resolver 8.8.8.8 1.1.1.1 9.9.9.9 127.0.0.1 ipv6=off;
        set $saved_redirect_location '$upstream_http_location';
        proxy_pass $saved_redirect_location;
    }
}
```

The `proxy_cache_path` cache definition is explained in the [Nginx docs](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_path){:target="_blank" rel="noopener"} in detail. We've defined the location on the disk `(/var/cache/nginx)` (make sure this is owned by the user the server runs as), the name of the cache `(keys_zone=media-files)` which we could re-use for multiple locations or servers, the memory size of the cache index `(:4m)`, and the maximum size of the cached files `(max_size=10g)`. Lastly, we'll consider the files expired from the cache if they're older than a day, or 1440 minutes `(inactive=1440m)`. Four megabytes will hold about 32,000 keys in the cache index, which is overkill here but 4 megs of reserved memory won't kill us.  Ten gigabytes you may need to season to taste depending on the size and number of media files you're serving in any given day.  And similarly, speaking of days, you might want to adjust the `inactive` setting if your files rarely change and you want to cache them longer than 24 hours.  

The first part of this Nginx site configuration `mysite.conf` (through the root `/` location and the `access_log` being turned off, as well as a redirect from non-SSL requests to the SSL listening port on 443) is pretty standard and should be familiar to anyone who has deployed Nginx as a reverse proxy for a backend app before. 

Where we're getting into new territory is the `media_download` location.  Lets break down what Nginx will match in a URL here:

`~` will match any preceding root URL, and the three `(.*?)` regex sections will match three URL parameters that we send along to Django for user and file authentication.  The first parameter will be the base64-encoded email address of the user, which our backend uses as a user id ([this guide](https://learndjango.com/tutorials/django-log-in-email-not-username){:target="_blank" rel="noopener"} covers the process of setting up a new Django projecct with a custom user model, the same user-signup library that I personally use ([django-allauth](https://github.com/pennersr/django-allauth){:target="_blank" rel="noopener"}), and changing the default user id in Django to email addresses instead of user names).

`internal` is important, it lets Nginx know that this is an internal redirect, this prevents the server from sending the client directly to the provided URL.  Since Nginx will itself be fetching these files from an external URL, it needs DNS resolution, so we've hard-coded some public DNS servers to use (8.8.8.8 is one belonging to Google, 1.1.1.1 is one belonging to Cloudflare, and 9.9.9.9 is one belonging to the firewall service Quad9, lastly we fall back to localhost's DNS resolution if those fail). 

In the next few lines beginning with `set` we're re-composing the *external* URL to the file to explicitly declare the `host` header for caching purposes. A simpler implementation would be to simply point the download to a file on the disk `(/path/to/file.mp4)`, but the problem with this method is by default Nginx will use the components of the original URL to build its cache.  Since we have user data in the URL, the URL to the *same file* will be *different* for each user, and that method won't work.

To visualize this, lets consider what a URL will look like:

```
http://www.mydomain.net/premium_media/5AYXdoaWxlY.../2/av2r2j-8f176710bf483694.../file.mp4
```

`5AYXdoaWxlY...` (shortened) is our user's base64 encoded email address, which will change for each user even if they request the same file.  `/2/` is the internal id of the file, which we can use to look up the file the user is requesting, and `av2r2j-8f176710bf483694...` (shortened) is the token that Django will use to authenticate the user which will also change for each user even if they request the same file. `/file.mp4` is the last part of the URL, the `$download_path` (equivalent to the file name). Since two of these URL components will be different each time the *same file* is requested, we can't use the *whole* URL as a cache key [as Nginx does by default](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_key){:target="_blank" rel="noopener"}, otherwise:

```
http://www.mydomain.net/premium_media/3AYXdslkdfZ.../2/av2r2h-9gdf323dsd483692.../file.mp4
http://www.mydomain.net/premium_media/4AYXsdaWxlY.../2/bv3r3i-8sfd8343ff483693.../file.mp4
http://www.mydomain.net/premium_media/5ADDdoaWxlX.../2/cv4r4j-8f176710bf483694.../file.mp4
```

All of the above files would be seen as different by the default Nginx cache key, and thus defeat the whole purpose of our cache: serving the same file to different users rather than fetching it from the storage backend every time it's requested.

In the `proxy_set_` and `proxy_hide_` lines we're setting the response defaults to both accommodate our cache mechanism and determine what the user will see in the response.  We've set the `host` header to the backend storage host (i.e. the url to AWS S3 or Digital Ocean or whatever storage service you use), the `authorization` and `cookie` headers to be blank, and hidden the headers returned by the backend storage host relating to the request itself, since our user doesn't need those. Finally, we've specified our cache, the `media-cache` which is created in the main Nginx config file.  

Next is the most important part of our cache mechanism, the `proxy_cache_key`. Rather than letting it default to the full URL of the file requested by the user for the reasons mentioned above, we are setting it to `$scheme` (http or https) plus `$proxy_host` (a default variable set by Nginx which should be your website's hostname, unless you change it...) plus the `$download_path;` which, as mentioned above, is equivalent to the file name.  All of these URL parameters will remain unchanged even if the user requesting them changes, so by this custom implementation of the cache keys we'll have our desired outcome of the same file being served from the disk cache to multiple users.

Last but not least `proxy_pass` is set to the URL returned from Django to tell Nginx to proxy the remote file, and we have some error handling, the most mentionable of which is the `@handle-redirect` directive. It's not uncommon for storage object services to return redirects in the normal course of serving a file, for instance if you mirror your files across multiple regions the storage service might redirect a user to a datacenter closer to them geographically.  If Nginx were not instructed to handle those, it would pass the redirect URL back to the user directly rather than maintaining the URL the user requested.  By telling Nginx to handle 3xx redirect responses in the same way it handles the other requests the URL remains the same to the user even if the back end returns a redirect URL.


# Django Configuration
---

On the Django side of things, this all plays very nicely with [django-storages](https://github.com/jschneier/django-storages){:target="_blank" rel="noopener"}. Out of the box it supports all of the major cloud providers' storage services, plus those API compatible with S3 from other cloud services, plus FTP, Dropbox, and Apache's libcloud wrapper.  

If you're using Wagtail as I am, the [wagtailmedia](https://github.com/torchbox/wagtailmedia){:target="_blank" rel="noopener"} addon gives you a nice admin UI to manage your media files with.  Lets take a look at what happens to a media file uploaded to a private S3 bucket in the Wagtail admin with django-storages and wagtailmedia:

![wagtailmedia with django-storages](https://rentfree.media/assets/images/wagtailmedia.jpg){:.img}

You can see in the URL at the bottom of the screenshot that when I hover over the file name to get the URL to it, django-storages has generated a signed URL for me to use since this file is in a private S3 bucket.  That URL is only good for 10 minutes, but it's the one Nginx will use to get the file and fill its local disk cache when a user requests the file in question.  Subsequent requests during that day will be served from the Nginx cache, but only after the user has been passed to the Django back end first for authentication. 

Next let's look at the Django view which responds to the Nginx media file requests.

```python
myapp/urls.py

from django.urls import include, path, re_path
from myapp.views import PremiumMediaView

urlpatterns = [
...
{% raw %}re_path(r'^premium_media/(?P<uidb64>[-\w]*)/(?P<fileid>[-\w]*)/(?P<token>[-\w]*)/(?P<file_name>[\w.]{0,256})$', PremiumMediaView.as_view(), name='premium_media'),{% endraw %}
...
]

```

The regex patterns beginning with `?P` and ending with `/` are matched by the arg names in the `<...>` declarations. So for our view, we can pass those args to the function that processes the user data and makes sure our user is granted access to the requested file, like in the view below:


```python
myapp/views.py

from myapp.models import CustomMedia
from myapp.tokens import premium_token
from urllib.parse import urlparse
from django.views.generic.base import View
from django.utils.http import urlsafe_base64_decode
from django.http import Http404, HttpResponse, HttpResponseForbidden

class PremiumMediaView(View):
    def get(self, request, uidb64, fileid, token, file_name):
        try:
            user = UserModel.objects.get(email=urlsafe_base64_decode(uidb64).decode())
        except:
            user = None
        try:
            file = CustomMedia.objects.get(pk=fileid)
        except:
            raise Http404('No such file exists')
        if user and premium_token.check_token(user, token) and user.stripe_subscription.status == 'active':
            url = file.url
            protocol = urlparse(url).scheme
            response = HttpResponse()
            response['X-Accel-Redirect'] = '/media_download/' + protocol + '/' + url.replace(protocol + '://', '')
            return response
        else:
            return HttpResponseForbidden()
```

Those familiar with django will probably wonder at this point, "what's a `premium_token` and what does it do?"  It's actually a very cool and simple included-battery of the Django framework: it's a repurposed password reset token.  

Django includes a secure means of generating a one-time password reset link for a user.  All the user has to do is click the "forgot password" link on the login form and Django will email that user a self-authenticating link that will expire either after one use or after a set amount of time has passed.  We can re-use this functionality to make login-links for subscriber media files, by simply telling the token function to make the token against different user model fields than the password reset token.

Here's how:

```python
myapp/tokens.py


from django.contrib.auth.tokens import PasswordResetTokenGenerator
from datetime import datetime, time
from django.utils.crypto import constant_time_compare, salted_hmac
from django.utils.http import base36_to_int, int_to_base36


class PremiumSubscriberTokenGenerator(PasswordResetTokenGenerator):

    def _make_hash_value(self, user, timestamp):
        return str(user.id) + str(user.is_paysubscribed) + str(user.uuid) + str(user.stripe_subscription.status)

    def check_token(self, user, token):
        """
        Check that a password reset token is correct for a given user.
        """
        if not (user and token):
            return False
        # Parse the token
        try:
            ts_b36, _ = token.split("-")
            # RemovedInDjango40Warning.
            legacy_token = len(ts_b36) < 4
        except ValueError:
            return False

        try:
            ts = base36_to_int(ts_b36)
        except ValueError:
            return False

        # Check that the timestamp/uid has not been tampered with
        if not constant_time_compare(self._make_token_with_timestamp(user, ts), token):
            # RemovedInDjango40Warning: when the deprecation ends, replace
            # with:
            #   return False
            if not constant_time_compare(
                self._make_token_with_timestamp(user, ts, legacy=True),
                token,
            ):
                return False

        # RemovedInDjango40Warning: convert days to seconds and round to
        # midnight (server time) for pre-Django 3.1 tokens.
        now = self._now()
        if legacy_token:
            ts *= 24 * 60 * 60
            ts += int((now - datetime.combine(now.date(), time.min)).total_seconds())
        # Check the timestamp is within limit.
        if (self._num_seconds(now) - ts) > 3153600000:
            return False

        return True

premium_token = PremiumSubscriberTokenGenerator()
```

Most of this has been copied and pasted from the core Django `PasswordResetTokenGenerator` class, we're simply overriding two methods.  First, we need to override `_make_hash_value` to tell our custom generator to use different fields than the default Django generator, which uses the user's password and last login date/time.  We want to use the user id, the field which tells whether or not the user is a paid subscriber `(user.is_paysubscribed)`, a uuid field `(user.uuid)` that is never shown to the user, and the user's payment status from our Stripe payment library `(user.stripe_subscription.status)` which tells whether or not the subscription is active.

Next, the default timeout on Django generated tokens is a few days.  You don't want to change the default for *all* tokens, because that would make password reset tokens lingering in users' email accounts valid indefinitely.  What we've done above with that in mind isn't rocket science, we've just copied and pasted the whole `check_token` method from the core Django `PasswordResetTokenGenerator` class, and changed the timeout to `3153600000` seconds instead of the default. Translated to something more readable, that amount of seconds is equal to 100 years.

As you can see here, the Django generated tokens aren't stored in the database, rather they are generated for the user, and then generated again when Django checks them. If any of the database fields have changed since the token was generated, or if the time limit has expired, the token will check `False`, otherwise `True` will be returned by the token check method.

With everything in place, [curl](https://curl.se/){:target="_blank" rel="noopener"} is a good tool to use to check and make sure all of this works.

We can make user / token link args in the Django shell:

```python

$ python manage.py shell
Python 3.8.5 (v3.8.5:580fbb018f, Jul 20 2020, 12:11:27) 
[Clang 6.0 (clang-600.0.57)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
(InteractiveConsole)
>>> from myapp.tokens import premium_token
>>> from myapp.models import CustomUser # or wherever your usermodel is...
>>> from django.utils.http import urlsafe_base64_encode
>>> from django.utils.encoding import force_bytes
>>> 
>>> user = CustomUser.objects.get(pk=1)
>>> token = premium_token.make_token(user)
>>> uidb64 = urlsafe_base64_encode(force_bytes(user.email))
>>> token
'av394n-8f17...'
>>> uidb64
'YWRtaW5AYXd...'
>>>  
```

With which we can go on to curl with the generated base64 email and token, replace the interior `/1/` and the file name with a media file id and name that exist in your private media S3 bucket...

```bash
$ curl -I https://www.mydomain.net/premium_media/YWRtaW5AYXd.../1/av394n-8f17.../sample1.m4a

HTTP/2 200 
server: nginx
date: Mon, 25 Oct 2021 23:45:33 GMT
content-type: audio/x-m4a; charset=utf-8
content-length: 2034429
expires: Tue, 26 Oct 2021 00:45:33 GMT
cache-control: max-age=3600
last-modified: Tue, 17 Aug 2021 23:23:58 GMT
etag: "c7cb4cfbf25ef6b3d3bd9b0d9ef77a7e"
x-cache-status: HIT
accept-ranges: bytes
``` 

If you run this twice, you should see a `MISS` on `x-cache-status` the first time you request a file, and a `HIT` on `x-cache-status` if you request the same file again. If you use a different user and different token to request the same file, you should see another `HIT` on the cache.

You can use whatever user fields you like to make tokens, but there are some considerations:

* There should be at least one field that the user cannot possibly see, otherwise a particularly clever user might be able to generate a token on their own whether or not they are actually a paid subscriber.  In my case user.id is the user's email, so that isn't very obscure, lots of users might know other users' email addresses. Their Stripe status isn't very obscure either, since a user familiar with Stripe might know that a valid subscription returns `active` as a status. What *is* obscure is the uuid. I generate uuids for users when they sign up and store that uuid in the database, but I don't use the uuid for anything except authentication token hashes so it's not feasible for a user to find out what their uuid is.

* If the user's subscription status changes, one of the fields used to make tokens also needs to change for existing tokens to be invalidated.  If a user pays their subscription fee on whatever schedule we are billing them, we don't want to invalidate their links.  Those links and the tokens within them should remain valid indefinitely. In this example, their `stripe_subscription_status` will show `active` if they are paid up and current, but will change to something else if they aren't.

* It should also be noted that your Django server doesn't know what a subscription is, nor does it know what Stripe is, or a subscription_status, or any other such thing. It is, after all, just a big dumb machine that serves things. It knows how to count, check true and false, and parse strings of text, which is to say it doesn't really "know" anything at all. Therefore it falls to you, the developer, to put conditional checks in your code to tell Django whether or not a user has access to a thing. Otherwise, if a user generates a `premium_token` when their subscription is lapsed or inactive or any other such status, it will be valid and let them download files, and would only be invalidated if their payment status becomes `True` instead of `False` which is the opposite of what you want (assuming you like making money from your subscriptions)! In my case I set my subscription profile page to never cache (so that it always loads fresh user data every time it is requested), and has this conditional statement in it around the fields that show users their subscription links:

```html
{% raw %}{% if request.user.stripe_subscription and request.user.stripe_subscription.status == 'active' %}
<h1 class="mb-4">{% trans "Your Premium Content Links" %}</h1>
<hr>
<p>{% blocktrans %}Links to the premium content you are subscribed to are shown below. 
DO NOT SHARE THEM. They are bound to your account, and enable password-less login to our 
premium services. You can import them to any device or application you choose such as 
feed readers or podcast players to access your premium episodes.{% endblocktrans %}</p>
{% if podcasts %}
<div class="col-12 text-indent">
  {% for podcast in podcasts %}
    <h4 class="mb3">{{ podcast.parent_page.rss_title }}:</h3>
    <p><a href="{{ podcast.parent_page.url }}premiumfeed/{{ uid }}/{{ token }}/">{{ ... }}/</a></p>
  {% endfor %}
</div>
{% endif %}{% endraw %}
```

So for a user to have this section of the subscription profile page rendered to them, they have to be logged in, have a `stripe_subscription` and that subscription must have an `active` status.  Otherwise, the whole `Premium Content Links` section of the subscription profile page will not be rendered, and thus a user without a paid subscription can never get Django to show them a `premium_token` link.  The subscription profile page is the only view on my site where a user can see a `premium_token` link.

All that's left here then, is how to make a token. That's easy. As you saw in the Django shell example above in which I manually created a base64 encoded email and a user token, the same `PasswordResetTokenGenerator` class that checks tokens also makes tokens.

```python
myapp/views.py

from django.db.models.query_utils import Q
from myapp.tokens import premium_token
from myapp.models import PodcastContentPage

def a_view(request):
    ...
    podcast_queryset = PodcastContentPage.objects.filter(Q(id__in=private_queryset_pages)).order_by('something')

    token = premium_token.make_token(request.user)
    
    uid = urlsafe_base64_encode(force_bytes(request.user.email))
    ...

    return render(request, 'myapp/subscriber_profile.html', {
        'token': token,
        'uid': uid,
        'podcasts': podcast_queryset if podcast_queryset else None,
    })
```

In my case, I use Wagtail's [routable page](https://docs.wagtail.io/en/stable/reference/contrib/routablepage.html){:target="_blank" rel="noopener"} methods to create an RSS feed under the index page of my individual articles / episodes / etc.  [Wagtail-personalisation](https://github.com/wagtail/wagtail-personalisation){:target="_blank" rel="noopener"} is a nifty library that can be custom-purposed to serve premium content to premium subscribers transparently, I use it for separating average-Joe users from paying users.

```python
from myapp.podcast_feeds import PodcastFeed #(django RSS framework custom feed)
from wagtail.core.models import Page
from wagtail.core.utils import resolve_model_string
from wagtail_personalisation.utils import exclude_variants
from wagtail.contrib.routable_page.models import RoutablePageMixin, route

class PodcastContentIndexPage(RoutablePageMixin, Page):
    ...

    @cache_page
    @route(r'^rss/$', name='rss')
    def rss_feed(self, request):
        """
        Renders a public RSS Feed for the public podcast episodes under this page.
        """

        queryset = resolve_model_string(self.index_query_pagemodel, self._meta.app_label)

        last = exclude_variants(queryset.objects.child_of(self).live()).first()
        first = exclude_variants(queryset.objects.child_of(self).live()).last()
        all_public = exclude_variants(queryset.objects.child_of(self).live()).order_by('-date_display')
        all_private = None
        rss_link = request.get_raw_uri()
        home_link = self.get_site().root_url
        token = None
        uidb64 = None


        # Construct the feed by passing ourself to it, so it can determine the feed's "link".
        feed = PodcastFeed(request, rss_link, home_link, first, last, all_public, all_private, token, uidb64)
        # 'feed' is a class-based view, so we need to call feed and pass it the request to get our response.
        return feed(request)

    @route(r'^premiumfeed/(?P<uidb64>[-\w]*)/(?P<token>[-\w]*)/$', name='premiumfeed')
    def premium_feed(self, request, uidb64, token):
        ...
        """
        basically the same methods as the public RSS feed and the same 
        base64 user id and token as the rest of this guide, but you have 
        to build the private / paid episode queryset rather than the public 
        one
        """
```

With something like the above we can build a queryset of podcast episode pages, to pass into Django's [syndication feed framework](https://docs.djangoproject.com/en/3.2/ref/contrib/syndication/){:target="_blank" rel="noopener"} to build an Apple podcasts / iTunes RSS feed, like Patreon does (only without the 15%-18% fee...). Or if you're doing articles instead of podcasts maybe you want to generate RSS feeds for premium articles, like Substack does (only without the 15%-18% fee...).  Or maybe you want to do both all under one banner, without any fees other than what the payment processor charges. And of course the users will be *your users*, not the users of some "platform" that could simply decide that [they're not going to let you speak (or pay your rent and bills) anymore](https://en.wikipedia.org/wiki/Patreon#Bans_of_specific_users){:target="_blank" rel="noopener"}.

# Hosting Considerations
---

Those of you who are familiar with web hosting might be thinking to yourselves "but, but, by doing this you're bypassing the egress bandwidth of the storage object service and routing all of the bandwidth through your web server."  Yes, as a matter of fact you are.  

This is worth considering from a bandwidth cost perspective.

As of this writing, [AWS S3](https://aws.amazon.com/s3/pricing/){:target="_blank" rel="noopener"} costs $0.09/GB for outbound transfer, [Microsoft Azure](https://azure.microsoft.com/en-us/pricing/details/storage/blobs/){:target="_blank" rel="noopener"} seems to cost about the same, and both of them have lots of oddball fees for requests and redundancy tiers and other such stuff.  [Digital Ocean](https://docs.digitalocean.com/products/billing/bandwidth/){:target="_blank" rel="noopener"} charges $0.01/GB across the board, and gives you a baseline allowance that adjusts based on the fixed prices of servers and other such services you buy, which is suited particularly well to this guide.  For instance, if you have $50.00 worth of servers to run your Django/Wagtail media distribution empire from, you'll start out with 5 terabytes of bandwidth pre-paid every month, and will only be billed $0.01/GB for the amount you use in excess of that.  [Linode](https://www.linode.com/pricing/){:target="_blank" rel="noopener"} pricing mirrors that of Digital Ocean as does [Backblaze](https://www.backblaze.com/blog/transparency-in-cloud-storage-costs/){:target="_blank" rel="noopener"}.

With AWS it's a 6 / half dozen proposition. You pay $0.09/GB, whether you spend that bandwidth serving from S3 or spend it serving from an EC2 Linux instance makes no difference, but for the fact that by proxying requests via EC2 you aren't paying the request / operations fees from S3 that you would be if you use S3 directly as a server.  Azure is similar to AWS in this regard. As mentioned at the top of this article Digital Ocean has a hard cap on requests even though they don't charge for them, so if you're doing this with a lot of paid users you have to proxy your downloads on Digital Ocean, lest you exceed the cap and run into a situation where paid users can't get the files they've paid for.

Most interestingly, Cloudflare has announced a competing S3 object storage service that [will not charge bandwidth egress fees](https://blog.cloudflare.com/introducing-r2-object-storage/){:target="_blank" rel="noopener"} but rather only charge fees for requests and the actual storage space used.  This will likely be a very big deal in media distribution, and could cut into the usage of platforms like Youtube and Twitch.TV, as users will be able to serve even HD video on their own at reasonable costs rather than relying on third parties due to the bandwidth costs of serving video.

In any case, with the tools and the know-how there's no reason to pay the egregious percentage-of-your-gross fees that platforms like Patreon and Substack charge, go forth and serve your own content!

Thanks to [Ewen @ Mediasuite](https://www.mediasuite.co.nz/blog/proxying-s3-downloads-nginx/){:target="_blank" rel="noopener"} and [Oleksiy Kovyrin @ Scribd](https://kovyrin.net/2010/07/24/nginx-fu-x-accel-redirect-remote/){:target="_blank" rel="noopener"} for their previous posts on this topic.  Despite being over a decade apart the how-to for all three guides remains basically unchanged.  If only front end frameworks were as stable as back end servers...

