---
layout: post
title: Complex Custom Field Pagination in Django
author: Neal
categories: [wagtail, pagination, django]
image: /assets/img/posts/code.jpg
alt: code
intro: ...there's always a thing that doesn't work the way one wants. Today, it turned out to be pagination.
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

As wonderful as Django's built-in methods and functions are for 99% of what we use it for, there's always a thing that doesn't work the way one wants it to, and in my case today, that thing turned out to be pagination.

The Django pagination class does a worthwhile job, though, so it's not wise to dismiss it.  While it has near zero configurability, it does lazy load pages and thus vastly reduce the strain on your database by only loading the page ranges that it needs to display.

# But what if I don't want to split pages by count?

Herein lies the catch.  The only thing the Django paginator supports out of the box, is a count.  You give it a number, and that's how many items per page that you will have.  There's no other configuration option built-in that you can change.

In my case, I need to split up an index of podcasts and / or video streams by season numbers (if the author chooses to publish by season rather than a linear list of episodes).  I could have an entire different Page class for series episodes I suppose, but that doesn't really look very high-effort to the end user if they have to have a whole different database table for a minor change in the content type.  For the purposes of media distribution as an audio podcast or a video podcast, the episodes are still sent over the wire via RSS feeds in date order, they are just given extra category tags in the XML that allows apps rendering the episodes to re-sort them by series number after the fact.

# Visualizing the data

Here's what our data will look like:


| episode id |  episode number |    title       | etc  | etc  | season number  |
| ---------- | --------------- | -------------- | ---- | ---- | -------------- |
|      1     |         1       |  first episode | blah | blah |      1         |
|      2     |         2       | second episode | blah | blah |      1         |
|      3     |         1       |  third episode | blah | blah |      2         |
|      4     |         2       | fourth episode | blah | blah |      2         |

There are no page splits in the RSS, this is only a concern on the website that will host the show.  Predictably, we can't put hundreds of episodes on a single page, nor can we split them by some arbitrary number, because seasons may have different numbers of episodes and any offset mismatch could break the whole thing into a mess.

Optimally the pages should be split by season number, so if a season has 10 to 20 episodes all of that season would be on a single page in the index, and the adjoining pages in the index would be the next season or previous season.  And while the Django paginator can't do this, it does have useful properties and methods which we don't want to lose.  For instance it tells the template whether there is a next page or previous page, and thus makes generating links for breadcrumbs and menus very easy.

# Solution: Looping Multiple Paginators

While we don't want to break the golden rule of never looping a model to get data (and thus defeating Django's generally good database query efficiency), we can loop a list of known numbers from an index to make a list of querysets pretty efficiently.  That is, after all, what the paginator is doing with its default settings to some degree.

Here's the secret sauce recipe:

```python
class MyPage(Page):

...

    def get_context(self, request, *args, **kwargs):
        context = super().get_context(request, *args, **kwargs)

        all_children = self.get_children().live().public().order_by('first_published_at')

        if all_children.first().season_number and self.rss_itunes_type == 'serial':
            from website.utils import LocalPaginator
            from website.utils import LocalPage
            oldest_season = all_children.first().season_number
            loop_end = all_children.last().season_number
            season_qs = dict()
            for i in range(oldest_season, loop_end + 1):
                season_qs[i] = LocalPaginator(all_children.filter(season_number=i), 200)
                if i > oldest_season:
                    season_qs[i].has_previous = True
                    season_qs[i].previous_page_number = i - 1
                    season_qs[i].num_pages = loop_end
                if i < loop_end:
                    season_qs[i].has_next = True
                    season_qs[i].next_page_number = i + 1
                    season_qs[i].num_pages = loop_end
                season_qs[i].number = i

            pagenum = request.GET.get('p', 1)

            if pagenum == 'latest':
                try:
                    paged_series_children = season_qs.get(loop_end).page(1)
                except:
                    paged_series_children = season_qs[len(season_qs)].page(1)
            else:
                try:
                    paged_series_children = season_qs.get(int(pagenum)).page(1)
                except:
                    paged_series_children = season_qs[len(season_qs)].page(1)

            context['index_paginated'] = paged_series_children
            context['index_children'] = all_children
            
            
        else:
            paginator = Paginator(all_children, self.index_num_per_page)

            pagenum = request.GET.get('p', 1)
            try:
                paged_children = paginator.page(pagenum)
            except (PageNotAnInteger, EmptyPage, InvalidPage) as e:
                paged_children = paginator.page(1)

            context['index_paginated'] = paged_children
            context['index_children'] = all_children

        if request.GET.get('tag', None):
            tags = request.GET.get('tag')
            all_children = all_children.filter(tags__slug__in=[tags])
            context["has_tags"] = True 
            context["index_paginated"] = all_children

        return context
```

