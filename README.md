Overview
====
Goals
----
Create a extensible, non intrusive, fast system for generation navigation groupings.

Reason
----
While there are several systems already out there for crating navigation menus, tabs, etc, they all (That I have found) fail at one or more of my goals.

I want to create a system that any django app can use so that it's default navigation appears or disappears in an existing site based only on having the application in INSTALLED_APPS or not.

Implementation
----
To meet the goal of non intrusive, and auto adding/removing the navigation groupings I looked to djangos own admin autodiscover feature and implemented a similar system _C/P the admin code and make changes where required_

So to enable django_nav you just make add it to your sites urls.py

    from django.conf.urls.defaults import *
    from django.contrib import admin
    import django_nav

    admin.autodiscover()
    django_nav.autodiscover()


    urlpatterns = patterns('',
        ...
    )

_"Ok thats all well and good but what is it discovering?"_ Well it is looking though all of your sites INSTALLED_APPS to see if they have a nav.py file, if they do it processes it.

Here is a simple nav.py file

    from django_nav import nav_groups, Nav, NavOption

    class SubOption(NavOption):
        """
        This is an Option of an Option, this can go on for the max
        recursion depth of python (Further then you want to try and go)
        """
        name = u'Sub Option'
        view = 'example-suboption'

    class TestOption(NavOption):
        """
            This is a Navigation Option, which can be used to build drop down menus
        """
        name = u'Test Option'
        view = 'example-option'
        options = [SubOption]

    class TestNav(Nav):
        """
            This is a primary Navigation link, Most apps should only define one of these
            If the application truly wants to have Navigation links for all of their landing pages
            They can use the NavOption and have the main Nav with their Home state
        """
        name = u'First Test Nav'
        view = 'home'
        nav_group = 'main'
        options = [TestOption]

    nav_groups.register(TestNav)


_"Great, I see some Navigation classes there, but what are the configuration options and whats with this `nav_group` thing?"_

Well the Nav and NavOption share all the same configuration options.
  * __name__ The name displayed in the html
  * __view__ the view this tab should try to reverse to
  * _options__ a list of NavOptions for sub-menus
  * __template__ the template to use (`defaults to django_nav/nav.html` & `django_nav/option.html`)
  * __conditional__ a `dict{'function': None, 'args': [], 'kwargs': {}}` where you set the function, args and kwargs to be used to pass the test. If the conditional method returns False the nav option is not rendered
  * __args__ list of args to use when reversing the view
  * __kwargs__ list of kwargs to use when reversing the view
  * __active_if__ a method you can override to help specify more exactly when your nav item should show as active. By default is checks if the reversed url == current_path. It is passed the revered url and the path

_"What is a nav_group?"_

Well the way I see a site there are three standard navigation groupings. __main__ where all the standard site navigation happens, often seen as tabs, side bar navigation, drop down menus, etc. Then there are two related groups __user_anon__ and __user__. _user_anon_ is a group of navigation options that only show up for an anonymous user, these are often only the login and register links. _user_ is a group of user related navigation, links to their profile, private messages, etc.

_"Ok, well now how do I get the tabs to show up on my site?"_

Here is an example base.html file

    <!DOCTYPE html>{% load nav %}
    <html>
        <head>
            <title>Example for Django Nav</title>
            <link rel="stylesheet" type="text/css" href="{{ MEDIA_URL }}css/stylesheet.css" />
        </head>
        <body>
            <div class="head">
                <img src="{{ MEDIA_URL }}img/logo.jpg" alt="" />
                {% if user.is_authenticated %}
                    {% get_nav "user" as user_nav %}
                {% else %}
                    {% get_nav user_anon as user_nav %}
                {% endif %}
                <ul class="user-nav">
                {% for nav in user_nav %}
                    {{ nav }}
                {% endfor %}
                </ul>
            </div>
            <div class="body">
                <div class="navigation">{% get_nav "main" %}
                    <ul class="nav-tabs">
                    {% for nav in main %}
                        {{ nav }}
                    {% endfor %}
                    </ul>
                </div>
                <div id="main-content">
                {% block content %}{% endblock %}
                </div>
            </div>
        </body>
    </html>

