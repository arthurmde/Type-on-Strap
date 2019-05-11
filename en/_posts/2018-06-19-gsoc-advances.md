---
layout: post
title: "GSoC Status Update - First Month"
feature-img: "assets/img/pexels/american-football-team.jpg"
thumbnail: "assets/img/thumbnails/american-football-team.jpg"
lang-ref: gsoc-2018-2
lang: en
categories:
  - Hacking
  - FLOSS
tags:
  - gsoc
  - debian
  - floss
  - distro-tracker
excerpt_separator: <!--more-->
---

In the past month I have been working on my [GSoC project]({% post_url 2018-05-09-google-summer-of-code-2018 %})
in Debian's [Distro Tracker](http://tracker.debian.org).
This project aims at designing and implementing new features in Distro Tracker 
to better support Debian teams to track the health of their packages and to
prioritize their work efforts.
In this post, I will describe the current status of my contributions, highlight
the main challenges, and point the next steps.

<!--more-->

## Work Management and Communication

I communicate with Lucas Kanashiro (my mentor) constantly via IRC and personally
at least once a week as we live in the same city. We have a weekly meeting
with Raphael Hertzog at #debian-qa IRC channel to report advances, collect
feedback, solve technical doubts, and planning the next steps.

I created a new [repository in Salsa](https://salsa.debian.org/arthurmde-guest/gsoc-2018)
to save the log of our IRC meetings and to track my tasks through the repository's
issue tracker.Besides that, once a month I'll post a new status update in my 
blog, such as this one, with more details regarding my contributions.

## Advances

When GSoC officially started, Distro Tracker already had some team-related
features. Briefly, a team is an entity composed by one or more users that are
interested in the same set of packages. Teams are created manually by users and
anyone may join public teams. The team page aggregates some basic
information about the team and the list of packages of interest.

Distro Tracker offers a page to enable users to [browser public teams](https://tracker.debian.org/teams/)
which shows a paginated, sorted list of names. It used to be hard to find
a team based on this list since Distro Tracker has more 110 teams distributed
over 6 pages. In this sense, [I created a new search field with auto-complete](https://salsa.debian.org/qa/distro-tracker/merge_requests/31)
on the top of teams page to enable users to find a team's page faster, as show
in the following figure:

![Search Field for Teams Page]({{ "/assets/img/screenshots/distro-tracker-team-search.png" }})

Also, I have been working on improving the current teams infrastructure
to enable Debian's teams to better track the health of their packages.
Initially, we decided to use the current data available in Distro Tracker to
create the first version of a new team's page based on PET.

Presenting team's packages data in a table on the team's page would be a
relatively trivial task. However, Distro Tracker architecture aims to provide
a generic core which can be extended through specific distro applications, such
as [Kali Linux](https://pkg.kali.org/).
The core source code provides generic infrastructure to
import data related to deb packages and also to present them in HTML pages.
Therefore, we had to consider this Distro Tracker requirement to properly
provide a extensible infrastructure to show packages data through tables in
so that it would be easy to add new table fields and to change the default
behavior of existing columns provided by the core source code.

So, based on the previously existing panels feature and on Hertzog's suggestions, 
I designed and developed a framework to create customizable package tables for
teams. This framework is composed of two main classes:

* **BaseTableField** - A base class representing fields to be displayed on
package tables. Among other things, it must define the column name and a
template to render the cell content for a package.
* **BasePackageTable** - A base class representing package tables which are
displayed on a team page. It may have several BaseTableFields to display
package's information. Different tables may show a different list of packages
based on its scope.

We have been discussing my implementation in an [open Merge Request](https://salsa.debian.org/qa/distro-tracker/merge_requests/31),
although we are very close to the version that should be incorporated.
The following figures show the comparison between the earlier PET's table
and our current implementation.

PET Packages Table         |  Distro Tracker Packages Table
:-------------------------:|:-------------------------:
[![PET Packages Table]({{ "/assets/img/screenshots/PET-data.png" }})]({{"/assets/img/screenshots/PET-data.png" }}) | [![Current Teams Page]({{ "/assets/img/screenshots/distro-tracker-team-page.png" }})]({{ "/assets/img/screenshots/distro-tracker-team-page.png" }})


Currently, the team's page only have one table, which displays all packages
related to that team. We are already presenting a very similar set of data to
PET's table. More specifically, the following columns are shown:
* **Package** - displays the package name on the cell. It is implemented by the
core's *GeneralInformationTableField* class
* **VCS** - by default, it displays the type of package's repository (i.e. GIT,
SVN) or **Unknown**. It is implemented by the core's *VcsTableField* class.
However, Debian app extend this behavior by adding the 
changelog version on the latest repository tag and displaying issues identified
by Debian's [VCS Watch](https://qa.debian.org/cgi-bin/vcswatch).
* **Archive** - displays the package version on distro archive. It is
implemented by the core's *ArchiveTableField* class.
* **Bugs** - displays the total number of bugs of a package. It is
implemented by the core's *BugsTableField* class. Ideally, each third-party apps
should extend this field table to both add links for their bug tracker system.
* **Upstream** - displays the upstream latest version available. This is a
specific table field implemented by Debian app since this data is imported
through Debian-specific tasks. In this sense, it is not available for other
distros.


As the table's cells are small to present detailed information,
we have added [Popper.js](https://getbootstrap.com/docs/4.1/components/popovers/),
a javascript library to display popovers. In this sense, some columns show
a popover with more details regarding its content which is displayed on mouse
hover. The following figure shows the popover to the *Package* column:

![Package's Popover]({{ "/assets/img/screenshots/distro-tracker-package-popover.png" }}){: .center }

In additional to designing the table framework, the main challenge
were to avoid the [N+1 problem](https://secure.phabricator.com/book/phabcontrib/article/n_plus_one/)
which introduces performance issues since for a set of **N**
packages displayed in a table, each field element must perform **1** or more
lookup for additional data for a given package. To solve this problem,
each subclass of **BaseTableField** must define a set of
[Django's Prefetch objects](https://docs.djangoproject.com/en/2.0/ref/models/querysets/#django.db.models.Prefetch)
to enable **BasePackageTable** objects to load all required data in batch in advance
through [prefetch_related](https://docs.djangoproject.com/en/2.0/ref/models/querysets/#prefetch-related),
as listed bellow.

```python
class BasePackageTable(metaclass=PluginRegistry):
    @property
    def packages_with_prefetch_related(self):
        """
        Returns the list of packages with prefetched relationships defined by
        table fields
        """
        package_query_set = self.packages
        for field in self.table_fields:
            for l in field.prefetch_related_lookups:
                package_query_set = package_query_set.prefetch_related(l)

        additional_data, implemented = vendor.call(
            'additional_prefetch_related_lookups'
        )
        if implemented and additional_data:
            for l in additional_data:
                package_query_set = package_query_set.prefetch_related(l)
        return package_query_set

    @property
    def packages(self):
        """
        Returns the list of packages shown in the table. One may define this
        based on the scope
        """
        return PackageName.objects.all().order_by('name')


class ArchiveTableField(BaseTableField):
    prefetch_related_lookups = [
        Prefetch(
            'data',
            queryset=PackageData.objects.filter(key='general'),
            to_attr='general_archive_data'
        ),
        Prefetch(
            'data',
            queryset=PackageData.objects.filter(key='versions'),
            to_attr='versions'
        )
    ]

    @cached_property
    def context(self):
        try:
            info = self.package.general_archive_data[0]
        except IndexError:
            # There is no general info for the package
            return

        general = info.value

        try:
            info = self.package.versions[0].value
            general['default_pool_url'] = info['default_pool_url']
        except IndexError:
            # There is no versions info for the package
            general['default_pool_url'] = '#'

        return general
```


Finally, it is worth noticing that we also improved the team's management
page by moving all team management features to a single page and improving
its visual structure:

![Teams Management]({{ "/assets/img/screenshots/distro-tracker-team-management-page.png" }})


## Next Steps

Now, we are moving towards adding other tables with different scopes, such
as the tables presented by PET:

![PET tables]({{ "/assets/img/screenshots/PET-tables.png" }})

To this end, we will introduce the *Tag* model class to categorize the packages
based on their characteristics. Thus, we will create an additional task
responsible for tagging packages based on their available data.
The relationship between packages and tags should be **ManyToMany**. In the end,
we want to perform a simple query to define the scope of a new table, such
as the following example to query all packages with Release Critical (RC) bugs:

```python
class RCPackageTable(BasePackageTable):
    def packages(self):
      tag = Tag.objects.filter(name='rc-bugs')
      return tag.packages.all()
```

We probably will need to work on Debian's VCSWatch to enable it to receive
update through Salsa's webhook, especially for real-time monitoring of
repositories. 

-------------------------------------------------

<span>Let's get moving on! \m/</span>