Overriding `get_context` should be familiar to anyone who has worked with Django and / or Wagtail before, so that's where this will live.  There are some caveats with how we start:

1. We can't work directly with an index (unless it's one of Django's fancy new (as of 3.x) "include" indexes that tack on extra data at the end? maybe?).  We need to use a queryset because we have to filter the results by season number and you can't filter indexes by data they don't have. As such, it's a good idea to define an index on the season number field here, to help the database get its page ranges as efficiently as possible.

2. In the interest of keeping complexity to a minimum we don't want to sort by anything but the data we're using in here.  If season numbers don't match the loop index numbers and url query parameter numbers, you will have a *lot* of extra complexity on your hands to make it all line up. So just do them in an order which makes the numbers align.  Generally that means starting with the lowest season number and working toward the highest season numbers, since page IDs go in the same direction.

3. Order of operations matters here. I've got three things in play.  First there's the consideration of whether someone landed on this index page via a category tag search.  In that case I don't want any sorting or splitting except for the tags, so the tag search query parameter comes last and will override any others that may match.  Secondly, there are perhaps people who don't want to split up their podcast or video cast into seasons, and just want straight episodes sorted by date.  In that case all of this season business should be ignored and that method of pagination by a set page count should work with Django's out of the box pagination functions.  Thirdly, the season pagination can only fire conditionally, if both the index page and its first child page are set up properly for season sorting.

4. I have some other pagination methods that are user-selectable in this setup that require `all_children` so we want to preserve those for pagination scenarios that might use them, and thus `all_children` still needs to be passed in the context for those index page options.

# Where is this going to fall apart as a concept?

As most good ideas do, this one suffers from a 'gotcha'.  You can't override a function return value or model field with a variable.  See how we are explicitly setting things like `has_previous` and `has_next` in our loop over the page ranges?  Yeah... you can't do that.  Those are properties in the Django core paginator class, and it's going to tell you exactly why you can't do that if you try to.

So the solution is to make a new paginator class (and the Page class it depends on) with some of those extra screws and bolts from the factory left out.  All I've done here is copied the Paginator class from Django core over to my app's `utils.py` file, along with its associated Page class, and reproduced all of its required imports.  Then, I simply deleted all of the functions and properties that I need to override from the local app copy, and imported it as a different name instead of the core Django Page and Paginator classes, taking care to rename "Page" references to "LocalPage" while doing so.

# Step by step: the data and the loop

With the properties and functions that were blocking my path gone, lets step through what our loop is doing:

```python
    all_children = self.get_children().live().public().order_by('first_published_at')

        if all_children.first().season_number and self.rss_itunes_type == 'serial':
            from website.utils import LocalPaginator
            from website.utils import LocalPage
            oldest_season = all_children.first().season_number
            loop_end = all_children.last().season_number
            season_qs = dict()
            for i in range(oldest_season, loop_end + 1):
                season_qs[i] = LocalPaginator(all_children.filter(season_number=i), 200)
                if i > oldest_season:
                    season_qs[i].has_previous = True
                    season_qs[i].previous_page_number = i - 1
                    season_qs[i].num_pages = loop_end
                if i < loop_end:
                    season_qs[i].has_next = True
                    season_qs[i].next_page_number = i + 1
                    season_qs[i].num_pages = loop_end
                season_qs[i].number = i
```

First we are defining our queryset that includes all pages beneath this index. Next, a couple of conditionals to make sure that we are only doing this to someone who really, really, really wants their podcast or video cast to be sorted by seasons, so much so that they have properly set up both the index page and the first episode page to do so.

With those preliminary items done, we next import the local versions of the Django core Paginator and Page classes, without the properties and functions that we're going to override in our paginator instances.  

Then, create an empty dict.  A dictionary is a great data container for this purpose, since it can have both arbitrary keys and arbitrary values that we can define programmatically.  We then need to do some simple math.

If we're going to put every episode from a season onto its own page, then logically we will have a number of pages that matches the total number of seasons.  That can be deduced from the variables we've set before the loop here.  By taking the season number from the first episode in the index, and the season number from the last episode in the index, we will have an always up-to-date knowledge of how many seasons there are, and how they are numbered.

To make everything line up as I mentioned above, we'll start with the `oldest_season` which should probably be 1, and loop until we reach the `loop_end` which will be the most recent season. The first season (1) should line up with the first page in the index listing (1) and that will also be the first paginator instance (1) of several that we are going to create.  If you want to change these, you are a lot braver than I am and will probably find yourself in a world of confusion and doubt.  Or you are a mathematical genius, but in either case I'm not.  The numbers lining up makes all of this relatively simple, so keep them that way.