The sections to pay attention to are where __get_nav__ is called. In the example I tried to show three different ways you can call it. The first two differ only in the quoting or not of the group name, and the third leaves off the _as variable_ section, thus using the group name as the navigation list.

Now you might think the three navigation groups are a little restrictive, which for any complex site would be true. Lets take a look at the nav.py's for a live site I maintain.

#### app1/nav.py

    from apps.django_nav import nav_groups, Nav
    from apps.django_nav.conditionals import user_has_perm

    class App1(Nav):
        name = 'App1'
        view = 'app1-new'
        nav_group = 'admin-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

        def active_if(self, url, path):
            return '/app1/' in path

    class New(Nav):
        name = 'New'
        view = 'app1-new'
        nav_group = 'app1-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

        def active_if(self, url, path):
            return '/app1/new/' in path

    class Submitted(Nav):
        name = 'Submitted'
        view = 'app1-submitted'
        nav_group = 'app1-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

        def active_if(self, url, path):
            return '/app1/submitted/' in path

    class Rejected(Nav):
        name = 'Rejected'
        view = 'app1-rejected'
        nav_group = 'app1-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

        def active_if(self, url, path):
            return '/app1/rejected/' in path

    class Details(Nav):
        name = 'Details'
        view = 'app1-view'
        nav_group = 'app1-view'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

    class App1List(Nav):
        name = 'App1 List'
        view = 'app1-App1-list'
        nav_group = 'app1-view'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

    class EditDetails(Nav):
        name = 'Details'
        view = 'app1-edit-view'
        nav_group = 'app1-view'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

    class EditApp1List(Nav):
        name = 'App1 List'
        view = 'app1-edit-app1-list'
        nav_group = 'app1-view'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app1.view_app1'}}

    nav_groups.register(App1)
    nav_groups.register(New)
    nav_groups.register(Submitted)
    nav_groups.register(Rejected)
    nav_groups.register(Details)
    nav_groups.register(App1List)

#### app2/nav.py

    from django_nav import nav_groups, Nav
    from django_nav.conditionals import user_has_perm

    class App2(Nav):
        name = 'App2'
        view = 'app2-admin-active'
        nav_group = 'admin-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class Active(Nav):
        name = 'Active'
        view = 'app2-admin-active'
        nav_group = 'app2-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class Updated(Nav):
        name = 'Updated'
        view = 'app2-admin-updated'
        nav_group = 'app2-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class Closed(Nav):
        name = 'Closed'
        view = 'app2-admin-closed'
        nav_group = 'app2-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class Details(Nav):
        name = 'Details'
        view = 'app2-admin-view'
        nav_group = 'app2-admin-view'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class App2List(Nav):
        name = 'App2 List'
        view = 'app2-admin-app2-list'
        nav_group = 'app2-admin-view'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}


    class Memberapp2(Nav):
        name = 'app2'
        view = 'app2-new'
        nav_group = 'member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class MemberApp2New(Nav):
        name = 'New'
        view = 'app2-new'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class MemberApp2Updated(Nav):
        name = 'Updated'
        view = 'app2-updated'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class MemberApp2Closed(Nav):
        name = 'Closed'
        view = 'app2-closed'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class Memberapp2All(Nav):
        name = 'All'
        view = 'app2-all'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class MemberApp2Hidden(Nav):
        name = 'Hidden'
        view = 'app2-hidden'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class MemberApp2WatchList(Nav):
        name = 'My Watch List'
        view = 'app2-watchlist'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}

    class MemberApp2FirmInterest(Nav):
        name = 'Interest'
        view = 'app2-firminterests'
        nav_group = 'app2-member-main'
        conditional = {'function': user_has_perm, 'args': [],
                       'kwargs': {'perm': 'app2.view_app2'}}


    nav_groups.register(App2)
    nav_groups.register(Active)
    nav_groups.register(Updated)
    nav_groups.register(Closed)
    nav_groups.register(Details)
    nav_groups.register(App1List)

    nav_groups.register(Memberapp2)
    nav_groups.register(Memberapp2New)
    nav_groups.register(Memberapp2Updated)
    nav_groups.register(Memberapp2FirmInterest)
    nav_groups.register(Memberapp2WatchList)
    nav_groups.register(Memberapp2Closed)
    nav_groups.register(Memberapp2Hidden)
    nav_groups.register(Memberapp2All)