With each loop of the range, we are making a whole new paginator instance, with an upper limit of 200 episodes per page.  I don't know of any podcast or video cast that puts 200 episodes into a season, so that seems like a good arbitrary number to throw in, and you have to give the paginator a number because that's all it knows.  It will only include episodes that actually exist, so this could just as well be 500 or 1,000 if you insist, just to be sure.

Because we want to work with greater-than or less-than logic, it makes sense to simply add 1 to the end of the last season for `loop_end` in the upper limit of the range loop as a place for the it to stop.  So if we have five seasons, we can say "less than six will surely capture all of the episodes."  For similar reasons the Django paginator pads the end of a set of paginated objects by adding one, so we can re-use that "last episode plus 1" as the total number of pages and not break anything in the paginator's default behavior, either.

Once the loop is finished we'll have a dict that looks like this...

```shell
(Pdb) season_qs

{2: <website.utils.LocalPaginator object at 0x7fb060fa9f60>, 
1: <website.utils.LocalPaginator object at 0x7fb060fd96d8>}
```

... which represents our two seasons of shows.  With all of our deleted properties and functions replaced with sensible variables taken from the loop, we have created paginator objects that can be templated just like a stock paginator object. Next, we need to move our logic to how the individual paginators are selected to render a page, since we have more than one of them rather than the single paginator object instance present in most configurations. 

# Step by step: dynamic selection and assignment of multiple paginators

Now we need to look for a request coming in that has a page number query paramenter, like so...

```python
        pagenum = request.GET.get('p', 1)

            if pagenum == 'latest':
                try:
                    paged_series_children = season_qs.get(loop_end).page(1)
                except:
                    paged_series_children = season_qs[len(season_qs)].page(1)
            else:
                try:
                    paged_series_children = season_qs.get(int(pagenum)).page(1)
                except:
                    paged_series_children = season_qs[len(season_qs)].page(1)

            context['index_paginated'] = paged_series_children
            context['index_children'] = all_children
```

... If there is a request with `?p=` as a query parameter, get its first value. Next, lets define a query word that will match a redirect defined elsewhere that always points to the `latest` episode's season number.  Presumably most people will want their most recent season prominently displayed at the front of the list, and as you noticed in the loop we started with season 1, not with the last season.

If you're closely reading this you might think: *"but why not just make the default URL for the episode index match the one I want them to land on, rather than setting up a redirect URL?"*  See point above about living in a world of regret, confusion, and doubt.  I can justify landing a user on the *last* page if they give me an invalid input, since I know that there will be a "previous" button on that page that goes to a valid alternative, and thus the trouble they could wander into is minimal. But if you default them to an arbitrary page for an unknown number of use cases *without* rewriting the URL, you will have (possibly) made the first (1) instance of the podcast index into the last (2 for now, but ever-changing) instance of the podcast season index.  The numbers no longer match, and you have created a pagination scheme that will be infinitely more complex to keep straight in your head and your code. Numbers... pages... paginator objects... query parameters... make them match, trust me.

Moving along, `else` if we land on a specific page number, we can grab the matching `season_qs` (paginator object) number that matches it, and since we are going to put all of a season's episodes on a single page, the `page` for the paginator to start with is always `(1)`. If someone asks for a page range out of bounds or some other error happens, we can put this all in `try` and `except` logic with a fallback to a page that we know exists and lands them where we want them to land by default.  A setting here of the `season_qs` paginator object associated with the *last* season (in other words, the length (`len`) of all paginator objects in the dict) will default any errors to dropping people off on the page for the most recent podcast season.

Then all you have to do is add the result to the context and send it along with `return context` at the very end of the function.

Lastly, it should be noted that all of this does not *complely* defeat the Django core paginator.  It's likely that you have shifted around some of the nesting of variables versus the default Django core paginator's output. For example what may have previously been `items.index_paginated.has_next` is now an extra layer removed, and you'll probably have to refer to it in the template as `items.index_paginated.paginator.has_next` to find out whether or not the current page has another season's episode listing page after it.  Remember that we put the data in a dict, so there's an extra layer of abstraction from the last variable in the chain that wasn't there before.  This is a simple thing to handle in the template, though...

```python
{% raw %}templates/pages/content_index_page.html

{% if self.rss_itunes_type == 'serial' %}
{% include "website/includes/pagination_serial_podcast.html" with items=index_paginated %}{% else %}
{% include "website/includes/pagination_standard.html" with items=index_paginated %}{% endif %}
{% endblock %}
{% endraw %}
```

The above will conditionally use a "standard" pagination template if the `rss_itunes_type` custom field on the episode index we're working on is set to "episodic" rather than "serial".  If it matches and renders the `pagination_serial_podcast.html` template, you can put the slightly offset variable nesting variance in that template by itself.

# What else am I forgetting?