Template Files example, the placement of each section is your designers responsibility, just like any other UI component of you site. But simplifying it down to this point lets your designer maintain the nav.py file as long as they have a basic understanding of code structure. They can add/remove/modify any navigation option in this way without needing to modify DB entries.

    <!-- Admin Section -->
    <div id="main_nav">{% get_nav "admin-main" as nav_tabs %}
        <ul class="main-nav">{% for nav in nav_tabs %}{{ nav }}{% endfor %}</ul>
    </div>
    <!-- Member Section -->
    <div id="main_nav">{% get_nav "member-main" as nav_tabs %}
        <ul class="main-nav">{% for nav in nav_tabs %}{{ nav }}{% endfor %}</ul>
    </div>
    <!-- App1 Section -->
    <div id="app1_nav">{% get_nav "app1-main" as app1_tabs %}
        <ul class="nav-tabs">{% for nav in app1_tabs %}{{ nav }}{% endfor %}</ul>
    </div>
    <!-- App1 sub Section -->
    <div id="detail-app1-tabs" class="clear">
    {% get_nav "app1-view" as apv_tabs section app1model.id %}
        <ul class="nav-tabs">{% for nav in apv_tabs %}{{ nav }}{% endfor %}</ul>
    </div>
    <!-- App2 Section -->
    <div id="app2_nav">{% get_nav "app2-main" as app2_tabs %}
        <ul class="nav-tabs">{% for nav in app2_tabs %}{{ nav }}{% endfor %}</ul>
    </div>
    <!-- App2 sub Section -->
    <div id="detail-app2-tabs" class="clear">
    {% get_nav "app2-view" as apv_tabs section app2model.id %}
        <ul class="nav-tabs">{% for nav in apv_tabs %}{{ nav }}{% endfor %}</ul>
    </div>

**NOTE** The subsection for App1 & 2 where I am passing a section and a model.id after my var_name. This is new in version 0.5 and you can pass args and kwargs after the var_name that will be used for the nav items args & kwargs to reverse the view.
The url those are reversing to looks something like

`url(r'^(?P<section>\w+)/view/(?P<model_id>\d+)/$', 'view', name='app1-view'),`


Future Features
----
I would also like to devise a method where nav/options can be populated from items in a DB, I think the it should be possible for applications to write a little middleware code that executes on request, but I have not tried it.

Competitors
----
[django-tabs](http://code.google.com/p/django-tabs/) - Requires you to manually set each tab and it's active state in your templates, so required editing of the template each time you want to add/remove and app

[django-nav-bar](http://code.google.com/p/django-navbar/) - Uses DB to maintain Menu Groupings, so you must take additional action to add/remove an app other then just add it to your INSTALLED_APPS

[Django Menuing System](http://www.rossp.org/blog/2007/nov/27/menus/) - Uses DB to maintain Menu Groupings, so you must take additional action to add/remove an app other then just add it to your INSTALLED_APPS

[Menu/navigation bars in a tag](http://www.djangosnippets.org/snippets/347/) - This is closest to what I am going for, but requires a MENUITEMS setting which you must maintain, but it is atleast in the same file where you are adding/remove an app from.

[greatlemers-django-tools](http://code.google.com/p/greatlemers-django-tools/) - Uses DB to maintain Menu Groupings, so you must take additional action to add/remove an app other then just add it to your INSTALLED_APPS