Last but not least, if we do this we should probably change the sitemap generator to return the season numbered links to the episode page so that Google knows about all of this.  Thankfully if you're using Wagtail this is pretty easy, as your page classes have a `get_sitemap_urls` function that you can simply override like so:

```python

def get_sitemap_urls(self, request=None):
    if self.rss_itunes_type == 'serial':
        all_children = self.get_index_children().live().public().order_by('first_published_at')
    else:
        pass
    if all_children.first().season_number and self.rss_itunes_type == 'serial':
        oldest_season = all_children.first().season_number
        loop_end = all_children.last().season_number
        season_qs = dict()
        season_list = []
        for i in range(oldest_season, loop_end + 1):
            season_qs[i] = all_children.filter(season_number=i).order_by('-latest_revision_created_at').first()
            season_qs[i].location = self.get_full_url(request) + '?p=' + str(season_qs[i].season_number)
            season_qs[i].lastmod = season_qs[i].latest_revision_created_at
            season_list.append({
                'location': season_qs[i].location,
                'lastmod': season_qs[i].lastmod,
            })

        return season_list
    else:
        # this is the stock sitemap return for episode-type indexes
        return [
            {
                'location': self.get_full_url(request),
                # fall back on latest_revision_created_at if last_published_at is null
                # (for backwards compatibility from before last_published_at was added)
                'lastmod': (self.last_published_at or self.latest_revision_created_at),
            }
        ]
```

Firstly, conditionally get the same queryset we worked with for the paginator based on whether or not this index page is set for "serial" episodes.  If it isn't there's no reason to define an SQL query, we can skip to the `else` return statement at the bottom which doesn't need any SQL queries.

Second, adapt the same paginator queryset from the main part of this article to make URLs and modifcation dates in a range loop instead of making paginators in a range loop.  The logic is exactly the same, we just need to specify URLs and modification dates in each pass of the loop instead of pagination information.

Lastly, append each pass of the loop to a list, as the Django sitemap generator wants a list of dicts for its input.  

The resulting XML output looks good to me:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9" xmlns:xhtml="http://www.w3.org/1999/xhtml">
<url>
    <loc>http://127.0.0.1:8000/</loc>
    <lastmod>2022-01-31</lastmod>
</url>
<url>
    <loc>http://127.0.0.1:8000/episodes/?p=1</loc>
    <lastmod>2022-02-04</lastmod>
</url>
<url>
    <loc>http://127.0.0.1:8000/episodes/?p=2</loc>
    <lastmod>2022-02-04</lastmod>
</url>
<url>
    <loc>http://127.0.0.1:8000/episodes/test-episode-1/</loc>
    <lastmod>2022-02-02</lastmod></url>
...
</urlset>
```

Now Google will leave your root `/episodes/` index alone and instead go to that page by the season numbers, as you would want.  If you have breadcrumbs in your templates or schema metadata you'll need similar logic in those to accomplish the same thing, of course.

And lastly, as mentioned a ways back up in this article you probably want to override the base `/episodes/` URL to always return a particular season, we can do that by overriding the `serve` method on the (Wagtail, in this case) page...

```python
def serve(self, request, *args, **kwargs):
    request.is_preview = getattr(request, 'is_preview', False)

    if not request.GET and self.rss_itunes_type == 'serial':
        path = request.path_info.strip('/')
        slug = self.slug
        if path == slug:
            latest_episode = self.get_index_children().first()
            latest_season = str(latest_episode.season_number)
            return redirect(str(request.get_raw_uri()) + '?p=' + latest_season, permanent=False)

    return super().serve(request, *args, **kwargs)
```

... Stepping through this, "if there is not a parameter in the URL, and if the 'serial' flag is set on the index page, and if the path stripped of slashes equals the slug, return me to the latest season index page, otherwise use the old serve method via the superclass."  This logic may need to be custom tailored a bit to your specifications. The main point here is to bypass sub-paths of the main URL for the index page.  I am using some URLs underneath `/episodes/` for my RSS feeds so I can't match everything under the sun, I need to only match a URL that *ends with* `/episodes/` and leave my RSS links alone.  If all of these conditional checks pass you need to return the superclass of the serve method, so that all other modifications you've made in various other places to the Django / Wagtail serving mechanisms get executed as they would otherwise.

And there you have it!  Django's paginator is still rather simple and clunky, but if you can orchestrate a half dozen of them working in unison, you can accomplish a little more than one of those paginators can do out of the box.

Enjoy!

<div style="text-align-center;">
<video src="https://user-images.githubusercontent.com/98802273/157801187-9b36aa8f-6cf3-4278-8123-3b37f347d660.mp4" controls="controls" style="max-width: 100% !important;height: auto !important;">
</video>

<br/>

